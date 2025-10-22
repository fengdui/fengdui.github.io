---
title: "mycat入门"
date: "2017-09-07"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

Mycat 是数据库中间件，就是介于数据库与应用之间，进行数据处理与交互的中间服务。
mycat内部使用了Druid SQL Parser进行sql解析改写，然后将请求转发与结果合并。

# 概念——节点主机(dataHost)
![img_18.png](/pic/img_18.png)

```xml
<dataHost name="demo" maxCon="1000" minCon="10" balance="0"
	writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
	<heartbeat>select user()</heartbeat>
	<writeHost host="hostM1" url="localhost:3307" user="root" password="root">
		<readHost host="hostS2" url="localhost:3308" user="root" password="root" />
	</writeHost>
</dataHost>
```

- balance="0"，不开启读写分离机制，所有读操作都发送到当前使用的读节点上。
- balance="1"，除了当前使用的写节点之外的所有读写节点(如果只有当前写节点除外)
- balance="2"，所有读写节点上随机分发。
- balance="3"，当前读节点所属的写节点。

demo中配置了0，即读写都是往writeHost。

- writeType="0"，所有写操作发送到配置的第一个writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后以切换后的为准，切换记录在配置文件中:dnindex.properties。
- writeType="1"，所有写操作都随机的发送到配置的writeHost，1.5以后废弃不推荐。

# 概念——分片节点
dataNode标签定义了MyCat中的数据节点，也就是我们通常说所的数据分片。一个dataNode标签就是一个独立的数据分片。

```xml
<dataNode name="test1" dataHost="demo" database="db1" />
<dataNode name="test2" dataHost="demo" database="db2" />
```

```xml
<table name="role" dataNode="test1,test2" primaryKey="role_id" autoIncrement="true" rule="mod-long" >
```

role表的数据就是test1,test2这两个分片节点上的数据之和。

# 概念——逻辑库

```xml
<schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
```

schema标签用于定义MyCat实例中的逻辑库，MyCat可以有多个逻辑库，每个逻辑库都有自己的相关配置。

- **checkSQLschema**：当该值设置为true时，如果我们执行语句`select * from TESTDB.user;`则MyCat会把语句修改为`select * from user;`。即把表示schema的字符去掉，避免发送到后端数据库执行时报错，因为TESTDB只是一个逻辑库，提供SQL语句的最好是不带这个字段。

- **sqlMaxLimit**：当该值设置为某个数值时。每条执行的SQL语句，如果没有加上limit语句，MyCat也会自动的加上所对应的值。不设置该值的话，MyCat默认会把查询到的信息全部都展示出来，造成过多的输出。所以，在正常使用中，还是建议加上一个值，用于减少过多的数据返回。当然SQL语句中也显式的指定limit的大小，不受该属性的约束。

# 概念——逻辑表

```xml
<table name="test" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" ></table>
```

Table标签定义了MyCat中的逻辑表，所有需要拆分的表都需要在这个标签中定义。

- **rule属性**：该属性用于指定逻辑表要使用的分片规则名字，规则名字在rule.xml中定义。
- **primaryKey属性**：该逻辑表对应真实表的主键。
- **type**：该属性定义了逻辑表的类型，目前逻辑表只有"全局表"和"普通表"两种类型。全局表：global。普通表：不指定该值为globla的所有表。
- **autoIncrement**：设置true，在查询或者插入的时候如果没有主键，mycat会根据配置生成一个全局序列号，然后改写sql。
  - 原SQL：`insert into table1(name) values('test');`
  - 改写后：`insert into table1(id,name) values(next value for MYCATSEQ_GLOBAL,'test');`

# 全局序列号

## 1. 本地文件方式

原理：此方式MyCAT将sequence配置到文件中，当使用到sequence中的配置后，MyCAT会更下classpath中的sequence_conf.properties文件中sequence当前的值。

配置方式：在sequence_conf.properties文件中做如下配置：
```properties
GLOBAL_SEQ.HISIDS=
GLOBAL_SEQ.MINID=1001
GLOBAL_SEQ.MAXID=1000000000
GLOBAL_SEQ.CURID=1000
```

其中HISIDS表示使用过的历史分段(一般无特殊需要可不配置)，MINID表示最小ID值，MAXID表示最大ID值，CURID表示当前ID值。

server.xml中配置：
```xml
<system><property name="sequnceHandlerType">0</property></system>
```

注：sequnceHandlerType需要配置为0，表示使用本地文件方式。

使用示例：
```sql
insert into table1(id,name) values(next value for MYCATSEQ_GLOBAL,'test');
```

缺点：当MyCAT重新发布后，配置文件中的sequence会恢复到初始值。
优点：本地加载，读取速度较快。

## 2. 数据库方式

原理：在数据库中建立一张表，存放
- sequence名称(name)
- sequence当前值(current_value)
- 步长(increment int类型每次读取多少个sequence，假设为K可理解为mycat在数据库中一次读取多少个sequence。当这些用完后，下次再从数据库中读取)

Sequence获取步骤：
1. 当初次使用该sequence时，根据传入的sequence名称，从数据库这张表中读取current_value和increment到MyCat中，并将数据库中的current_value设置为原current_value值+increment值；
2. MyCat将读取到current_value+increment作为本次要使用的sequence值，下次使用时，自动加1，当使用increment次后，执行步骤1)相同的操作。

MyCat负责维护这张表，用到哪些sequence，只需要在这张表中插入一条记录即可。若某次读取的sequence没有用完，系统就停掉了，则这次读取的sequence剩余值不会再使用。

配置方式：
server.xml配置：
```xml
<system><property name="sequnceHandlerType">1</property></system>
```

注：sequnceHandlerType需要配置为1，表示使用数据库方式生成sequence

## 3. 本地时间戳方式

ID=64位二进制(42(毫秒)+5(机器ID)+5(业务编码)+12(重复累加))
换算成十进制为18位数的long类型，每毫秒可以并发12位二进制的累加。
除了最高位bit标记为不可用以外，其余三组bit占位均可浮动，看具体的业务需求而定。默认情况下41bit的时间戳可以支持该算法使用到2082年，10bit的工作机器id可以支持1023台机器，序列号支持1毫秒产生4095个自增序列id。

## 4. ZK ID生成器

还有2种使用zk生成全局序列号。

# mycat查询

## 分页查询（无offset，无orderby的情况）
```sql
select * from table limit 2;
```

返回的结果集取决于哪个DB节点最先返回结果给Mycat,相同情况下，同一个SQL，在Mycat上执行时会有不同的返回结果。

## 分页查询（无offset，有orderby的情况）
```sql
select * from table order by id limit 2;
```

接收到各个DB节点的返回结果后，对其进行最小堆运算，计算出所有结果集中最小的两条记录返回给应用。

## 分页查询（有offset，有orderby的情况）
```sql
select * from table order by id limit 5,2;
```

limit m,n的SQL语句时会对其进行改写，改写成limit 0,m+n来保证查询结果的逻辑正确性。所以，Mycat发送到后端DB上的SQL语句是`select * from table order by id limit 0,7;`

对于有t个DB节点的全分片limit m,n操作，Mycat需要处理的数据量为(m+n)*t个。比如实际应用中有50个DB节点，要执行limit 1000,10操作，则Mycat处理的数据量为50500条，返回结果集为10，当偏移量更大时，内存和CPU资源的消耗则是数十倍增加。

# 事务

用户会话Session中设定autocommit=false，开启一个事务过程，这个会话中随后的所有SQL语句进入事务模式，ServerConnection（前端连接）中有一个变量txInterrupted控制是否事务异常需要回滚。

当某个SQL执行过程中发生错误，则设置txInterrupted=true，表明此事务需要回滚。当用户提交事务（commit指令）的时候，Session会检查事务回滚变量，若发现事务需要回滚，则取消Commit指令在相关节点上的执行过程，返回错误信息，Transaction need rollback，用户只能回滚事务，若所有节点都执行成功，则向每个节点发送Commit指令，事务结束。

# 弱XA

Mycat的事务是一种弱XA的事务，与XA事务相似的地方是，只有所有节点都执行成功（Prepare阶段都成功），才开始提交事务，与XA不同的是，在提交阶段，若某个节点宕机，没有手段让此事务在故障节点恢复以后继续执行。

begin开启事务，Mycat不会立即把命令发送到DB节点上，等后续下发SQL时，Mycat从连接池获取非自动提交的连接去执行。

Mycat会等待各个节点的返回结果，如果都执行成功，Mycat给该连接标识为Prepare Ready状态，如果有一个节点执行失败，则标识为Rollback状态。

执行完成后Mycat等待前端发送commit或rollback命令。发送commit命令时，Mycat检测当前连接是否为Prepare Ready状态，若是，则将commit命令发送到各个DB节点。

但是，这一阶段是无法保证一致性的，如果一个DB节点在commit时故障，而其他DB节点commit成功，Mycat会一直等待故障DB节点返回结果。Mycat只有收到所有DB节点的成功执行结果才会向前端返回执行成功的包，此时Mycat只能一直waiting直至TIMEOUT，导致事务一致性被破坏。

# 分片规则

## 1. 分片枚举 PartitionByFileMap

通过在配置文件中配置可能的枚举id，自己配置分片，本规则适用于特定的场景，比如有些业务需要按照省份或区县来做保存，而全国省份区县固定的，这类业务使用本条规则。

支持数字和字符串：
```properties
10000=0
10010=1
杭州=2
```

## 2. 固定分片hash算法 PartitionByLong

|<———————1024———————————>|
//|<—-256—>|<—-256—>|<———-512————->|
//| partition0 | partition1 | partition2 |

对2^10取模，然后根据配置判断在哪一段中。

## 3. 范围约定 AutoPartitionByLong

此分片适用于，提前规划好分片字段某个范围属于哪个分片：
```properties
0-500M=0
500M-1000M=1
```

## 4. 取模 PartitionByMod

mod分片节点数

## 5. 按日期（天）分片 PartitionByDate

dateFormat：日期格式
sBeginDate：开始日期
sEndDate：结束日期
sPartionDay：分区天数，即默认从开始日期算起，分隔10天一个分区

## 6-15 其他分片规则

6. 取模范围约束
7. 截取数字做hash求模范围约束
8. 应用指定
9. 截取数字hash解析
10. 一致性hash
11. 按单月小时拆分
12. 范围求模分片
13. 日期范围hash分片
14. 冷热数据分片
15. 按月份列分区，每个自然月一个分片。

# 分片join

Mycat目前版本支持跨分片的join，主要实现的方式有四种：全局表，ER分片，catletT(人工智能)和ShareJoin。

## 1. 字典表

如果你的业务中有些数据类似于数据字典，比如配置文件的配置，常用业务的配置或者数据量不大很少变动的表，这些表往往不是特别大，而且大部分的业务场景都会用到，那么这种表适合于Mycat全局表，无须对数据进行切分，只要在所有的分片上保存一份数据即可，Mycat在Join操作中，业务表与全局表进行Join聚合会优先选择相同分片内的全局表join，避免跨库Join，在进行数据插入操作时，mycat将把数据分发到全局表对应的所有分片执行，在进行数据读取时候将会随机获取一个节点读取数据。

全局表的插入、更新操作会实时在所有节点上执行，保持各个分片的数据一致性。
全局表的查询操作，只从一个节点获取。
全局表可以跟任何一个表进行JOIN操作。

全局表配置比较简单，不用写Rule规则，如下配置即可：
```xml
<table name="company" primaryKey="ID" type="global" dataNode="dn1,dn2,dn3" />
```

需要注意的是，全局表每个分片节点上都要有运行创建表的DDL语句。

## 2. ER Join

```xml
<table name="role" dataNode="test1,test2" primaryKey="role_id" autoIncrement="true" rule="mod-long" >
<childTable name="user" primaryKey="user_id" joinKey="role_id" parentKey="role_id" autoIncrement="true"/>
</table>
```

只能支持一对一和一对多的情况。

## 3. Share join

目前支持2个表的join，原理就是解析SQL语句，拆分成单表的SQL语句执行，然后把各个节点的数据汇集。

```sql
/*!mycat:catlet=demo.catlets.ShareJoin */ select a.*,b.id, b.name as tit from customer a,company b
on a.company_id=b.id;
```

## 4. catlet（不介绍）

mycat提供api，通过编程解决业务系统中特定几个必须跨分片的SQL的JOIN逻辑。

## 5. Spark/Storm对join扩展

mycat后续的功能会引入spark和storm来做跨分片的join，大致流程是这样的在mycat调用spark,storm的api，把数据传送到spark,storm，在spark,storm进行join，在把数据传回mycat，mycat在返回给客户端。

未来2.0以上版本可能支持。