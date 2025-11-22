---
title: "redis架构"
date: "2018-03-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

# 主从复制模式 (Master-Slave Replication)
```plaintext
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Redis Master  │───▶│   Redis Slave   │───▶│   Redis Slave   │
│    (Write)      │    │    (Read)       │    │    (Read)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

主节点配置 (redis-master.conf)  
port 6379  
从节点配置 (redis-slave.conf)  
port 6380  
replicaof 127.0.0.1 6379  

# 哨兵模式 (Sentinel)
```plaintext
┌─────────────────┐    ┌─────────────────┐
│  Sentinel       │    │  Sentinel       │
│   Cluster       │    │   Cluster       │
└─────────────────┘    └─────────────────┘
        │                       │
        ▼                       ▼
┌─────────────────┐    ┌─────────────────┐
│   Redis Master  │◄───│   Redis Slave   │
└─────────────────┘    └─────────────────┘
```

sentinel.conf  
sentinel monitor mymaster 127.0.0.1 6379 2  
sentinel down-after-milliseconds mymaster 5000  
sentinel failover-timeout mymaster 10000  
sentinel parallel-syncs mymaster 1  

```
JedisSentinelPool pool = new JedisSentinelPool(
    "mymaster",
    new HashSet<String>(Arrays.asList("sentinel1:26379", "sentinel2:26379")),
    jedisPoolConfig
);
```
# 集群模式 (Cluster)
```plaintext
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node 1    │  │   Node 2    │  │   Node 3    │
│  Master     │  │  Master     │  │  Master     │
│  Slot 0-5460│  │Slot 5461-10922││Slot 10923-16383│
└─────────────┘  └─────────────┘  └─────────────┘
        │               │               │
        ▼               ▼               ▼
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Node 4    │  │   Node 5    │  │   Node 6    │
│   Slave     │  │   Slave     │  │   Slave     │
└─────────────┘  └─────────────┘  └─────────────┘
```

每个节点的 redis.conf  
port 6379  
cluster-enabled yes  
cluster-config-file nodes-6379.conf  
cluster-node-timeout 15000  

启动所有节点  
redis-server redis-6379.conf  
redis-server redis-6380.conf

创建集群  
redis-cli --cluster create \  
127.0.0.1:6379 127.0.0.1:6380 127.0.0.1:6381 \  
127.0.0.1:6382 127.0.0.1:6383 127.0.0.1:6384 \  
--cluster-replicas 1  

# 代理模式 (Proxy)

```plaintext
┌─────────────────┐
│   Redis Proxy   │
│   (Twemproxy,   │
│    Codis)       │
└─────────────────┘
        │
        ▼
┌─────────────────┐  ┌─────────────────┐
│   Redis         │  │   Redis         │
│   Cluster       │  │   Cluster       │
└─────────────────┘  └─────────────────┘
```