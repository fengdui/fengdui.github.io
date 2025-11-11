---
title: "jasypt导致springboot无法加载bootstrap.yml"
date: "2023-08-10"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

项目里面接了jasypt，导致springboot无法加载bootstrap.yml  
在bootstrap.yml 中添加如下配置解决  
jasypt.encryptor.bootstrap=false  

在 Spring Cloud 环境中，存在两个 ApplicationContext：

Bootstrap ApplicationContext - 早期初始化，用于加载外部配置

Main ApplicationContext - 主应用上下文

如果 bootstrap: true（默认值或未设置），Jasypt 会尝试在 Bootstrap 阶段 就初始化并开始解密  
当 bootstrap: false 时：  
启动流程：
1. Bootstrap ApplicationContext 初始化
   → 加载 bootstrap.yml 中的原始属性（包括 ENC(...) 未解密状态）
   → Jasypt 不参与，跳过解密

2. Main ApplicationContext 初始化  
   → 加载 application.yml
   → Jasypt 配置生效
   → Jasypt 初始化并包装所有 PropertySource

3. 属性使用时
   → 通过包装后的 EncryptablePropertySource 获取
   → 实时解密 ENC(...) 值