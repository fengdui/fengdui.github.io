---
title: "InfluxDB Prometheus"
date: "2022-01-09"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
公司监控系统 使用 InfluxDB 存储监控数据

Measurement（比如 cpu_usage, temperature） 这是分组 类似关系型数据库的表

Point 一条数据记录，由 Measurement, Tag Set, Field Set, Timestamp 构成

Tags（比如 host=server01, region=us-east）—— 作为带索引的过滤维度  类似WHERE 过滤条件的列，InfluxDB 会自动为其建立索引

Fields（比如 value=0.85, temperature=23.5）—— 作为实际指标值 不建索引。

Timestamp —— 每条数据必备的时间戳

写入 InfluxDB 时，用 Influx Line Protocol：cpu,host=web01 user=0.65 1672531200000000000

Prometheus 查询协议并是一套完整的交互体系，其核心可以概括为：以 PromQL (Prometheus Query Language) 为查询语言，以标准 HTTP API 为交互方式

你的查询请求中，核心就是一段用 PromQL 编写的表达式。比如，sum by (method) (rate(api_http_requests_total[5m]))  

Prometheus 引擎解析 PromQL 表达式，从底层存储中检索出匹配的时间序列数据，并进行计算

VictoriaMetrics 也完全支持 Prometheus 的查询协议