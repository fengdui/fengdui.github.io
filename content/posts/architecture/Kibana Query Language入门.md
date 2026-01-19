---
title: "Kibana Query Language入门"
date: "2020-05-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

公司的统一日志系统采用ELK（Elasticsearch、Logstash、Kibana）架构，实现了日志的采集、存储、查询和可视化。  
查询日志使用Kibana Query Language（KQL）进行查询。KQL是一种基于Lucene查询语法的查询语言，用于在Kibana中搜索和过滤日志数据。  
裸词查询	error 不带双引号	模糊搜索：在配置了全文索引的默认字段中，搜索包含 “error” 这个词的文档。  
短语查询	"connection timeout"	精确短语查询：搜索完全匹配整个短语 “connection timeout” 的文档，词序和完整性都必须一致。  
字段查询：通过 字段名: 值 的形式限定搜索范围，更高效。  
示例：status: error （精确匹配 status 字段为 “error” 的日志）  
示例：message: "timeout" （在 message 字段中搜索包含 “timeout” 的日志）  
逻辑运算符：使用 and, or, not 连接条件，直观易懂。  
示例：status: error and host: web-server-01  
通配符与范围查询：  
通配符：host: web-server-* （匹配所有以 web-server- 开头的 host）  
范围：response_time > 500  
