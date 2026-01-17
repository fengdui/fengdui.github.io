---
title: "pt online schema change 在线变更表结构"
date: "2019-05-10"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

假设我们有一个大表 user，现在想给它添加一列 age，执行命令：pt-online-schema-change --alter "ADD COLUMN age INT" D=database,t=user ...

步骤1：前置检查与创建新表
检查表结构、主键、是否存在触发器、外键约束等。

根据原表结构 user，创建一个新的、空的目标表 _user_new。

对 _user_new 表执行指定的 DDL 语句（即 ADD COLUMN age INT）。此时，原表 user 不受任何影响。

步骤2：创建触发器（实现增量同步）
这是保证数据一致性的最关键环节。PT-OSC 会在原表 user 上创建三个 AFTER 触发器（AFTER 是为了避免因触发器失败而影响原业务）：

pt_osc_database_user_ins：当原表有 INSERT 操作时，将这条新数据也插入到 _user_new 表中。

pt_osc_database_user_upd：当原表有 UPDATE 操作时，将对应行在 _user_new 表中也进行更新。

pt_osc_database_user_del：当原表有 DELETE 操作时，将对应行在 _user_new 表中也进行删除。

这些触发器确保了从数据复制开始到结束的这段时间内，对原表的所有写操作都不会丢失，都会体现在新表中。

步骤3：分批复制数据（全量迁移）
PT-OSC 会根据主键或唯一索引，将原表 user 的数据按块（Chunk）划分。

以可控的速度（通过 --chunk-size 和 --chunk-time 参数调节），逐块地将数据从 user 复制到 _user_new。

在复制每一块时，会使用 SELECT ... FOR UPDATE 之类的机制来保证这一小块数据的一致性，但锁定时间非常短。

由于步骤2的触发器在同时工作，在复制过程中，如果某个已经被复制的行发生了更新，触发器会保证新表中的该行也被更新。

步骤4：原子切换
当数据复制完成，并且增量数据通过触发器也同步得几乎一致时（有一个短暂的追赶期），PT-OSC 会执行一个原子性切换：

sql
RENAME TABLE database.user TO database._user_old, database._user_new TO database.user;
这个 RENAME 操作在 MySQL 中是原子的，即瞬间完成。完成后：

原表 user 变成了 _user_old。

拥有新结构的新表 _user_new 变成了 user，开始为应用程序提供服务。

此时应用程序可能会有一个非常短暂的连接中断，但通常无感知。

步骤5：清理工作
删除旧表 _user_old。

删除在原表（现在已经改名为 _user_old）上创建的三个触发器。

至此，整个在线表结构变更完成。