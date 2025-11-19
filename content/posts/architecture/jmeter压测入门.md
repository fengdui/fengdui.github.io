---
title: "jmeter压测入门"
date: "2017-08-27"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

jmeter新建测试计划 配置完成线程数 http请求参数之后保存  
拿到jmx文件  
jmeter -n -t <测试计划文件.jmx> -l <结果文件.jtl> -e -o <报告输出文件夹>  

Waiting for possible Shutdown/StopTestNow/HeapDump/ThreadDump message on port 4445  
summary +    138 in 00:00:21 =    6.5/s Avg:  1152 Min:   134 Max:  2087 Err:     0 (0.00%) Active: 10 Started: 10 Finished: 0  
summary +    198 in 00:00:30 =    6.6/s Avg:  1521 Min:   439 Max:  2453 Err:     0 (0.00%) Active: 10 Started: 10 Finished: 0  
summary =    336 in 00:00:51 =    6.5/s Avg:  1370 Min:   134 Max:  2453 Err:     0 (0.00%)  
summary + 176656 in 00:00:30 = 5917.7/s Avg:     1 Min:     0 Max:  2248 Err: 176526 (99.93%) Active: 10 Started: 10 Finished: 0  
summary = 176992 in 00:01:21 = 2180.0/s Avg:     4 Min:     0 Max:  2453 Err: 176526 (99.74%)  
summary + 405843 in 00:00:30 = 13505.1/s Avg:     0 Min:     0 Max:  2305 Err: 405775 (99.98%) Active: 10 Started: 10 Finished: 0  
summary = 582835 in 00:01:51 = 5239.4/s Avg:     1 Min:     0 Max:  2453 Err: 582301 (99.91%)  
