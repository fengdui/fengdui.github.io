---
title: "Atomikos分布式事务"
date: "2017-07-04"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
```
@Bean(name="dmXaDataSource")
public DataSource dmXaDataSource(){
    AtomikosDataSourceBean dataSource = new AtomikosDataSourceBean();
    dataSource.setUniqueResourceName("dm");
    dataSource.setXaDataSourceClassName(getXaDataSourceClassName(jdbcType));
    Properties properties = new Properties();
    if("oracle".equals(jdbcType)){
        properties.put("URL", jdbcUrl);
    }
    else{
        properties.put("url", jdbcUrl);
    }
    
    properties.put("user", username);
    properties.put("password", password);
    dataSource.setXaProperties(properties);
    dataSource.setMaxPoolSize(10);
    dataSource.setTestQuery(connectionTestQuery);
    return dataSource;
}
```
```
private String getXaDataSourceClassName(String jdbcType){
    if("mysql".equals(jdbcType)){
        return "com.mysql.jdbc.jdbc2.optional.MysqlXADataSource";
    }
    else if("oracle".equals(jdbcType)){
        return "oracle.jdbc.xa.client.OracleXADataSource";
    }
    
    return "com.mysql.jdbc.jdbc2.optional.MysqlXADataSource";
}
```
然后要把dmXaDataSource注册到sqlsessionfactory中  
XA事务支持：通过AtomikosDataSourceBean创建支持XA协议的数据源，能够参与分布式事务  
资源标识：每个XA数据源都有唯一标识(uniqueResourceName)，用于在分布式事务中识别不同资源  
连接池管理：Atomikos提供连接池功能，配置了最大连接数等参数  
多数据库支持：系统能够根据数据库类型自动适配不同的XA数据源实现  

AtomikosDataSourceBean实现了XA协议，能够参与分布式事务协调：
管理事务分支和资源协调
支持两阶段提交（2PC）
确保跨多个数据库操作的原子性

spring事务管理器集成：AtomikosTransactionManager能够与Spring的事务管理机制无缝集成，实现声明式事务管理
```
@Bean(name="dmTransactionManager")
public PlatformTransactionManager dmTransactionManager(
        final @Qualifier("dmXaDataSource") DataSource dmXaDataSource) {
    return new AtomikosTransactionManager();
}
```
