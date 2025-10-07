---
title: "keepalived使用notify脚本来实现IP漂移"
date: "2025-01-16"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

Keepalived不设置virtual_ipaddress也可以实现主备切换‌。在Keepalived的配置中，可以通过使用notify脚本来实现IP漂移，而不需要在配置文件中显式设置virtual_ipaddress。这种方式主要通过Keepalived的notify脚本机制来实现主备切换，而不是通过虚拟IP地址的绑定和漂移。

实现方式
‌配置Keepalived‌：在Keepalived的配置文件中，可以定义一个notify脚本，该脚本会在主备状态切换时被调用。通过这个脚本，可以实现主备切换时的特定操作，比如重启服务、切换日志记录等。

‌编写Notify脚本‌：可以编写一个Notify脚本，该脚本会在Keepalived检测到主节点故障时执行，从而实现服务的平滑切换。这个脚本可以包含重启服务、更新配置文件等操作。

‌心跳检测‌：Keepalived会定期发送心跳消息，如果检测到主节点故障，则会调用Notify脚本进行切换。