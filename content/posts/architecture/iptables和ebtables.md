---
title: "iptables和ebtables"
date: "2025-01-14"
tags: ["架构", "Linux", "网络", "防火墙"]
ShowToc: true
TocOpen: true
---

# iptables

入站数据PREROUTING-->INPUT；
出站数据OUTPUT-->POSTROUTING；
转发数据PREROUTING-->FORWARD-->POSTROUTING。

```bash
iptables -A INPUT -p tcp --dport 8099 -j ACCEPT
```

## Linux路由查找流程

在Linux中，路由查找遵循以下顺序：
- local表：包含本地接口的路由，优先级最高。
- main表：默认路由表，包含手动添加的静态路由和通过DHCP、RADVD等获取的动态路由。
- default表：如果在前面两个表中找不到路由，Linux会查找default表。

![img_20.png](/pic/img_20.png)
![img_21.png](/pic/img_21.png)
![img_22.png](/pic/img_22.png)

## iptables主要链

- INPUT链：处理输入数据包
- OUTPUT链：处理输出数据包
- FORWARD链：处理转发数据包
- PREROUTING链：用于目标地址转换（DNAT）
- POSTROUTING链：用于源地址转换（SNAT）

## iptables常用参数

- -t 表名
- -N 创建用户自定义的链
- -A 添加规则到链的结尾，最后一条
- -I 插入规则到链的开头，第一条。 越靠前的规则优先级越高。
- -p 匹配协议 如tcp,udp,icmp
- -j 动作 接受或者拒绝
- -m set --match-set --match-set setname flag[,flag]... 其中flags是src和/或dst规范的逗号分隔列表，最多只能有六个
- -m mark --mark

## iptables常用命令示例

```bash
iptables -nL INPUT
$iptables_cmd -t mangle -N $ethMark
$iptables_cmd -t mangle -I OUTPUT -p tcp -j $ethMark  # 添加一条规则 跳转到$ethMark
$iptables_cmd -t mangle -A $ethMark -m mark --mark $mark1 -j ACCEPT
$iptables_cmd -t mangle -I OUTPUT -m set --match-set ${ethRule1} src,src,dst -j MARK --set-mark ${mark2}
# 这里src,src,dst 3个是因为规则集里面也是3个
$iptables_cmd -t nat -A PREROUTING -d $dbIP -p tcp --dport $dbPort -j REDIRECT --to-ports $proxyPort
```

# ipset

```bash
ipset add dstport $proxyPort
# 建立规则集1
ipset create ${ethRule1} hash:ip,port,net family $inet
```

# ip route

via 网关

dev 网卡

metric 路由的优先级

```bash
route -n
route add default gw [网关IP地址]  # 添加默认路由
ip rule show
ip route add default via ${GATEWAY} dev ${ETH} metric 0  
ip rul add fwmark $mark1 table $table1
ip route list table 33849
exec_errlog sudo ip -6 route add ${NETIP_MASK} dev ${ETH} metric $rnd
exec_errlog sudo ip -6 route add default via ${GATEWAY} dev ${ETH} metric $rnd
```

# ebtables

![img_23.png](/pic/img_22.png)
![img_24.png](/pic/img_24.png)
![img_25.png](/pic/img_25.png)
