---
title: "nlp入门"
date: "2016-06-19"
tags: ["人工智能"]
ShowToc: false
TocOpen: false
---
词干 去掉所有词缀后的单词基础形式，如单词“hearing”和“hears”中的“hear”即为词干  词干 是基于规则的、粗暴的砍削，结果可能不是一个真正的单词  
词元 是基于词典的、精细的映射，结果一定是一个有意义的、标准的单词（即词典中的词条）  
词根 词根是一个单词中最基本、不可再分的有意义单位。它承载着最核心的词汇意义。一个词根可以派生出许多不同的单词 不一定能独立成词：例如 -ceive- (在 receive, perceive 中), -struct- (在 construct, destruction 中) 都不能单独使用。  
在 uncomfortably 中： 我们可以将其分解：un- (前缀) + comfort (词根) + -able (后缀) + -ly (后缀)。 这里的 comfort 就是词根。它是所有相关单词（comfortable, discomfort, comforting）的意义来源。  
停用词  
词汇  
生词 首次遇到的新的、不熟悉的词汇,需要借助上下文或工具来理解其含义  
词形归并 是将单词的多种屈折形式（有时也包括派生形式）归并到一个统一的标准形式下 同一词根的不同拼写形式（如“running”→“run”）合并为标准形式  
词干还原 词干还原通过规则截断词尾（如“running”→“runn”），可能生成非词典形式  
one-hot编码  
在one-hot编码中，每一个token使用一个长度为N的向量表示，N表示词典的数量。one-hot编码又称独热编码，将每个词表示成具有n个元素的向量，这个词向量中只有一个元素是1，其他元素都是0，不同词汇元素为0的位置不同，其中n的大小是整个语料中不同词汇的总数  
word embedding  
将单词转化为向量也可以称为词嵌入 word embedding  
Sentence Embedding  
word2vec模型，简单来说，它就是通过一个个句子去发掘出词与词之间的关系，再通过向量去表示出这种关系 分为 CBOW (Continuous Bag-of-Words) 和 Skip-gram 两种  
CBOW 使用周围的词语 w(t-2),w(t-1),w(t+1),w(t+2) 来预测当前词 w(t)，而 Skip-gram 则正好相反，它使用当前词 w(t) 来预测它的周围词语  
GloVe是通过整个语料库所有的句子去分析词与词之间的关系，  
共现矩阵co-occurrence matrix  
余弦相似度  
![img_41.png](/pic/img_41.png)
句子向量=所含词向量的平均值  
词元化是“分割”过程。它不改变单词的语义，只是将它拆分成更小的零件。它旨在保留所有原始信息。输入输出是不同层面的东西 对于像BERT、GPT、Qwen这样的大模型，我们只使用词元化，绝对不使用词干还原或词形归并。  
