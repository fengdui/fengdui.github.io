---
title: "hbase日志接入"
date: "2021-07-22"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

## 分页怎么做
* 分页查询的时候，需要在rowkey上做范围查询，比如查询所有的商品，那么rowkey的范围就是[food_0001, food_9999]
* 需要计算好结束的rowkey，比如查询第1页，每页10条，那么结束的rowkey就是food_0010 
* 记得多取一条, 如果能取到+1条, 说明还有下一页, 给到前端一个下一页的标识
* 交互也需要注意下，各种分页参数, 直接定位到页数什么的是不支持的，只支持上一页下一页这种，如果是需要按照时间排序，那么需要在rowkey上设计好时间戳，比如“itemtype_itemid_timestamp”
* 比如，用户的rowkey的内容是“itemtype_itemid_timestamp”
* food_0001_1626944000000
* food_0002_1626944000000
* food_0003_1626944000000
* food_0004_1626944000000


## 热点key设计

# Salting prefix
* 比如，用户的rowkey的内容是“itemtype_itemid”
* food_0001
* food_0002
* food_0003
* food_0004
* 如果食品的商品占了大头，那么，food所在的region势必就会成为热点，那么就可以在rowkey的前面加一个随机bucket，假设做8个分桶，那么写入的时候随机加一个前缀{0,1,2,3,4,5,6,7}
* 4_food_0001
* 3_food_0002
* 7_food_0003
* 0_food_0004
* 因为前缀是随机的，所以，在读数据的时候，需要读8次【分桶数目】才能找到数据。同样，如果要扫描所有foot品类的商品的时候，
* 也需要做8次【分桶数目】才能找到。 放大读请求，用户需要读多次，同样前缀scan也一样
# Hashing prefix
* 同样，用户的rowkey的内容是“itemtype_itemid”
* food_0001
* food_0002
* food_0003
* food_0004
* rowkey的前面增加一个hashprefix = substr(md5sum(itemtype_itemid),0,4)_itemtype_itemid
* E0AE_food_0001
* 0A24_food_0002
* 87FA_food_0003
* F89B_food_0004
* 不能做scan，用户查询的时候，需要知道hash的key的部分，构造出完整的rowkey做get
# Reversed key
* 对于递增的rowkey，比如会员id，新生成的id的总是连续的，这种场景适用于 key reverse,比如连续生成的uid如下
* 0012345
* 0012346
* 0012347
* 0012348
* 0012349
* 其实和时间戳类似，这也会形成一个写热点，所以对rowkey做reverse操作：
* 5432100
* 6432100
* 7432100
* 8432100
* 9432100
* 这样就有效的规避了热点。
* 另外，对于需要将url请求作为hbase rowkey的场景，可以将作为rowkey的url反转，从而避免热点。

# 分区规划
* 每个RegionServer节点配置3-10个Region。也应考虑数据总量，HBase官方建议单个Region大小在10GB到50GB之间为宜，可根据预估数据量反推分区数
* 之前分区少了 线上直接分裂了
# 常见场景
* 监控数据，时间序列数据
* 即rowkey中包含时间戳或者递增部分，如果使用时间戳或者递增部分作为rowkey前缀，会导致漂移的热点，即，同一时刻写入会写到同一个region，然后随着时间的推移或者rowkey的增长，热点会移动到下一个区间的region。
* 解决的方式：避免使用时间戳或者递增序列作为rowkey前缀，比如如果是监控数据的话，使用metricname+timestamp可以规避时间热点
* 历史数据，订单数据，交易数据
* 一般这类数据都会有一个天然的前缀，比如订单数据的rowkey就可以设计成：
* shopid_orderNo,这样shopid就是天然的分桶，避免写热点，又可以轻易满足用户对店铺订单的查询和单个订单详情的查询