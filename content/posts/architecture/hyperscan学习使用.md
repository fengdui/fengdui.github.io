---
title: "hyperscan学习使用"
date: "2025-07-21"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
Hyperscan 是一个高性能、多模式正则表达式匹配库，它的核心设计目标就是在大量的文本数据中，以极快的速度同时匹配成千上万条正则表达式规则 
在构建一个需要实时分析网络流量、同时匹配上万条攻击规则的防火墙或监控系统时，Hyperscan 是专为此而生的工业级解决方案，而 Java 正则表达式在性能和架构上完全无法胜任  
原理不一样 速度不一样 这里使用pcre性能跟不上了

```
import com.gliwka.hyperscan.wrapper.Database;
import com.gliwka.hyperscan.wrapper.Expression;
import com.gliwka.hyperscan.wrapper.Scanner;
import com.gliwka.hyperscan.wrapper.StringMatchEventHandler;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.util.HashSet;
import java.util.LinkedList;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

@Slf4j
public class HyperScanRegularExpressService {

    private Database db = null;

    private Map<Integer, String> REGULARS_MAP= new ConcurrentHashMap(10);


    public boolean regularMath(String regular) {
        Expression expression = new Expression(regular);
        try (Database db = Database.compile(expression)) {
            return true;
        }catch (Throwable  e) {
            log.error("规则：{} hyperscan不支持，默认走pcre, {}", regular, e.getMessage());
            return false;
        }
    }

    public void build()  throws Exception {
        close();
        if(REGULARS_MAP == null || REGULARS_MAP.isEmpty()) {
            log.warn("hyper scan支持的规则列表为空");
            return;
        }
        LinkedList<Expression> expressions = new LinkedList<>();
        for (Map.Entry<Integer, String> entry : REGULARS_MAP.entrySet()) {
            expressions.add(new Expression(entry.getValue(), entry.getKey()));
        }
        db = Database.compile(expressions);
    }

    public Set<Integer> expressionMatch(String expression) throws Exception {
        if(db == null) {
            return null;
        }
        Set<Integer> result = new HashSet<>();
        com.gliwka.hyperscan.wrapper.Scanner scanner = null;
        try {
            scanner = new Scanner();
            scanner.allocScratch(db);
            scanner.scan(db, expression, new StringMatchEventHandler() {
                @Override
                public boolean onMatch(Expression expression, long l, long l1) {
                        result.add(expression.getId());
                        return true;
                    }
            });
        }finally {
            if(scanner != null) {
                scanner.close();
            }
        }
        return result;
    }

    public void close() throws IOException {
        if(db != null) {
            db.close();
        }
    }
}
```