---
title: "RoaringBitmap"
date: "2026-02-20"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

BitMap的问题在于，不管业务中实际的元素基数有多少，它占用的内存空间都恒定不变。  
如果BitMap中的位的取值范围是1到100亿之间，那么BitMap就会开辟出100亿Bit的存储空间。  
但是如果实际上值只有100个的话，100亿Bit的存储空间只有100Bit为1，其余全部为0，数据存储空间浪费严重，数据越稀疏，空间浪费越严重。  

RoaringBitmap将数据的前半部分，即2^16(这里为高16位)部分作为桶的编号，将分为2^16=65536个桶，RBM中将这些小桶称之为Contriner(容器)  
此时Contriner并没有创建 存储数据时，按照数据的高16位做为Contriner的编号去找对应的Contriner(找不到就创建对应的Contriner)，再将低16位放入该Contriner中  

共有4种Container：arraycontainer(数组容器)，bitmapcontainer(位图容器)，runcontainer(行程步长容器)，sharedcontainer(共享容器)  
在创建一个新Container时，如果只插入一个元素，RBM默认会用ArrayContainer来存储。其中每一个元素的类型为 short int 占两个字节。  
当ArrayContainer的容量超过4096后，会自动转成BitmapContainer  
runcontainer(行程步长容器)  
这是一种利用步长来压缩空间的方法  
比如连续的整数序列 11, 12, 13, 14, 15, 27, 28, 29 会被压缩为两个二元组 11, 4, 27, 2 表示：11后面紧跟着4个连续递增的值，27后面跟着2个连续递增的值  