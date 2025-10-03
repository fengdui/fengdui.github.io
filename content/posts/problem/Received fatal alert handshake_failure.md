---
title: "Received fatal alert: handshake_failure"
date: "2024-08-09"
tags: ["问题"]
ShowToc: false
TocOpen: false
---




![img_4.png](/pic/img_4.png)
可以写个demo 启动参数加上-Djavax.net.debug=ssl 看一下日志
这个问题一般是算法套件客户端和服务端不一致，java发请求不行，但是curl是可以的，显然是java支持的算法和服务端没有交集。

查看服务端支持的算法
nmap --script ssl-enum-ciphers -p 443 in.xxx.be.srv.com

![img_5.png](/pic/img_5.png)

查看客户端java支持的算法
bin/jrunscript -e "print(java.util.Arrays.toString(javax.net.ssl.SSLServerSocketFactory.getDefault().getSupportedCipherSuites()))"

打印出来的显然没有

TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_RSA_WITH_AES_256_GCM_SHA384

又去查了资料，使用JDK 1.8.0_161或更高版本时，不需要再安装JCE Policy File。JDK 1.8.0_161默认启用无限强度加密。

手动安装一下http://www.oracle.com/technetwork/java/javase/downloads/jce8-download-2133166.html 可以了。

https://curl.se/docs/ssl-ciphers.html