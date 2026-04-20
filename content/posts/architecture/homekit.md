---
title: "homekit"
date: "2021-12-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
HomeKit 是苹果公司推出的智能家居平台

说白了就是接入苹果生态 自己生产的设备接入之后可以用苹果的app控制

当工厂在生产支持HomeKit的智能设备（如智能灯、插座）时，需要将atoken烧录（写入）到设备的固件中。这个过程是与苹果的服务器交互来完成的 这个token是苹果颁发的

Setup Code则是用户为设备首次配网时所需的临时"配对码"