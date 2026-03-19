---
title: "funasr 3dspeaker压测"
date: "2025-12-01"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
自己给公司做funasr 3dspeaker的压测, 又没有pgu资源, 只能在4c8g的服务器上做压  
单次跑大概0.2s左右 多线程压tps大概5左右 即单线程和多线程效果差不多  
因为这些都是cpu密集型任务, 单次0.2s 一个cpu一直跑也就5tps 而且内部是单线程加锁互斥独占模型 或者模型内部加锁 单个模型多线程没啥效果  
个人理解如果非要用cpu上生产 内部使用多线程或者线程池 每个线程独占模型 池化技术  
https://github.com/modelscope/FunASR  
https://github.com/modelscope/3D-Speaker  
