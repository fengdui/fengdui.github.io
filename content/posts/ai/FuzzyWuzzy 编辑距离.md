---
title: "FuzzyWuzzy 编辑距离"
date: "2026-03-26"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---
FuzzyWuzzy 的核心算法基础是 Levenshtein 距离（编辑距离），它衡量的是：将一个字符串转换成另一个字符串，最少需要多少次“单字符编辑操作”。  
Partial Ratio 使用滑动窗口 + 子串匹配 优化了算法  