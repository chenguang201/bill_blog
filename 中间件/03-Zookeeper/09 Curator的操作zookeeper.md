# 1 curator简介
curator是Netflix公司开源的一个zookeeper客户端，后捐献给apache，curator框架在zookeeper原生API接口上进行了包装，解决了很多zooKeeper客户端非常底层的细节开发。提供zooKeeper各种应用场景(比如：分布式锁服务、集群领导选举、共享计数器、缓存机制、分布式队列等)的抽象封装，实现了Fluent风格的API接口，是最好用，最流行的zookeeper的客户端。



**<font style="color:#E8323C;">原生zookeeperAPI的不足</font>**

+ 连接对象异步创建，需要开发人员自行编码等待
+ 连接没有自动重连超时机制
+ watcher一次注册生效一次
+ 不支持递归创建树形节点



**<font style="color:#E8323C;">curator特点</font>**

+ 解决session会话超时重连
+ watcher反复注册
+ 简化开发api
+ 遵循Fluent风格的API
+ 提供了分布式锁服务、共享计数器、缓存机制等机制

# 2 curator的操作
添加依赖：

```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  zookeeper</artifactId>
  <version>3.4.10</version>
</dependency>

<dependency>
  <groupId>org.apache.curator</groupId>
  curator-framework</artifactId>
  <version>2.6.0</version>
  <type>jar</type>
  <exclusions>
    <exclusion>
      <groupId>org.apache.zookeeper</groupId>
      zookeeper</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.apache.curator</groupId>
  curator-recipes</artifactId>
  <version>2.6.0</version>
  <type>jar</type>
</dependency>

</dependencies>
```



```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.curator.retry.RetryNTimes;
import org.apache.curator.retry.RetryOneTime;
import org.apache.curator.retry.RetryUntilElapsed;

public class CuratorConnection {
    public static void main(String[] args) {
        // session重连策略

        //3秒后重连一次，只重连1次
        RetryPolicy retryPolicy = new RetryOneTime(3000);


        //每3秒重连一次，重连3次
        RetryPolicy retryPolicy1 = new RetryNTimes(3, 3000);


        //每3秒重连一次，总等待时间超过10秒后停止重连
        RetryPolicy retryPolicy2 = new RetryUntilElapsed(10000, 3000);

        //baseSleepTimeMs * Math.max(1, random.nextInt(1 << (retryCount+1)))
        RetryPolicy retryPolicy4 = new ExponentialBackoffRetry(1000, 3);

        // 创建连接对象
        CuratorFramework client = CuratorFrameworkFactory.builder()
            // IP地址端口号
            .connectString("192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:21 83")
            // 会话超时时间
            .sessionTimeoutMs(5000)
            // 重连机制
            .retryPolicy(retryPolicy)
            // 命名空间
            .namespace("create")
            // 构建连接对象
            .build();
        // 打开连接
        client.start();
        System.out.println(client.isStarted());
        // 关闭连接
        client.close();
    }
}
```



## 2.1 新增节点
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import java.util.ArrayList;
import java.util.List;
public class CuratorCreate {
    String IP = "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
            .connectString(IP)
            .sessionTimeoutMs(5000)
            .retryPolicy(retryPolicy)
            .namespace("create")
            .build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void create1() throws Exception {
        // 新增节点
        client.create()
            // 节点的类型
            .withMode(CreateMode.PERSISTENT)
            // 节点的权限列表 world:anyone:cdrwa
            .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
            // arg1:节点的路径
            // arg2:节点的数据
            .forPath("/node1", "node1".getBytes());
        System.out.println("结束");
    }
    @Test
    public void create2() throws Exception {
        // 自定义权限列表
        // 权限列表
        List<ACL> list = new ArrayList<ACL>();
        // 授权模式和授权对象
        Id id = new Id("ip", "192.168.60.130");
        list.add(new ACL(ZooDefs.Perms.ALL, id));
        client.create().withMode(CreateMode.PERSISTENT).withACL(list).forPath("/node2", "node2".getBytes());
        System.out.println("结束");
    }
    @Test
    public void create3() throws Exception {
        // 递归创建节点树
        client.create()
            // 递归节点的创建
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
            .forPath("/node3/node31", "node31".getBytes());
        System.out.println("结束");
    }
    @Test
    public void create4() throws Exception {
        // 异步方式创建节点
        client.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
            // 异步回调接口
            .inBackground(new BackgroundCallback() {
                public void processResult(CuratorFramework
                                          curatorFramework, CuratorEvent curatorEvent) throws Exception {
                    // 节点的路径
                    System.out.println(curatorEvent.getPath());
                        // 时间类型
                        System.out.println(curatorEvent.getType());
                        }
                        })
                        .forPath("/node4","node4".getBytes());
                        Thread.sleep(5000);
                        System.out.println("结束");
                        }
                        }
```



## 2.2 更新节点
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
public class CuratorSet {
    String IP ="192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    
    CuratorFramework client;
    
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
            .connectString(IP)
            .sessionTimeoutMs(5000)
            .retryPolicy(retryPolicy)
            .namespace("set").build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void set1() throws Exception {
        // 更新节点
        client.setData()
            // arg1:节点的路径
            // arg2:节点的数据
            .forPath("/node1", "node11".getBytes());
        System.out.println("结束");
    }
    @Test
    public void set2() throws Exception {
        client.setData()
            // 指定版本号
            .withVersion(2)
            .forPath("/node1", "node1111".getBytes());
        System.out.println("结束");
    }
    @Test
    public void set3() throws Exception {
        // 异步方式修改节点数据
        client.setData()
            .withVersion(-1).inBackground(new BackgroundCallback() {
                public void processResult(CuratorFramework curatorFramework,
                                          CuratorEvent curatorEvent) throws Exception {
                    // 节点的路径
                    System.out.println(curatorEvent.getPath());
                    // 事件的类型
                    System.out.println(curatorEvent.getType());
                }
            }).forPath("/node1", "node1".getBytes());
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```



## 2.3 删除节点
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import java.util.ArrayList;
import java.util.List;
public class CuratorDelete {
    String IP =
    "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
            .connectString(IP)
            .sessionTimeoutMs(10000)
            .retryPolicy(retryPolicy)
            .namespace("delete").build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void delete1() throws Exception {
        // 删除节点
        client.delete()
            // 节点的路径
            .forPath("/node1");
        System.out.println("结束");
    }
    @Test
    public void delete2() throws Exception {
        client.delete()
            // 版本号
            .withVersion(0)
            .forPath("/node1");
        System.out.println("结束");
    }
    @Test
    public void delete3() throws Exception {
        //删除包含字节点的节点
        client.delete()
            .deletingChildrenIfNeeded()
            .withVersion(-1)
            .forPath("/node1");
        System.out.println("结束");
    }
    @Test
    public void delete4() throws Exception {
        // 异步方式删除节点
        client.delete()
            .deletingChildrenIfNeeded()
            .withVersion(-1)
            .inBackground(new BackgroundCallback() {
                public void processResult(CuratorFramework
                                          curatorFramework, CuratorEvent curatorEvent) throws Exception {
                    // 节点路径
                    System.out.println(curatorEvent.getPath());
                    // 事件类型
                    System.out.println(curatorEvent.getType());
                }
            })
            .forPath("/node1");
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```



## 2.4 查看节点
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
public class CuratorGet {
    String IP =
            "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
                .connectString(IP)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .namespace("get").build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void get1() throws Exception {
        // 读取节点数据
        byte [] bys=client.getData()
                // 节点的路径
                .forPath("/node1");
        System.out.println(new String(bys));
    }
    @Test
    public void get2() throws Exception {
        // 读取数据时读取节点的属性
        Stat stat=new Stat();
        byte [] bys=client.getData()
                // 读取属性
                .storingStatIn(stat)
                .forPath("/node1");
        System.out.println(new String(bys));
        System.out.println(stat.getVersion());
    }
    @Test
    public void get3() throws Exception {
        // 异步方式读取节点的数据
        client.getData()
                .inBackground(new BackgroundCallback() {
                    public void processResult(CuratorFramework
                                                      curatorFramework, CuratorEvent curatorEvent) throws Exception {
                        // 节点的路径
                        System.out.println(curatorEvent.getPath());
                        // 事件类型
                        System.out.println(curatorEvent.getType());
                        // 数据
                        System.out.println(new
                                String(curatorEvent.getData()));
                    }
                })
                .forPath("/node1");
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```



## 2.5 查看子节点


```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.List;

public class CuratorGetChild {
    String IP =
            "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;

    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
                .connectString(IP)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .build();
        client.start();
    }

    @After
    public void after() {
        client.close();
    }

    @Test
    public void getChild1() throws Exception {
        // 读取子节点数据
        List<String> list = client.getChildren()
                // 节点路径
                .forPath("/get");
        for (String str : list) {
            System.out.println(str);
        }
    }

    @Test
    public void getChild2() throws Exception {
        // 异步方式读取子节点数据
        client.getChildren()
                .inBackground(new BackgroundCallback() {
                    public void processResult(CuratorFramework
                                                      curatorFramework, CuratorEvent curatorEvent) throws Exception {
                        // 节点路径
                        System.out.println(curatorEvent.getPath());
                        // 事件类型
                        System.out.println(curatorEvent.getType());
                        // 读取子节点数据
                        List<String> list = curatorEvent.getChildren();
                        for (String str : list) {
                            System.out.println(str);
                        }
                    }
                })
                .forPath("/get");
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```



## 2.6 检查节点是否存在
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

public class CuratorExists {
    String IP =
    "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;

    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
            .connectString(IP)
            .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
            .namespace("get").build();
        client.start();
    }

    @After
    public void after() {
        client.close();
    }

    @Test
    public void exists1() throws Exception {
        // 判断节点是否存在
        Stat stat = client.checkExists()
            // 节点路径
            .forPath("/node2");
        System.out.println(stat.getVersion());
    }

    @Test
    public void exists2() throws Exception {
        // 异步方式判断节点是否存在
        client.checkExists()
            .inBackground(new BackgroundCallback() {
                public void processResult(CuratorFramework
                                          curatorFramework, CuratorEvent curatorEvent) throws Exception {
                    // 节点路径
                    System.out.println(curatorEvent.getPath());
                    // 事件类型
                    System.out.println(curatorEvent.getType());
                    System.out.println(curatorEvent.getStat().getVersion());
                }
            })
            .forPath("/node2");
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```



## 2.7 watcherAPI
curator提供了两种Watcher(Cache)来监听结点的变化。

+ Node Cache : 只是监听某一个特定的节点，监听节点的新增和修改。
+ PathChildren Cache : 监控一个ZNode的子节点. 当一个子节点增加， 更新，删除时， Path Cache会改变它的状态， 会包含最新的子节点， 子节点的数据和状态。

```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

public class CuratorWatcher {
    String IP =
            "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;

    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory
                .builder()
                .connectString(IP)
                .sessionTimeoutMs(10000)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
    }

    @After
    public void after() {
        client.close();
    }

    @Test
    public void watcher1() throws Exception {
        // 监视某个节点的数据变化
        // arg1:连接对象
        // arg2:监视的节点路径
        final NodeCache nodeCache = new NodeCache(client, "/watcher1");
        // 启动监视器对象
        nodeCache.start();
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            // 节点变化时回调的方法
            public void nodeChanged() throws Exception {
                System.out.println(nodeCache.getCurrentData().getPath());
                System.out.println(new
                        String(nodeCache.getCurrentData().getData()));
            }
        });
        Thread.sleep(100000);
        System.out.println("结束");
        //关闭监视器对象
        nodeCache.close();
    }

    @Test
    public void watcher2() throws Exception {
        // 监视子节点的变化
        // arg1:连接对象
        // arg2:监视的节点路径
        // arg3:事件中是否可以获取节点的数据
        PathChildrenCache pathChildrenCache = new
                PathChildrenCache(client, "/watcher1", true);
        // 启动监听
        pathChildrenCache.start();
        pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
                                                                  // 当子节点方法变化时回调的方法
             public void childEvent(CuratorFramework curatorFramework, PathChildrenCacheEvent pathChildrenCacheEvent) throws Exception {
                    // 节点的事件类型
                  System.out.println(pathChildrenCacheEvent.getType());
                    // 节点的路径
                  System.out.println(pathChildrenCacheEvent.getData().getPath());
                    // 节点数据
                  System.out.println(new String(pathChildrenCacheEvent.getData().getData()));
             }

         });
        Thread.sleep(100000);
        System.out.println("结束");
        // 关闭监听
        pathChildrenCache.close();
    }
}
```



## 2.8 事务
```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
public class CuratorTransaction {
    String IP =
            "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory.builder()
                .connectString(IP)
                .sessionTimeoutMs(10000).retryPolicy(retryPolicy)
                .namespace("create").build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void tra1() throws Exception {
        client.inTransaction().
                create().forPath("node1", "node1".getBytes()).
                and().
                create().forPath("node2", "node2".getBytes()).
                and().
                commit();
    }

}
```



## 2.9 分布式锁
+ InterProcessMutex：分布式可重入排它锁
+ InterProcessReadWriteLock：分布式读写锁

```java
package com.ali.zk.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.framework.recipes.locks.InterProcessLock;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import
        org.apache.curator.framework.recipes.locks.InterProcessReadWriteLock;
import
        org.apache.curator.framework.recipes.locks.InterProcessSemaphoreMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
public class CuratorLock {
    String IP =
            "192.168.60.130:2181,192.168.60.130:2182,192.168.60.130:2183";
    CuratorFramework client;
    @Before
    public void before() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
        client = CuratorFrameworkFactory
                .builder()
                .connectString(IP)
                .sessionTimeoutMs(10000)
                .retryPolicy(retryPolicy)
                .build();
        client.start();
    }
    @After
    public void after() {
        client.close();
    }
    @Test
    public void lock1() throws Exception {
        // 排他锁
        // arg1:连接对象
        // arg2:节点路径
        InterProcessLock interProcessLock = new InterProcessMutex(client,
                "/lock1");
        System.out.println("等待获取锁对象!");
        // 获取锁
        interProcessLock.acquire();
        for (int i = 1; i <= 10; i++) {
            Thread.sleep(3000);
            System.out.println(i);
        }
        // 释放锁
        interProcessLock.release();
        System.out.println("等待释放锁!");
    }
    @Test
    public void lock2() throws Exception {
        // 读写锁
        InterProcessReadWriteLock interProcessReadWriteLock=new
                InterProcessReadWriteLock(client, "/lock1");
        // 获取读锁对象
        InterProcessLock
                interProcessLock=interProcessReadWriteLock.readLock();
        System.out.println("等待获取锁对象!");
        // 获取锁
        interProcessLock.acquire();
        for (int i = 1; i <= 10; i++) {
            Thread.sleep(3000);
            System.out.println(i);
        }
        // 释放锁
        interProcessLock.release();
        System.out.println("等待释放锁!");
    }
    @Test
    public void lock3() throws Exception {
        InterProcessReadWriteLock interProcessReadWriteLock=new
                InterProcessReadWriteLock(client, "/lock1");
        // 获取写锁对象
        InterProcessLock
                interProcessLock=interProcessReadWriteLock.writeLock();
        System.out.println("等待获取锁对象!");
        // 获取锁
        interProcessLock.acquire();
        for (int i = 1; i <= 10; i++) {
            Thread.sleep(3000);
            System.out.println(i);
        }
        // 释放锁
        interProcessLock.release();
        System.out.println("等待释放锁!");
    }
}
```

