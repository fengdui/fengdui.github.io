---
title: "借助FlexLexer完成sql标准化和常量提取"
date: "2025-03-03"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

借助FlexLexer完成sql标准化和常量提取  
标准化前  
select * from aj_gs_jbxx where /*order by id desc*/ 1=1 and ajbh='10';  
标准化后  
select * from aj_gs_jbxx where /*1*/ 2=3 and ajbh=4;  


# SQL标准化技术原理

## 核心技术架构

### 1. 词法分析引擎（Flex）

**技术原理**：使用有限状态自动机（FSA）进行词法分析

```cpp
// 正则表达式定义
REX_INTEGER     [-+]?{REX_DIGIT}+
REX_DECIMAL     [-+]?(({REX_DIGIT}*\.{REX_DIGIT}+)|({REX_DIGIT}+\.{REX_DIGIT}*))
REX_ORCLBV      (":"[A-Za-z0-9_]+)          // Oracle: :var
REX_PGBV1       ("${"({REX_VARIABLE}|{REX_NUMBER})"}")  // PG: ${var}
REX_INPARAM     (in|IN|values){REX_INPARAM_SPACE}\(({REX_INPARAM}[,]{REX_INPARAM_SPACE})*{REX_INPARAM}\)
```

**状态机转换**：
```
INPUT → TOKENIZE → CLASSIFY → NORMALIZE → OUTPUT
```

### 2. Token分类系统

**数据结构**：
```cpp
enum DataType {
    NUMBER = 1,      // 数字常量
    DSTRING,         // "字符串"
    QSTRING,         // '字符串'
    BINDORCL,        // :var (Oracle)
    BINDPG,          // $1, ${var} (PostgreSQL)
    BINDSQLSERVER,   // @var (SQL Server)
    COMMENT,         // -- /* */ #
    TABLE,           // 表名/字段名
    INPARAM,         // IN(1,2,3)
    OPERATOR         // =, >, <, !=
};
```

### 3. 占位符生成算法

**替换策略**：
```cpp
// 不同场景的占位符格式
#define PLACEHOLDER(index)       (std::to_string(++index))        // 1,2,3
#define PLACEHOLDER_WEB(index)   (":"+std::to_string(++index))    // :1,:2,:3  
#define PLACEHOLDER_AUDIT(index) ("?")                            // ?,?,?
#define PLACEHOLDER_API(index)   ("$"+std::to_string(++index))    // $1,$2,$3
```

**核心替换逻辑**：
```cpp
bool assembleRetInfo(SQLNormalizeRet& ret, const std::string& value, 
                    const std::string& bvname, const ValueType valueType, 
                    normal_context_sptr& ctx) {
    uint32_t bindSize = ret.bindValue_.size();
    std::string placeholder;
    
    switch (ctx->m_normType) {
        case PLACEHOLDER_WEB:   placeholder = PLACEHOLDER_WEB(bindSize); break;
        case PLACEHOLDER:       placeholder = PLACEHOLDER(bindSize); break;
        case PLACEHOLDER_AUDIT: placeholder = PLACEHOLDER_AUDIT(bindSize); break;
    }
    
    // 保存原始值到元数据
    DetailRet dret;
    dret.value_ = value;
    dret.valueType_ = valueType;
    dret.offset_ = ret.normalizeSQL_.size();
    ret.bindValue_.emplace_back(dret);
    
    // 替换为占位符
    ret.normalizeSQL_ += placeholder;
    return true;
}
```

### 4. 多数据库语法适配

**数据库特征映射**：
```cpp
// 绑定变量语法识别
std::map<DBType, DataType> dbbindvar_map = {
    {ORACLE,     BINDORCL},      // :var
    {MYSQL,      QUESTION},      // ?
    {POSTGRESQL, BINDPG},        // $1, ${var}
    {SQLSERVER,  BINDSQLSERVER}, // @var
    {DB2,        BINDDB2}        // #var#
};

// 数据库方言处理
std::map<DBType, DBType> orclspecs_map;   // Oracle语法族
std::map<DBType, DBType> mysqlspecs_map;  // MySQL语法族
```

### 5. IN参数优化算法

**长列表压缩**：
```cpp
// 原始: SELECT * FROM users WHERE id IN (1,2,3,4,5,6,7,8,9,10)
// 标准化: SELECT * FROM users WHERE id IN (?)

uint32_t handleINParamLongData(SQLNormalizeRet& ret, size_t& index, 
                              normal_context_sptr& ctx) {
    std::string tempStr = "";
    bool isEnd = false;
    
    // 遍历收集所有IN参数内容
    for (++index; index < ctx->m_sqlList.size(); ++index) {
        switch (ctx->m_sqlList[index].type_) {
            case OPERATOR:
            case BRACKETS:
                tempStr += ctx->m_value + " ";
                break;
            default:
                isEnd = true;
                break;
        }
        if (isEnd) break;
    }
    
    // 整个IN参数替换为单个占位符
    assembleRetInfo(ret, tempStr, "", ValueType(11), ctx);
    return --index;
}
```

### 6. 正则表达式引擎（PCRE2）

**封装架构**：
```cpp
class NormalPcre2 {
private:
    struct Pcre2Pointer {
        pcre2_code* m_pcre2_code_sptr;           // 编译后的正则
        pcre2_match_data* m_pcre2_match_data_sptr; // 匹配数据
    };
    
public:
    Pcre2Pointer_T create(const std::string& regex);     // 预编译正则
    std::vector<std::string> match(const std::string& targetStr, 
                                  const Pcre2Pointer_T& pcre2Sptr); // 匹配
    bool isPcreMatch(const std::string& targetStr, 
                    const Pcre2Pointer_T& pcre2Sptr);    // 快速判断
};
```

**性能优化**：
- **预编译**：正则表达式一次编译，多次使用
- **内存池**：复用匹配数据结构
- **RAII管理**：自动资源释放

### 7. 处理流水线

```
原始SQL → 预处理 → Flex词法分析 → Token分类 → 标准化处理 → 结果组装
   ↓         ↓         ↓           ↓         ↓         ↓
"SELECT    去空格    [SELECT]    KEYWORD   保持原样   SELECT
* FROM     换行符    [*]         OPERATOR  保持原样   * FROM  
users      分号      [FROM]      KEYWORD   保持原样   users
WHERE              [users]      TABLE     保持原样   WHERE
id = 123"          [WHERE]      KEYWORD   保持原样   id = ?
                   [id]         TABLE     保持原样
                   [=]          OPERATOR  保持原样
                   [123]        NUMBER    替换为?
```

### 8. 元数据结构

```cpp
struct DetailRet {
    std::string bindname_;    // 绑定变量名
    std::string value_;       // 原始值
    ValueType valueType_;     // 值类型
    uint32_t offset_;         // 在标准化SQL中的位置
};

struct SQLNormalizeRet {
    std::string normalizeSQL_;           // 标准化后的SQL
    std::vector<DetailRet> bindValue_;   // 被替换的值列表
    SQLType sqlType_;                    // SQL类型(DDL/DML/DCL/TCL)
    std::string sqlCmdName_;             // SQL命令名
    bool iswhite_;                       // 白名单标记
};
```

## 技术特点

### 性能优化
- **零拷贝**：Token处理过程中避免不必要的字符串拷贝
- **预编译正则**：正则表达式编译一次，重复使用
- **流式处理**：逐Token处理，内存占用恒定

### 扩展性设计
- **插件化数据库支持**：通过配置文件添加新数据库
- **可配置占位符格式**：支持不同审计系统需求
- **模块化架构**：词法分析、语法处理、标准化分离

### 容错机制
- **语法容错**：遇到未知Token时保持原样
- **编码兼容**：支持UTF-8多字节字符
- **长SQL截断**：超长SQL自动截断处理

## 实际应用示例

### 标准化前后对比

**原始SQL**：
```sql
SELECT u.name, u.age FROM users u 
WHERE u.id = 123 AND u.status = 'active' 
AND u.created_date > '2023-01-01' 
AND u.department_id IN (1,2,3,4,5)
```

**标准化后**：
```sql
SELECT u.name, u.age FROM users u 
WHERE u.id = ? AND u.status = ? 
AND u.created_date > ? 
AND u.department_id IN (?)
```

**元数据**：
```json
{
  "bindValue": [
    {"value": "123", "valueType": "NUMBER", "offset": 45},
    {"value": "active", "valueType": "QSTRING", "offset": 65},
    {"value": "2023-01-01", "valueType": "QSTRING", "offset": 95},
    {"value": "1,2,3,4,5", "valueType": "INPARAM", "offset": 125}
  ],
  "sqlType": "DML",
  "sqlCmdName": "SELECT"
}
```

## 支持的数据库

| 数据库类型 | 绑定变量语法 | 特殊处理 |
|-----------|-------------|----------|
| Oracle | `:var` | 双引号字符串保持原样 |
| MySQL | `?` | 支持`#`注释 |
| PostgreSQL | `$1`, `${var}` | 支持`$`占位符 |
| SQL Server | `@var` | 支持`N'unicode'` |
| DB2 | `#var#` | 特殊绑定变量语法 |

这套技术架构实现了高性能、高精度的SQL标准化，是企业级数据库审计系统的核心技术基础。