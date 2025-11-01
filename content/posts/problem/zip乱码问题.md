---
title: "zip乱码问题"
date: "2023-06-30"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

环境  
echo $LANG   en_US.UTF-8  
原zip文件用命令解压：  
unzip -d test offlineEncryptTool.zip 不乱码   在压缩回去 在解压(不指定编码)不乱码  
unzip -O GBK -d test2 offlineEncryptTool.zip 不乱码 在压缩回去 在解压(不指定编码)不乱码  
unzip -O UTF-8 -d test3 offlineEncryptTool.zip  不乱码 在压缩回去 在解压(不指定编码)不乱码  

原zip文件用代码解压出来（解压出来是不乱码的）的文件在用代码压缩（代码中是utf-8）然后在用命令解压：  
unzip -d test offlineEncryptTool.zip 乱码  
unzip -O GBK -d test2 offlineEncryptTool.zip 不乱码  
unzip -O UTF-8 -d test3 offlineEncryptTool.zip  不乱码  
在windiws下解压 不乱码  

原zip文件用代码解压出来（解压出来是不乱码的）的文件在用代码压缩（代码中是gbk）然后在用命令解压：  
unzip -d test offlineEncryptTool.zip 乱码  
unzip -O GBK -d test2 offlineEncryptTool.zip 不乱码  
unzip -O UTF-8 -d test3 offlineEncryptTool.zip  乱码  

原zip文件用代码解压出来（解压出来是不乱码的）的文件在用命令压缩 然后再用命令解压：  
unzip -d test offlineEncryptTool.zip 不乱码   
unzip -O GBK -d test2 offlineEncryptTool.zip 不乱码  
unzip -O UTF-8 -d test3 offlineEncryptTool.zip  不乱码  

对比第一种情况和第四种情况 说明不是java代码解压的问题  
对于第二种和第四种场景 差别是压缩回去用的是java和命令 肯定是java压缩回去的不对  
发现用代码压缩回去的压缩包平台格式是dos  所以必须得加-O才可以 不加使用默认平台unix -i所以unzip -d test offlineEncryptTool.zip 乱码  

现在是想办法压缩回去还是unix的 在哪个平台压缩就用哪个平台  
![img_31.png](/pic/img_31.png)
奇怪的地方是 第一步源文件unzip -d test offlineEncryptTool.zip 不乱码  
可能第一步解压的源文件就是dos的 但是java压缩有一些处理 导致在不加-O解压有问题  
又或者是源文件是unix java压缩变成了dos 这不太可能  
源文件加不加-o都能解压出来不乱码 因为我在linux平台解压 不加就是-i 这说明了java压缩回去的文件不支持-o和-i了  
只能对应上 看一下压缩前的平台是什么 压缩之后的平台是什么 有没有改变 用java压缩之后是什么平台就只能-对应平台  
不知道java里面做了什么  

最后改完了  
unzip -d test offlineEncryptTool.zip 不乱码   
unzip -O GBK -d test2 offlineEncryptTool.zip 不乱码  
unzip -O UTF-8 -d test3 offlineEncryptTool.zip  不乱码  
这就又很奇怪了 -o -i都能解压出来不乱码  
说明之前有问题的java代码只能 -o不能-i  
正常的java代码和全程用命令的情况能够同时支持-o-i  