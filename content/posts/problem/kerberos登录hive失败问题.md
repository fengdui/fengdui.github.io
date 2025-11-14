---
title: "kerberos登录hive unable to obtain password from user"
date: "2025-06-24"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

使用运维发的kerberos认证文件登录hive本地测试成功 
上服务器报错 uable to obtain password from user
调试一波源码发现进入 
sun.security.krb5.internal.ktab.KeyTab#readServiceKeys
sun.security.jgss.krb5.Krb5Util#keysFromJavaxKeyTab
客户端和服务端两边的密码都对不上 也就是本地能行是kerberos配置文件里面的算法我本地java版本有
但是通过arthas jad服务器同一个类发现 支持的算法要少 java版本不一样

![img_32.png](/pic/img_32.png)

![img_33.png](/pic/img_33.png)

BUILTIN_ETYPES = new int[]{18, 17, 16, 23, 1, 3};
BUILTIN_ETYPES_NOAES256 = new int[]{17, 16, 23, 1, 3}

所以服务器的加密库算法库没有包含我现在需要的加密算法 所以无法提取密码 unable to obtain password from user
让运维以现有支持的算法重新导出一份kerberos文件