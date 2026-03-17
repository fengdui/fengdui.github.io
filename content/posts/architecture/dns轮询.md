---
title: "dns轮询"
date: "2018-07-11"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
配置多个IP：网站管理员在DNS服务商的后台，为域名（例如 www.example.com）添加多条A记录。每条记录都指向一台提供相同服务的服务器IP（例如 192.168.1.1、192.168.1.2、192.168.1.3）。
轮流返回IP 每个ip对应一个nginx服务器 避免nginx单点问题
