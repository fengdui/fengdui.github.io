---
title: "mybatis的sqlsession模型和spring事务"
date: "2020-12-05"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
# mybatis的sqlsession模型
- 我们知道，在spring中配置FactoryBean,实际上会调用其getObject方法注册到spring容器中，
- 我们使用mybatis一般会使用MapperScannerConfigurer扫码指定包下的Mapper接口,生成MapperFactoryBean,所以我们在service中注入的dao一般就是MapperFactoryBean#getObject()的返回值。
- 返回的代理如下:
![img_37.png](/pic/img_37.png)
![img_38.png](/pic/img_38.png)
# 多数据源
- 在项目中使用了2个datasource,因此使用了ChainedTransactionManager代理多个DataSourceTransactionManager。
- 对于代理的每个事务管理器都开启事务
```
try {
    for (PlatformTransactionManager transactionManager : transactionManagers) {
        mts.registerTransactionManager(definition, transactionManager);
    }
}
public void registerTransactionManager(TransactionDefinition definition, PlatformTransactionManager transactionManager) {
    getTransactionStatuses().put(transactionManager, transactionManager.getTransaction(definition));
}
```
- 每个事务管理器都包含不同的datasource, 对带有@Transactional注解的方法,进入切面时需要开启新事务的时候会从datasource中获取一个connection, 然后放到threadlocal中。
```
if (txObject.isNewConnectionHolder()) {
    TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
}
```
mybatis获取threadLocal中的connection。
- sqlsession和connection的关系
- 之前说到mybatis创建sqlSession是在SqlSessionInterceptor中,该类实现了InvocationHandler。
```
private class SqlSessionInterceptor implements InvocationHandler {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
        SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
    try {
      Object result = method.invoke(sqlSession, args);
      if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
        // force commit even on non-dirty sessions because some databases require
        // a commit/rollback before calling close()
        sqlSession.commit(true);
      }
      return result;
    } catch (Throwable t) {
      Throwable unwrapped = unwrapThrowable(t);
      if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
        // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        sqlSession = null;
        Throwable translated = SqlSessionTemplate.this.exceptionTranslator
            .translateExceptionIfPossible((PersistenceException) unwrapped);
        if (translated != null) {
          unwrapped = translated;
        }
      }
      throw unwrapped;
    } finally {
      if (sqlSession != null) {
        closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
      }
    }
  }
}
```
- 一路追踪下去,发现最后使用的transaction的实现类是SpringManagedTransaction,从threadlocal中获取连接, 保存时和获取时的key都是datasource,从而保证不同的sqlsessionfactory获取正确的连接。
- 所以如果事务管理器和sqlsessionfactory对应的datasource不一致，将获取不到对应的连接, 导致事务无法回滚的问题。
```
private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
 
    LOGGER.debug(() -> "JDBC Connection [" + this.connection + "] will"
        + (this.isConnectionTransactional ? " " : " not ") + "be managed by Spring");
}
```
- 最后事务提交，将所有threadlocal中不同数据源的connection commit或者rollback。