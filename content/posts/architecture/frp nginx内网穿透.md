---
title: "frp nginx内网穿透"
date: "2025-11-30"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
frps（服务端）必须部署在拥有公网IP的服务器上（比如阿里云、腾讯云等云服务器）域名通过DNS解析指向了frps所在的公网服务器IP  
frps：部署在具有公网 IP 的服务器上，作为服务端，负责监听公网端口，接收外部请求，并与 frpc 通信。 frpc：部署在内网需要暴露的服务器上，作为客户端，主动连接到 frps，并维持一条长连接（控制连接）。同时，根据配置，frpc 会建立多条数据隧道（工作连接），用于转发具体的流量  
frpc 启动后，主动向公网 frps 的 7000 端口发起连接，并保持长连接。frps 记录该客户端的身份，并等待后续指令  
frpc 告知 frps 自己需要代理的服务类型（HTTP）和域名（www.example.com）。frps 在公网服务器上监听 8080 端口（根据 vhost_http_port 配置），并准备将发往该域名的 HTTP 请求转发给该 frpc  
用户在公网访问 http://www.example.com:8080，DNS 解析指向公网服务器 IP，请求到达 frps 监听的 8080 端口  
frps 根据 HTTP 请求的 Host 头（www.example.com）匹配到对应的代理规则，找到与该域名关联的 frpc 连接。然后 frps 通过之前建立的隧道，将 HTTP 请求数据封装后发送给 frpc  
frpc 收到数据后，解封装，并根据本地配置（local_port=80），将请求转发给本机的 Nginx（通常是127.0.0.1:80）。这一步可以是简单的 TCP 转发，也可以保留 HTTP 协议头  
Nginx 接收请求，按照正常的 Web 服务逻辑处理（如返回静态文件、执行 PHP、反向代理到其他内网服务等），生成 HTTP 响应返回给 frpc  
frpc 将响应数据通过隧道发送给 frps，frps 再将其发送给公网上的客户端（用户浏览器）  