---
title: "sqlserver加入域控"
date: "2024-01-12"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

在“域”模式下，至少有一台服务器负责每一台联入网络的电脑和用户的验证工作，相当于一个单位的门卫一样，称为“域控制器（Domain Controller，简写为DC）”。
安装完后会重启，重启后，在输入账号处，可以看到账号名称发生了变化，变成了 TEST\Administrator ，这是域账号的登录模式，要登录一个域，需要在账号前面加上域名称和斜杠


![img_28.png](/pic/img_28.png)
![img_29.png](/pic/img_29.png)

sudo realm join --verbose ABC.COM -U 'administrator@ABC.COM' --membership-software=adcli  


[root@xxx ~]# sudo realm join --verbose ABC.COM -U 'administrator@ABC.COM' --membership-software=adcli
- Resolving: _ldap._tcp.abc.com
- Performing LDAP DSE lookup on: 192.168.240.208
- Successfully discovered: abc.com
- Required files: /usr/sbin/oddjobd, /usr/libexec/oddjob/mkhomedir, /usr/sbin/sssd, /usr/sbin/adcli
- LANG=C /usr/sbin/adcli join --verbose --domain abc.com --domain-realm ABC.COM --domain-controller 192.168.240.208 --login-type user --login-user administrator@ABC.COM --stdin-password
- Using domain name: abc.com
- Calculated computer account name from fqdn: XXX
- Using domain realm: abc.com
- Sending NetLogon ping to domain controller: 192.168.240.208
- Wrote out krb5.conf snippet to /var/cache/realmd/adcli-krb5-lSA4Mf/krb5.d/adcli-krb5-conf-b9oYkV
- Authenticated as user: administrator@ABC.COM
- Using GSS-SPNEGO for SASL bind

https://learn.microsoft.com/zh-cn/azure-data-studio/enable-kerberos?tabs=ubuntu
https://learn.microsoft.com/zh-CN/troubleshoot/sql/database-engine/connect/using-kerberosmngr-sqlserver
https://learn.microsoft.com/zh-cn/sql/connect/jdbc/using-kerberos-integrated-authentication-to-connect-to-sql-server?view=sql-server-ver16
https://learn.microsoft.com/zh-cn/sql/connect/jdbc/setting-the-connection-properties?view=sql-server-ver16
https://learn.microsoft.com/zh-cn/previous-versions/sql/sql-server-2008-r2/cc280745(v=sql.105)
https://learn.microsoft.com/zh-cn/sql/database-engine/configure-windows/register-a-service-principal-name-for-kerberos-connections?view=sql-server-ver16

- 1、首先获取域名以及域服务所处的ip，还有sqlserver服务器的主机名，可以通过setspn -L %COMPUTERNAME%命令获取，还需要windwos账号的密码
- 2、/capaa/server/ddd-web/config目录下新建krb5.ini文件，将default_realm的值设置为域名，修改realms配置，域名指向的kdc服务ip为上述拿到的域服务所处ip
- 3、修改/etc/hosts文件，将sqlserver服务器ip 与sqlserver服务器主机名做映射
- 4、页面ip填写sqlserver主机名，密码为windows账户密码，用户名为windows账号@域名
- 5、测试连接是否成功
https://learn.microsoft.com/zh-cn/windows-server/security/kerberos/configuring-kerberos-over-ip
192.168.240.209 WIN-UP5CAMCI1PF.abc.com  
192.168.240.208 WIN-BBT1IDC8T1N.abc.com  
192.168.240.207 WIN-KMVUE1IKP3O.abc.com  

![img_30.png](/pic/img_30.png)

怎么给加入域控的sqlserver做代理
不清楚代理那边的方案 按常理说是代理就是原来sqlserver服务器做什么现在一模一样伪造一套
使用SSMS连服务端 SSMS所在机器要加域 但是java客户端不用
