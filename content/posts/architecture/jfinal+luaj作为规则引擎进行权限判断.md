---
title: "jfinal+luaj作为规则引擎进行权限判断"
date: "2025-03-18"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
先对于每个业务模块定义一个jfinal模板 内置一些变量用于动态渲染  
比如a用户对于表a需要放行 b用户对于表b需要脱敏 把业务上可变的参数作为模板引擎里面的占位符  
当用户配置玩规则之后使用使用jfinal模板引擎渲染 得到一个代码片段  
这个片段是lua的脚本片段 是数据片段  
每个模块还有一个主的lua脚本 这个是内置的一些逻辑和方法 这个主lua脚本 通过require的方式加载模板引擎渲染的数据lua脚本  
实现了逻辑与数据的分离，便于维护和扩展。MainFile负责处理业务逻辑，而DataFile则专注于提供数据支持  

原始未渲染的模版一个例子如下
```
#for(identityKey: dataMap.keySet())
    dataMap["#(identityKey)"] = {
        #for(item : dataMap.get(identityKey))
        {
            id = #(item.id),
            name = "#(item.name)",
            catalog = "#(item.catalog)",
            schema = "#(item.schema)",
            table = "#(item.table)",
            ruleType = #(item.ruleType),
            -- 其他字段...
        },
        #end
    }
#end
```
LuaJ用于加载和执行Lua脚本 加载的是Main lua
```
@Override
public boolean loginAction(String uuid, Integer dbId, List<String> identityIds) {
    LuaValue luaFunction = LuaJUtils.getFunction(globals, "check_login");
    LuaValue dbIdLua = LuaValue.valueOf(dbId);
    LuaTable identityIdsLua = new LuaTable();
    for (int i = 0; i < identityIds.size(); i++) {
        identityIdsLua.set(i + 1, identityIds.get(i));
    }
    LuaValue luaResponse = luaFunction.call(dbIdLua, identityIdsLua);
    return luaResponse.toboolean();
}

```


main lua里面有一个loadAllData方法 去加载data lua
当用户配置变更时 会会触发reload

```
function loadAllData(dbIdToFileMap)
    for id, fileName in pairs(dbIdToFileMap) do
        local data = require(fileName)
        -- 输出 data 文件中的变量
        sqlRuleMap[id] = data.sqlRuleMap
        sqlIdMap[id] = data.sqlIdMap
    end
    return true
end
```