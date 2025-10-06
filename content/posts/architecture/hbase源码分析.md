---
title: "hbase源码分析"
date: "2017-12-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

hbase regionLocator 获取region过程
- htable#get 方法
- RequestConverter.buildGetRequest(getLocation().getRegionInfo().getRegionName(), getReq);
- 这句话是位于 创建的callable中
- 得到的location是在prepare方法中
- RegionServerCallable#prepare方法

```
public void prepare(final boolean reload) throws IOException {
    try (RegionLocator regionLocator = connection.getRegionLocator(tableName)) {
        this.location = regionLocator.getRegionLocation(row, reload);
    }
    if (this.location == null) {
        throw new IOException("Failed to find location, tableName=" + tableName +
        ", row=" + Bytes.toString(row) + ", reload=" + reload);
    }
    setStub(getConnection().getClient(this.location.getServerName()));
}
```

- connection.getRegionLocator(tableName) 这里的connection是HConnectionImplementation
- HRegionLocator#getRegionLocation(final byte [] row, boolean reload)
- 然后一路走到了RegionLocations locateRegion(final TableName tableName,
- final byte [] row, boolean useCache, boolean retry, int replicaId)

```
if (tableName.equals(TableName.META_TABLE_NAME)) {
    return locateMeta(tableName, useCache, replicaId);
} else {
    // Region not in the cache - have to go to the meta RS
    return locateRegionInMeta(tableName, row, useCache, retry, replicaId);
}

```

- 这里判断了是否查找mete表 这里还不是，待会会再回来的
- HRegionLocator#locateRegionInMeta

```
try {
    Result regionInfoRow = null;
    ReversedClientScanner rcs = null;
    try {
        rcs = new ClientSmallReversedScanner(conf, s, TableName.META_TABLE_NAME, this,
        rpcCallerFactory, rpcControllerFactory, getMetaLookupPool(), 0);
        regionInfoRow = rcs.next();
    } finally {
        if (rcs != null) {
        rcs.close();
    }
}

```

- 上面代码是得到regionInfoRow，可以看到tableName是mete表名
- 这里面是ClientSmallReversedScanner#loadCache
- loadcache这个方法先调用nextScanner 创建了ScannerCallableWithReplicas类型的smallReversedScanCallable作为ClientSmallReversedScanner的成员变量
- 这个ScannerCallableWithReplicas内部维护这着一个SmallReversedScannerCallable
- 调用玩nextScanner 之后，调用caller，这个caller是RpcRetryingCaller，是在new ClientSmallReversedScanner的时候构造函数里面创建的
- values = this.caller.callWithoutRetries(smallReversedScanCallable, scannerTimeout);
- 这个caller又会调用smallReversedScanCallable，smallReversedScanCallable#call方法
- callRegionLocations rl = RpcRetryingCallerWithReadReplicas.getRegionLocations(true,
- RegionReplicaUtil.DEFAULT_REPLICA_ID, cConnection, tableName,
- currentScannerCallable.getRow());
- 点进去发现

```
if (!useCache) {
    rl = cConnection.relocateRegion(tableName, row, replicaId);
} else {
    rl = cConnection.locateRegion(tableName, row, useCache, true, replicaId);
}
```

- 又回到了前面的ConnectionManager#locateRegion
- 这里tablename已经是mete了所以走locateMeta方法，这里就简单了 获取zookeeper上的hbase/meta-region-server节点下的信息
- 返回一个RegionLocations对象
- 然后就一路返回