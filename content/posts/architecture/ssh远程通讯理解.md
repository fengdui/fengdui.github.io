---
title: "ssh远程通讯理解"
date: "2025-11-10"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

最近接触到了公司跳板机要颁发证书给个人 然后用证书ssh到服务器上才能通过认证
接触到了很多业务也是这样做的 这部分属于客户端认证 服务器认证客户端证书
# ffs
Amazon ffs 需要将证书烧录到设备 通过证书才能连接到amazon云上 证明设备的身份
# 远程安装
公司使用sshj和winrm4j远程到用户的服务器上 用户把私钥上传到服务器 服务器使用私钥进行ssh通讯
# 跳板机
个人使用ssh+私钥连接跳板机

用户私钥 → SSH客户端 → 服务器 → 检查 ~/.ssh/authorized_keys → 找到匹配公钥 → 允许登录

用户私钥+证书 → SSH客户端 → 服务器 → 检查证书签名(用CA公钥) → 证书有效 → 允许登录

公钥认证需要上传公钥到服务器 authorized_keys里 参考github
证书方便了 只要是ca签发的都可以 不用一个个上传

问题
登录最后都是以root用户 ssh root@xxx.com 登录服务器 证书的主体里面没有root 那个人信息是不是没用了

因为连接到跳板机上还要进一步ssh到具体的服务器上 需要将配置信息穿透过去 跳板机上没有你~/.ssh目录下面的那些东西嘛
需要使用ssh agent

config配置
```
Host aaa.com
HostName aaa.com
User root
ForwardAgent yes
AddkeysToAgent yes
IdentityFile ~/.ssh/id_ed25519
IdentitiesOnly yes
```
# 1. 在本地确保 Agent 运行并加载密钥
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519  # 或你的私钥文件

# 2. 连接到跳板机并启用 Agent 转发
ssh -A username@jump-host

# 3. 在跳板机内验证 Agent 是否可用
ssh-add -l  # 应该显示你的密钥

# 4. 现在连接到目标机应该不需要密码
ssh username@target-host

https证书校验逻辑 这个是服务端认证 客户端校验服务端证书

* 服务器 证书包含
* 公钥 + 基本信息 + 上一级机构的信息  + 前面三者的哈希在用上级机构的私钥加密
* 其中上一级机构的信息包含

* 客户端校验证书 发现上一级机构的信息 从而找到上一级的证书 拿到上一级证书的公钥 解密当前证书的哈希得到解密后的哈希
* 然后客户端在对当前证书 公钥 + 基本信息 + 上一级机构的信息进行哈希和对比 一样的话说明当前证书是上一级证书机构颁发的
* 然后在继续校验上一级机构 整个链都对
* 然后用服务器的证书的公钥进行加解密传输
