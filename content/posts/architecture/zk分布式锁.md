---
title: "zk分布式锁"
date: "2022-11-15"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


```
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.List;
import java.util.TreeSet;
import java.util.concurrent.CountDownLatch;

public class DistributedLock {
    Logger logger = LoggerFactory.getLogger(DistributedLock.class);
    /**
     * zk客户端
     */
    private ZooKeeper zk;
    /**
     * zk是一个目录结构，root为最外层目录
     */
    private String root = "/locks";
    private String lockName;
    private ThreadLocal<String> nodeId = new ThreadLocal<>();
    private CountDownLatch connectedSignal = new CountDownLatch(1);
    /**
     * session失效时间，3秒后zookeeper自动释放节点，解决了死锁的问题
     */
    private final static int sessionTimeout = 3000;
    private final static byte[] data = new byte[0];

    public DistributedLock(String config, String lockName) {
        this.lockName = lockName;
        try {
            zk = new ZooKeeper(config, sessionTimeout, event -> {
                // 建立连接
                if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                    connectedSignal.countDown();
                }
            });
            connectedSignal.await();
            Stat stat = zk.exists(root, false);
            if (null == stat) {
                // 创建根节点
                zk.create(root, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            }
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void lock() {
        try {
            // 创建临时子节点
            String myNode = zk.create(root + "/" + lockName, data, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
            logger.debug(Thread.currentThread().getName() + myNode, "created");
            // 取出所有子节点
            List<String> subNodes = zk.getChildren(root, false);
            TreeSet<String> sortedNodes = new TreeSet<>();
            for (String node : subNodes) {
                sortedNodes.add(root + "/" + node);
            }
            String smallNode = sortedNodes.first();
            String preNode = sortedNodes.lower(myNode);
            if (myNode.equals(smallNode)) {
                // 如果是最小的节点，则表示取得锁
                logger.debug(Thread.currentThread().getName() + myNode, "get lock");
                this.nodeId.set(myNode);
                return;
            }
            CountDownLatch latch = new CountDownLatch(1);
            // 同时注册监听
            // 判断比自己小一个数的节点是否存在，如果不存在则无需等待锁，同时注册监听
            Stat stat = zk.exists(preNode, new LockWatcher(latch));
            if (stat != null) {
                logger.debug(Thread.currentThread().getName(), myNode, "waiting for" + root + "/" + preNode + "released lock");
                // 等待其他线程释放锁，session超时后其他线程的锁节点自动释放,增加100毫秒确保真的释放
                int blockTime = sessionTimeout + 100;
                while (blockTime >= 0) {
                    blockTime -= 100;
                    //防止一直消耗 CPU
                    Thread.sleep(100L);
                }
//                latch.wait();
                nodeId.set(myNode);
            }

        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public void unlock() throws Exception {
        logger.debug(Thread.currentThread().getName(), nodeId.get(), "unlock ");
        if (null != nodeId) {
            zk.delete(nodeId.get(), -1);
        }
        nodeId.remove();
    }

    class LockWatcher implements Watcher {
        private CountDownLatch latch;

        public LockWatcher(CountDownLatch latch) {
            this.latch = latch;
        }

        @Override
        public void process(WatchedEvent event) {
            // 建立连接
            if (event.getType() == Event.EventType.NodeDeleted) {
                latch.countDown();
            }
        }
    }
}
```
