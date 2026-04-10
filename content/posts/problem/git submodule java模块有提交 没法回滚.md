---
title: "git submodule java模块有提交 没法回滚"
date: "2026-04-08"
tags: ["问题"]
ShowToc: false
TocOpen: false
---

项目parent 仓库是“聚合仓库”，子目录其实都是 submodule。parent 只记录每个子模块的 commit SHA，不记录子模块内部文件历史  
你本地子模块当前检出的 SHA（actual）  
和 parent 当前提交里记录的 SHA（expected）  
不一致了  
一旦不一致，parent 就会把这些子模块路径显示成 M（或在 submodule status 里显示 +），即使你没在 parent 里直接改源码文件  

因为我初始化工程之后 切了parent之后 子模块都是各种 a b c等等不同的SHA 但是我又统一切成了master 导致不一致了  
不应该这么操作 子模块应该跟着parent走 git submodule update --init --recursive 进行初始化到parent记录的子模块  

