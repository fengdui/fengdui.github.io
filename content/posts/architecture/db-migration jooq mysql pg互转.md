---
title: "db-migration jooq mysql pg互转"
date: "2025-05-26"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

db-migration结合jooq实现flway同步pgsql库版本  
现在项目里面维护的sql是mysql语法的 需要改写flway执行sql的时候的语法 把mysql的sql转成pgsql的语法  
参考了https://gitee.com/mengweijin/db-migration 这个库去改写flway 拿到sql再去转换  
如果有些库比如国产数据库flway不支持 也可以使用db-migration扩展  

Mysql2PGTranslate
```
import org.jooq.Param;
import org.jooq.Query;
import org.jooq.SQLDialect;
import org.jooq.impl.DSL;

import java.util.Iterator;
import java.util.Map;

public class Mysql2PGTranslate {
    public static String translate(String sql, boolean quoteSymbol, int upperLowerCase) {
        if (sql.length() >= 20 && sql.contains("/*sqltranslateSkip*/")) {
            return sql;
        }

        sql = sql.trim();
        if (sql.charAt(sql.length() - 1) == ';') {
            sql = sql.substring(0, sql.length() - 1);
        }

        boolean bindFlag = false;
        Query query = DSL.using(SQLDialect.MYSQL).parser().parseQuery(sql);
        Iterator<Map.Entry<String, Param<?>>> it = query.getParams().entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<String, Param<?>> entry = it.next();
            Param<?> value = entry.getValue();
            if (value.getValue() == null) {
                bindFlag = true;
                query.bind(entry.getKey(), "_MC_TODO_REPLACE_");
            }
        }

        String result = DSL.using(SQLDialect.POSTGRES).renderInlined(query);
        if (bindFlag) {
            result = result.replace("'_MC_TODO_REPLACE_'", "?");
        }
        return result;
    }

    public static void main(String[] args) {
        String sql = "select\n" +
                "   *\n" +
                "    from soc_monitor_warn\n" +
                "    where state = 1\n" +
                "    or (state = 3 and create_time > `1123`);";
        System.out.println(translate(sql, true, 0));
    }
}
```
PG2MysqlTranslate
```
import org.jooq.Query;
import org.jooq.SQLDialect;
import org.jooq.impl.DSL;

public class PG2MysqlTranslate {
    public static String translate (String sql, boolean quoteSymbol, int upperLowerCase) {
        if (sql.length() >= 20 && sql.contains("/*sqltranslateSkip*/")) {
            return sql;
        }

        Query query = DSL.using(SQLDialect.POSTGRES).parser().parseQuery(sql);
        String result = DSL.using(SQLDialect.MYSQL).renderInlined(query);
        return result;
    }
}
```