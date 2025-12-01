---
title: "Service Mesh"
date: "2019-08-10"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
```
[服务A] ←→ [Sidecar代理] ←→ [网络] ←→ [Sidecar代理] ←→ [服务B]
    │           │                       │           │
    └───────────┘                       └───────────┘
     本地通信                            本地通信
···

Sidecar 代理
部署模式：每个服务实例旁部署一个代理容器
但是 公司没有使用istio这种服务网格
而是自研了代理 去做协议的转换 因为公司用到了多种协议和多种语言 php go java node dubbo grpc http
有时候php 调用 java 服务 就需要用 dubbo 协议
于是 就需要在代理中去做协议的转换 比如 http 转 dubbo
后端只需要关注业务逻辑 不需要关注协议转换 全都写成dubbo接口


