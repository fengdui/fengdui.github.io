---
title: "使用clickhouse优化大数据量写入和查询"
date: "2025-09-02"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

最近使用clickhouse存储了千万数据 当然这点数据是小case  
但是实在是没有其他可用的存储了 拿ck代替es  
clickhouse这类olad对于大批量写入是很友好的 对于修改是效率很低的 当然es也不支持修改  
一般是使用了各种表引擎  
ReplacingMergeTree 引擎  一条数据插入会有版本号  
-- 使用子查询获取最新版本  
SELECT *  
FROM user_actions  
WHERE (user_id, action_time, version) IN (  
SELECT user_id, action_time, max(version)  
FROM user_actions  
GROUP BY user_id, action_time  
);  
避免使用final  
其他CollapsingMergeTree VersionedCollapsingMergeTree也是类似的  

没有用clickhouse的Mutation机制  
查询的时候借助PREWHERE 提效  