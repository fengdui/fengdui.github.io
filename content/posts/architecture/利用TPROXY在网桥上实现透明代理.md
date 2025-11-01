---
title: "利用TPROXY在网桥上实现透明代理"
date: "2025-03-17"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

# 透明代理
意思是客户端根本不需要知道有代理服务器的存在，它改变你的request fields（报文），并会传送真实IP，多用于路由器的NAT转发中。注意，加密的透明代理则是属于匿名代理，意思是不用设置使用代理了，例如Garden 2程序。
# 旁路模式
设置网卡为混杂模式接收所有经过网卡的数据包，包括不是发给本机的包，即不验证MAC地址

网桥结合tproxy实现透明代理  
将网桥绑定网卡  
ebtables过滤数据包并重定向到iptables  
iptables结合tproxy重定向流量到代理服务  
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j MARK --set-mark 1  
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j TPROXY --tproxy-mark 1 --on-port 3128  


# 利用TPROXY在网桥上实现透明代理(Fully Transparent Proxy)功能 (CentOS 7)

## 1. TPROXY是什么

你可能听说过TPROXY，它通常配合负载均衡软件HAPrxoy或者缓存软件Squid使用。

在所有"Proxy"类型的应用中都一个共同的问题，就是后端的目标服务器上看到的连接的Source IP都不再是用户原始的IP，而是前端的"Proxy"服务器的IP。

拿HAProxy举例来说，假设你有3台后端Web服务器提供服务，前端使用HAProxy作为负载均衡设备。所有用户的HTTP访问会先到达HAProxy，HAProxy作为代理，将这些请求按照指定的负载均衡规则分发到后边的3台Web服务器上。这个操作本身没有任何问题，因为HAProxy就应该是这么工作的。但是对于某些对于用户的访问IP有限制的敏感应用，问题来了：后端服务器上的ACL无法限制哪些IP可以访问，因为在它看来，所有连接的SOURCE IP都是HAProxy的IP。

这就是为什么TPROXY产生的原因，最早TPROXY是作为Linux内核的一个patch，从2.6.28以后TPRXOY已经进入官方内核。TPRXOY允许你"模仿"用户的访问IP，就像负载均衡设备不存在一样，当HTTP请求到达后端的Web服务器时，在后端服务器上用netstat查看连接的时候，看到连接的SOURCE IP就是用户的真实IP，而不是haproxy的IP。**TPROXY名字中的T表示的就是transparent(透明)**。

### TPROXY主要功能
1. 重定向一部分经过路由选择的流量到本地路由进程(类似NAT中的REDIRECT)
2. 使用非本地IP作为SOURCE IP初始化连接
3. 无需iptables参与，在非本地IP上起监听

如果想要了解TPROXY的具体工作原理，请参考作者本人写的PPT：
http://people.netfilter.org/hidden/nfws/nfws-2008-tproxy_slides.pdf

## 2. Centos 7系统，iptables透明网桥实现

在2.6.28以后的内核中，TPROXY已经是官方内核的一部分了。CentOS 7 默认使用的是3.10内核，所以不需要额外添加。

在一切开始之前，需要安装两个包：bridge-utils，ebtables

```bash
# yum install bridge-utils ebtables
```

### 2.1 建立网桥

使用bridge工具桥接eth0与eth1网口：

```bash
/sbin/modprobe bridge
/usr/sbin/brctl addbr br0
/sbin/ifup eth0
/sbin/ifup eth1
/usr/sbin/brctl addif br0 eth0
/usr/sbin/brctl addif br0 eth1
/sbin/ip link set br0 up
```

在网桥上设置IP地址：

```bash
# ifconfig br0 192.168.10.90 netmask 255.255.255.0
```

这样就完成了网桥的配置工作。但这时配置没有保存下来，如果要重启有效，需要添加网桥的配置文件。

### 2.2 保存网桥配置

创建或编辑以下配置文件：

**/etc/sysconfig/network-scripts/ifcfg-br0**

```ini
DEVICE=br0
TYPE=Bridge
BOOTPROTO=static
ONBOOT=yes
IPADDR=192.168.10.90
NETMASK=255.255.255.0
```

**/etc/sysconfig/network-scripts/ifcfg-eth0**

```ini
TYPE="Ethernet"
BOOTPROTO="none"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"
BRIDGE="br0"
```

**/etc/sysconfig/network-scripts/ifcfg-eth1**

```ini
TYPE="Ethernet"
BOOTPROTO="none"
NAME="eth1"
DEVICE="eth1"
ONBOOT="yes"
BRIDGE="br0"
```

这样，机器重启以后也会保留已配置好的网桥。

## 3. 启用bridge-nf

由于网桥工作于数据链路层，在iptables没有开启bridge-nf时，数据会直接经过网桥转发，结果就是对FORWARD的设置失效；

CentOS默认不开启bridge-nf，启动方法：编辑文件`vim /etc/sysctl.conf`添加：

```conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
```

然后执行命令：

```bash
# sysctl -p
```

这时候，可能会出现

```bash
# sysctl -p
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-ip6tables: No such file or directory
sysctl: cannot stat /proc/sys/net/bridge/bridge-nf-call-iptables: No such file or directory
```

是由于CentOS 7 默认没有加载br_netfilter模块，我们加载它：

```bash
# modprobe br_netfilter
```

为了让系统启动的时候，就自动加载此模块，我们得加一个模块启动脚本：

**vim /etc/sysconfig/modules/br_netfilter.modules**

```bash
#!/bin/sh
/sbin/modinfo -F filename br_netfilter > /dev/null 2>&1
if [ $? -eq 0 ]; then
/sbin/modprobe br_netfilter
fi
```

文件保存完成，记得加上可执行权限，这样就将br_netfilter加入了开机启动：

```bash
# chmod 755 /etc/sysconfig/modules/br_netfilter.modules
```

## 4. 添加iptables，ebtables规则

在添加规则之前，先确认一下`/etc/sysctl.conf`文件内容是否包含如下项：

```conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1

net.ipv4.ip_forward = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.br0.rp_filter = 0
net.ipv4.conf.eth0.rp_filter = 0
net.ipv4.conf.eth1.rp_filter = 0
```

确认无误后，执行：

```bash
# sysctl -p
```

让配置生效。

### 添加iptables基础规则

```bash
/sbin/iptables -F
/sbin/iptables -F -t mangle
/sbin/iptables -t mangle -N DIVERT
/sbin/iptables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT
/sbin/iptables -t mangle -A DIVERT -j MARK --set-mark 1
/sbin/iptables -t mangle -A DIVERT -j ACCEPT
```

### 添加路由规则

在lo介面上建立代号为100的路由表给iptables比对fwmark：

```bash
/sbin/ip -f inet rule add fwmark 1 lookup 100
/sbin/ip -f inet route add local default dev lo table 100
```

### 以SSH 22端口代理为例的规则配置

```bash
# 首先添加一台对本机访问22端口的通过规则，此例本机连接网卡是eth2
iptables -t mangle -A PREROUTING -i eth2 -p tcp --dport 22 -j ACCEPT

# 将桥接模式的封包移到路由模式，此例eth1是数据流入口（接客户端），eth0是数据流出口（接服务端）
ebtables -t broute -A BROUTING -i eth0 -p ipv4 --ip-proto tcp --ip-sport 22 -j redirect --redirect-target DROP
ebtables -t broute -A BROUTING -i eth1 -p ipv4 --ip-proto tcp --ip-dport 22 -j redirect --redirect-target DROP

# 将22端口的流量重定向到TCP端口9876
iptables -t mangle -A PREROUTING -p tcp --dport 22 -j TPROXY --tproxy-mark 0x1/0x1 --on-port 9876
```

经过如上配置，网桥上的透明代理设置基本完成了。

## 5. 运行代理转发程序

socket有一个IP_TRANSPARENT选项，其含义就是可以使一个服务器程序侦听所有的IP地址，哪怕不是本机的IP地址，这个特性在实现透明代理服务器时十分有用，而其使用也很简单：

```c
int opt =1;
setsockopt(server_socket, SOL_IP, IP_TRANSPARENT, &opt, sizeof(opt));
```

此时，自己编写一个socket代理程序，开启如上参数，监听在9876端口上，就可以完成透明代理的功能了。

当然，我们也可以使用HAProxy这个软件实现透明代理的功能，只要在它的GROUP配置中加入如下一行：

source 0.0.0.0 usrsrc clientip

就可以达到一样的效果。此处就不再赘述，可参考HAProxy的部署文章。


