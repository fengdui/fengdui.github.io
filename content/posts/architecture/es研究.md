---
title: "es研究"
date: "2018-11-15"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

- 1 数据先写入到buffer里面，在buffer里面的数据时搜索不到的，同时将数据写入到translog日志文件之中
- 2 如果buffer快满了，或是一段时间之后，就会将buffer数据refresh到一个新的OS cache之中，
- 然后每隔1秒，就会将OS cache的数据写入到segment file之中，但是如果每一秒钟没有新的数据到buffer之中，
- 就会创建一个新的空的segment file，只要buffer中的数据被refresh到OS cache之中，
- 就代表这个数据可以被搜索到了。当然可以通过restful api 和Java api，
- 手动的执行一次refresh操作，就是手动的将buffer中的数据刷入到OS cache之中，
- 让数据立马搜索到，只要数据被输入到OS cache之中，buffer的内容就会被清空了。
- 同时进行的是，数据到shard之后，就会将数据写入到translog之中，每隔5秒将translog之中的数据持久化到磁盘之中
- 3 重复以上的操作，每次一条数据写入buffer，同时会写入一条日志到translog日志文件之中去，
- 这个translog文件会不断的变大，当达到一定的程度之后，就会触发commit操作。
- 4 将一个commit point写入到磁盘文件，里面标识着这个commit point 对应的所有segment file
- 5 强行将OS cache 之中的数据都fsync到磁盘文件中去。
- 解释：translog的作用：在执行commit之前，所有的而数据都是停留在buffer或OS cache之中，
- 无论buffer或OS cache都是内存，一旦这台机器死了，内存的数据就会丢失，
- 所以需要将数据对应的操作写入一个专门的日志问价之中，一旦机器出现宕机，再次重启的时候，
- es会主动的读取translog之中的日志文件的数据，恢复到内存buffer和OS cache之中。
- 6 将现有的translog文件进行清空，然后在重新启动一个translog，此时commit就算是成功了，
- 默认的是每隔30分钟进行一次commit，但是如果translog的文件过大，也会触发commit，整个commit过程就叫做一个flush操作，
- 我们也可以通过ES API,手动执行flush操作，手动将OS cache 的数据fsync到磁盘上面去，
- 记录一个commit point，清空translog文件
- 补充：其实translog的数据也是先写入到OS cache之中的，默认每隔5秒之中将数据刷新到硬盘中去，
- 也就是说，可能有5秒的数据仅仅停留在buffer或者translog文件的OS cache中，如果此时机器挂了，
- 会丢失5秒的数据，但是这样的性能比较好，我们也可以将每次的操作都必须是直接fsync到磁盘，但是性能会比较差。
- 7 如果时删除操作，commit的时候会产生一个.del文件，里面讲某个doc标记为delete状态，那么搜索的时候， 会根据.del文件的状态，就知道那个文件被删除了。
- 8 如果时更新操作，就是讲原来的doc标识为delete状态，然后重新写入一条数据即可。
- 9 buffer每次更新一次，就会产生一个segment file 文件，所以在默认情况之下， 就会产生很多的segment file 文件，将会定期执行merge操作
- 10 每次merge的时候，就会将多个segment file 文件进行合并为一个，同时将标记为delete的文件进行删除，
- 然后将新的segment file 文件写入到磁盘，这里会写一个commit point，标识所有的新的segment file， 然后打开新的segment file供搜索使用。
- 总之，segment的四个核心概念，refresh，flush，translog、merge


_cat/indices?pretty
curl -X PUT 192.168.245.65:9200/testindex1?pretty
curl -X PUT "192.168.245.65:9200/testindex1/testtype1/1?pretty" -H 'Content-Type: application/json' -d'
{
"name": "John Doe"
}'
http://192.168.245.65:9200/_search?q=*&pretty
http://192.168.245.65:9200/testindex1/_search?q=*&pretty

curl -u elastic:elastic -k https://192.168.202.74:9200/_alias