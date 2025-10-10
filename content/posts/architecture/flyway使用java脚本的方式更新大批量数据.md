---
title: "flyway使用java脚本的方式更新大批量数据"
date: "2024-10-30"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


之前项目是用了flyway去自动升级数据库，进行表结构的变更，以及dml操作，现在出现了千万级的数据，单个update sql无法方便的进行数据更新，甚至还有数据迁移，添加索引，rename表的操作。

本着不造轮子的原则，当我们遇到问题，基本上前人已经遇到过，并且已经有现成的解决方案，而且flyway没那么弱，肯定有这种功能。

目前使用sql升级的方式如下

    FluentConfiguration configuration = new FluentConfiguration();
    configuration.dataSource(dynamicDataSource);
    configuration.schemas(tenant);
    //仅加载upgrade的DML
    configuration.locations("classpath:db/init", "classpath:db/test");

    Flyway flyway = configuration.load();
    MigrateResult migrateResult = flyway.migrate();

点击locations方法进入，注释上说，可以指定一个包名，包里面可以放入sql和java代码。
于是写个java类，继承BaseJavaMigration。

    package db.test;

    import lombok.extern.slf4j.Slf4j;
    import org.flywaydb.core.api.migration.BaseJavaMigration;
    import org.flywaydb.core.api.migration.Context;

    @Slf4j
    public class V3_20_13__data extends BaseJavaMigration {
        @Override
        public void migrate(Context context) throws Exception {
            log.info("migrateStart");
        }

        @Override
        public Integer getChecksum() {
            return 32013;
        }
    }

试一下果然行，这里面只是打了个日志，具体怎么分批可以context.getConnection()拿到连接后业务上自行处理。

![img_10.png](/pic/img_10.png)

刚开始没有实现checkSum方法, flyway\_schema\_history表里的checkSum是null, 后面阅读源码发现，checkSum是直接读的 org.flywaydb.core.api.migration.JavaMigration#getChecksum方法, 那我直接实现这个方法就行。

ScanningJavaMigrationResolver， 扫描java类，
The classes must have a name like R\_\_My\_description, V1\_\_Description or V1\_1\_3\_\_Description。

FixedJavaMigrationResolver 这个可以直接实例化一些迁移类去做迁移工具，不用扫描路径，直接设置进去configuration，例如

    FluentConfiguration configuration = new FluentConfiguration();
    configuration.javaMigrations();

SqlMigrationResolver，扫描sql脚本，The SQL files must have names like V1\_\_Description.sql, V1\_1\_\_Description.sql, or R\_\_description.sql。
