---
title: "esp-idf烧录固件至esp32开发板"
date: "2025-10-16"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

乐鑫官方物联网开发框架 ESP-IDF（Espressif IoT Development Framework）是乐鑫科技（Espressif Systems）为其ESP系列芯片（如ESP32、ESP32-S3等设计的官方开源软件开发框架  
vscode有插件 搜idf安装 可以在vscode里面使用esp-idf的一些功能 比如烧录固件 查看日志等  

按下面一路点下来即可  
![img_16.png](/pic/img_16.png)

menuconfig 配置项目

![img_17.png](/pic/img_17.png)

开发板的类型一定要选择对  
ota地址填你自己的 这个接口是 OTA版本和设备激活状态检查 他会请求这个地址去获取固件更新地址 mqtt连接地址 生成激活码  等会设备起来的时候会请求这个地址

monitor设备启动之后日志会有 W (928) Application: Alert 配网模式: 手机连接热点 Xiaozhi-BD2D 进行配网 这里也可以配置ota地址

配网成功后就会请求上面说的那个ota地址  

然后激活 后台就有了这个设备 这个看公司自己的业务逻辑

设备上有个按钮 点一下就会连接到后端 然后语言对话即可  


https://github.com/xinnan-tech/xiaozhi-esp32-server  设备连接后端  
https://dl.espressif.cn/dl/esp-idf/  
https://github.com/78/xiaozhi-esp32 固件
https://icnynnzcwou8.feishu.cn/wiki/JEYDwTTALi5s2zkGlFGcDiRknXf 案例  