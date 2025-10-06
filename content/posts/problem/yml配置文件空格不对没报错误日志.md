---
title: "Received fatal alert: handshake_failure"
date: "2024-08-09"
tags: ["问题"]
ShowToc: false
TocOpen: false
---


java -jar -Dlogging.config=./config/log4j22.xml -Dspring.config.location=./config/application2.yml asset-web.jar
-Dspring.config.location=./config/application2.yml 指定配置文件 配置文件有误无法打日志 不指定可以打日志