---
title: "mysql索引使用原则"
date: "2021-06-11"
tags: ["架构"]
ShowToc: false
TocOpen: false
---



覆盖索引  
延迟关联  
通过使用覆盖索引查询返回需要的主键,再根据主键关联原表获得需要的数据。  
eq_ref 关联查询 使用唯一索引或者主键  
ref 普通索引  
using where是指优化器需要通过索引回表查询数据  
using index 索引覆盖  
using index condition 查询的列不全在索引中，where条件中是一个前导列的范围  
possible_keys：可能在哪个索引找到记录。  
key：实际使用索引。  
ref：哪些列，或者常量用于查找索引上的值。  
MySQL limit不会传递到引擎层，只是在服务层进行数据过滤。查询数据时，先由引擎层通过索引过滤出一批数据（索引过滤），然后服务层进行二次过滤（非索引过滤）。  
引擎层过滤后会将获取的数据暂存，服务层一条一条数据获取，获取时引擎层回表获得完成数据交给服务层，服务层判断是否匹配查询条件（非索引过滤），如果匹配会继续判断是否满足limit限制的数据范围，符合并且范围内的数据都查完了才返回。  

谓词下推  
比如 select * from t1, t2 where t1.a > 3 and t2.b > 5 ，假设 t1 和 t2 都是 100 条数据。如果把 t1 和 t2 两个表做笛卡尔集了再过滤，我们要处理 10000 条数据，  
而如果先做过滤条件，那么数据量就会少很多。这就是谓词下推的作用，我们尽量把过滤条件下靠近叶子节点做，可以减少访问许多数据。  

insert 语句会先在插入间隙上加上插入意向锁，然后开始写数据，写完数据之后再对记录加上 X 记录锁  
insert ……on duplicate key update 的加锁流程其实就是 insert 和 update 的结合，  
如果没有唯一键冲突，和 insert 加锁流程一样，如果有唯一键冲突，会对唯一键加 Next-Key 锁（lock_mode X），  
对主键加记录锁（lock_mode X locks rec but not gap），然后再执行 update 操作。  
记录锁（LOCK_REC_NOT_GAP）: lock_mode X locks rec but not gap  
间隙锁（LOCK_GAP）: lock_mode X locks gap before rec  
Next-key 锁（LOCK_ORNIDARY）: lock_mode X  
插入意向锁（LOCK_INSERT_INTENTION）: lock_mode X locks gap before rec insert intention  
如果在 supremum record 上加锁，  
locks gap before rec会省略掉，间隙锁会显示成 lock_mode X，插入意向锁会显示成 lock_mode X insert intention。  
间隙锁和间隙锁并不冲突  
插入意向锁和插入意向锁之间互不冲突 插入意向锁只会和间隙锁或 Next-key 锁冲突  
插入意向锁不影响其他事务加其他任何锁 插入意向锁与间隙锁和 Next-key 锁冲突  
insert on duplicate key update 如果命中主键或者唯一键索引，加行锁，未命中加gap锁，即会阻塞插入数据  
bug在5.7.26以及8.0.15版本上已经修复了，当插入数据时，不会在形成间隙锁  
MySQL limit不会传递到引擎层，只是在服务层进行数据过滤。查询数据时，先由引擎层通过索引过滤出一批数据（索引过滤），然后服务层进行二次过滤（非索引过滤）。  
rr级别快照查询不存在的数据 然后查询这个不存在的数据 在查询会得到这个数据 幻读    
semi-consistent read  
https://www.aneasystone.com/archives/2017/12/solving-dead-locks-three.html  
https://mp.weixin.qq.com/s/wSlNZcQkax-2KZCNEHOYLA  
https://mp.weixin.qq.com/s/_36Sy0FldFRNvLRpHxfucQ  