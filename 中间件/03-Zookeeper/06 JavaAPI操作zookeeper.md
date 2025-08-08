

znode是zooKeeper集合的核心组件，zookeeper API提供了一小组方法使用zookeeper集合来操纵znode的所有细节。

客户端应该遵循以步骤，与zookeeper服务器进行清晰和干净的交互。

+ 连接到zookeeper服务器。zookeeper服务器为客户端分配会话ID。
+ 定期向服务器发送心跳。否则，zookeeper服务器将过期会话ID，客户端需要重新连接。
+ 只要会话ID处于活动状态，就可以获取/设置znode。
+ 所有任务完成后，断开与zookeeper服务器的连接。如果客户端长时间不活动，则zookeeper服务器将自动断开客户端。



# 1 连接到ZooKeeper
```xml
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  zookeeper</artifactId>
  <version>3.4.10</version>
</dependency>
```

```java
package com.ali.zk;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.util.concurrent.CountDownLatch;

public class ZookeeperConnection {
    public static void main(String[] args) {
        try {
            // 计数器对象
            CountDownLatch countDownLatch = new CountDownLatch(1);
           // arg1:服务器的ip和端口
           // arg2:客户端与服务器之间的会话超时时间 以毫秒为单位的
           // arg3:监视器对象
            ZooKeeper zooKeeper = new ZooKeeper("192.168.60.130:2181",
                    5000, new Watcher() {
                @Override
                public void process(WatchedEvent event) {
                    if (event.getState() == Event.KeeperState.SyncConnected) {
                        System.out.println("连接创建成功!");
                        countDownLatch.countDown();
                    }
                }
            });
            // 主线程阻塞等待连接对象的创建成功
            countDownLatch.await();
            // 会话编号
            System.out.println(zooKeeper.getSessionId());
            zooKeeper.close();
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```



# 2 新增节点


```java
// 同步方式
create(String path, byte[] data, List<ACL> acl, CreateMode createMode)
    
// 异步方式
create(String path, byte[] data, List<ACL> acl, CreateMode createMode，AsyncCallback.StringCallback callBack,Object ctx)
// path - znode路径。例如，/node1 /node1/node11
// data - 要存储在指定znode路径中的数据
// acl - 要创建的节点的访问控制列表。zookeeper API提供了一个静态接口
// ZooDefs.Ids 来获取一些基本的acl列表。例如，ZooDefs.Ids.OPEN_ACL_UNSAFE返回打开znode的acl列表。
// createMode - 节点的类型,这是一个枚举。
// callBack-异步回调接口
// ctx-传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ZKCreate {
    String IP = "192.168.1.200:2181";
    ZooKeeper zooKeeper;

    @Before
    public void before() throws Exception {
        // 计数器对象
        CountDownLatch countDownLatch = new CountDownLatch(1);
        // arg1:服务器的ip和端口
        // arg2:客户端与服务器之间的会话超时时间 以毫秒为单位的
        // arg3:监视器对象
        zooKeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 主线程阻塞等待连接对象的创建成功
        countDownLatch.await();
    }

    @After
    public void after() throws Exception {
        zooKeeper.close();
    }

    @Test
    public void create1() throws Exception {
        // arg1:节点的路径
        // arg2:节点的数据
        // arg3:权限列表 world:anyone:cdrwa
        // arg4:节点类型 持久化节点
        zooKeeper.create("/create/node1", "node1".getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    }

    @Test
    public void create2() throws Exception {
        // Ids.READ_ACL_UNSAFE world:anyone:r
        zooKeeper.create("/create/node2", "node2".getBytes(),
                ZooDefs.Ids.READ_ACL_UNSAFE, CreateMode.PERSISTENT);
    }

    @Test
    public void create3() throws Exception {
        // world授权模式
        // 权限列表
        List<ACL> acls = new ArrayList<ACL>();
        // 授权模式和授权对象
        Id id = new Id("world", "anyone");
        // 权限设置
        acls.add(new ACL(ZooDefs.Perms.READ, id));
        acls.add(new ACL(ZooDefs.Perms.WRITE, id));
        zooKeeper.create("/create/node3", "node3".getBytes(), acls,
                CreateMode.PERSISTENT);
    }

    @Test
    public void create4() throws Exception {
        // ip授权模式
        // 权限列表
        List<ACL> acls = new ArrayList<ACL>();
        // 授权模式和授权对象
        Id id = new Id("ip", "192.168.60.130");
        // 权限设置
        acls.add(new ACL(ZooDefs.Perms.ALL, id));
        zooKeeper.create("/create/node4", "node4".getBytes(), acls,
                CreateMode.PERSISTENT);
    }

    @Test
    public void create5() throws Exception {
        // auth授权模式
        // 添加授权用户
        zooKeeper.addAuthInfo("digest", "itcast:123456".getBytes());
        zooKeeper.create("/create/node5", "node5".getBytes(),
                ZooDefs.Ids.CREATOR_ALL_ACL, CreateMode.PERSISTENT);
    }

    @Test
    public void create6() throws Exception {
        // auth授权模式
        // 添加授权用户
        zooKeeper.addAuthInfo("digest", "itcast:123456".getBytes());
        // 权限列表
        List<ACL> acls = new ArrayList<ACL>();
        // 授权模式和授权对象
        Id id = new Id("auth", "itcast");
        // 权限设置
        acls.add(new ACL(ZooDefs.Perms.READ, id));
        zooKeeper.create("/create/node6", "node6".getBytes(), acls,
                CreateMode.PERSISTENT);
    }

    @Test
    public void create7() throws Exception {
        // digest授权模式
        // 权限列表
        List<ACL> acls = new ArrayList<ACL>();
        // 授权模式和授权对象
        Id id = new Id("digest", "itheima:qlzQzCLKhBROghkooLvb+Mlwv4A=");
        // 权限设置
        acls.add(new ACL(ZooDefs.Perms.ALL, id));
        zooKeeper.create("/create/node7", "node7".getBytes(), acls,
                CreateMode.PERSISTENT);
    }

    @Test
    public void create8() throws Exception {
        // 持久化顺序节点
        // Ids.OPEN_ACL_UNSAFE world:anyone:cdrwa
        String result = zooKeeper.create("/create/node8",
                "node8".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.PERSISTENT_SEQUENTIAL);
        System.out.println(result);
    }

    @Test
    public void create9() throws Exception {
        // 临时节点
        // Ids.OPEN_ACL_UNSAFE world:anyone:cdrwa
        String result = zooKeeper.create("/create/node9",
                "node9".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL);
        System.out.println(result);
    }

    @Test
    public void create10() throws Exception {
            // 临时顺序节点
            // Ids.OPEN_ACL_UNSAFE world:anyone:cdrwa
        String result = zooKeeper.create("/create/node10",
                "node10".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL);
        System.out.println(result);
    }

    @Test
    public void create11() throws Exception {
        // 异步方式创建节点
        zooKeeper.create("/create/node11", "node11".getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT, new
                        AsyncCallback.StringCallback() {
                            @Override
                            public void processResult(int rc, String path, Object ctx,
                                                      String name) {
                                // 0 代表创建成功
                                System.out.println(rc);
                                // 节点的路径
                                System.out.println(path);
                                // 节点的路径
                                System.out.println(name);
                                // 上下文参数
                                System.out.println(ctx);
                            }
                        }, "I am context");
        Thread.sleep(10000);
        System.out.println("结束");
    }
}
```



# 3 更新节点


```java
// 同步方式
setData(String path, byte[] data, int version)
    
// 异步方式
setData(String path, byte[] data, int version，AsyncCallback.StatCallback callBack， Object ctx)
// path- znode路径
// data - 要存储在指定znode路径中的数据。
// version- znode的当前版本。每当数据更改时，ZooKeeper会更新znode的版本号。
// callBack-异步回调接口
// ctx-传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.concurrent.CountDownLatch;

public class ZKSet {
    String IP = "192.168.60.130:2181";
    ZooKeeper zookeeper;

    @Before
    public void before() throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(1);
            // arg1:zookeeper服务器的ip地址和端口号
            // arg2:连接的超时时间 以毫秒为单位
            // arg3:监听器对象
        zookeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 使主线程阻塞等待
        countDownLatch.await();
    }

    @After
    public void after() throws Exception {
        zookeeper.close();
    }

    @Test

    public void set1() throws Exception {
        // arg1:节点的路径
        // arg2:节点修改的数据
        // arg3:版本号 -1代表版本号不作为修改条件
        Stat stat = zookeeper.setData("/set/node1", "node13".getBytes(), 2);
        // 节点的版本号
        System.out.println(stat.getVersion());
        // 节点的创建时间
        System.out.println(stat.getCtime());
    }

    @Test
    public void set2() throws Exception {
        // 异步方式修改节点
        zookeeper.setData("/set/node2", "node21".getBytes(), -1, new
                AsyncCallback.StatCallback() {
                    @Override
                    public void processResult(int rc, String path, Object ctx,
                                              Stat stat) {
                        // 0 代表修改成功
                        System.out.println(rc);
                        // 修改节点的路径
                        System.out.println(path);
                        // 上线文的参数对象
                        System.out.println(ctx);
                        // 的属性信息
                        System.out.println(stat.getVersion());
                    }
                }, "I am Context");
        Thread.sleep(50000);
        System.out.println("结束");
    }
}
```



# 4 删除节点
```java
// 同步方式
delete(String path, int version)
    
// 异步方式
delete(String path, int version, AsyncCallback.VoidCallback callBack,Object ctx)
// path - znode路径。
// version - znode的当前版本
// callBack-异步回调接口
// ctx-传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import java.util.concurrent.CountDownLatch;
public class ZKDelete {
    String IP = "192.168.60.130:2181";
    ZooKeeper zooKeeper;
    @Before
    public void before() throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(1);
            // arg1:zookeeper服务器的ip地址和端口号
            // arg2:连接的超时时间 以毫秒为单位
            // arg3:监听器对象
        zooKeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected)
                {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 使主线程阻塞等待
        countDownLatch.await();
    }
    @After
    public void after() throws Exception {
        zooKeeper.close();
    }
    @Test
    public void delete1() throws Exception {
        // arg1:删除节点的节点路径
        // arg2:数据版本信息 -1代表删除节点时不考虑版本信息
        zooKeeper.delete("/delete/node1",-1);
    }
    @Test
    public void delete2() throws Exception {
        // 异步使用方式
        zooKeeper.delete("/delete/node2", -1, new
                AsyncCallback.VoidCallback() {
                    @Override
                    public void processResult(int rc, String path, Object ctx) {
                        // 0代表删除成功
                        System.out.println(rc);
                        // 节点的路径
                        System.out.println(path);
                        // 上下文参数对象
                        System.out.println(ctx);
                    }
                },"I am Context");
        Thread.sleep(10000);
        System.out.println("结束");
    }
}
```



# 5 查看节点
```java
// 同步方式
getData(String path, boolean b, Stat stat)
    
// 异步方式
getData(String path, boolean b，AsyncCallback.DataCallback callBack，Object ctx)
// path -znode路径。
// b -是否使用连接对象中注册的监视器。
// stat -返回znode的元数据。
// callBack -异步回调接口
// ctx -传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.concurrent.CountDownLatch;

public class ZKGet {
    String IP = "192.168.1.200:2181";
    ZooKeeper zooKeeper;

    @Before
    public void before() throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(1);
            // arg1:zookeeper服务器的ip地址和端口号
            // arg2:连接的超时时间 以毫秒为单位
            // arg3:监听器对象
        zooKeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 使主线程阻塞等待
        countDownLatch.await();
    }

    @After
    public void after() throws Exception {
        zooKeeper.close();
    }
    @Test

    public void get1() throws Exception {
        // arg1:节点的路径
        // arg3:读取节点属性的对象
        Stat stat = new Stat();
        byte[] bys = zooKeeper.getData("/get/node1", false, stat);
        // 打印数据
        System.out.println(new String(bys));
        // 版本信息
        System.out.println(stat.getVersion());
    }

    @Test
    public void get2() throws Exception {
            //异步方式
        zooKeeper.getData("/get/node1", false, new
                AsyncCallback.DataCallback() {
                    @Override
                    public void processResult(int rc, String path, Object ctx,
                                              byte[] data, Stat stat) {
                        // 0代表读取成功
                        System.out.println(rc);
                        // 节点的路径
                        System.out.println(path);
                        // 上下文参数对象
                        System.out.println(ctx);
                        // 数据
                        System.out.println(new String(data));
                        // 属性对象
                        System.out.println(stat.getVersion());
                    }
                }, "I am Context");
        Thread.sleep(10000);
        System.out.println("结束");
    }
}
```



# 6 查看子节点


```java
// 同步方式
getChildren(String path, boolean b)
    
// 异步方式
getChildren(String path, boolean b,AsyncCallback.ChildrenCallback callBack,Object ctx)
// path -Znode路径。
// b -是否使用连接对象中注册的监视器。
// callBack -异步回调接口。
// ctx -传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ZKGetChid {
    String IP = "192.168.60.130:2181";
    ZooKeeper zooKeeper;

    @Before
    public void before() throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(1);
            // arg1:zookeeper服务器的ip地址和端口号
            // arg2:连接的超时时间 以毫秒为单位
            // arg3:监听器对象
        zooKeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 使主线程阻塞等待
        countDownLatch.await();
    }

    @After
    public void after() throws Exception {
        zooKeeper.close();
    }

    @Test
    public void get1() throws Exception {
        // arg1:节点的路径
        List<String> list = zooKeeper.getChildren("/get", false);
        for (String str : list) {
            System.out.println(str);
        }
    }

    @Test
    public void get2() throws Exception {
            // 异步用法
        zooKeeper.getChildren("/get", false, new
                AsyncCallback.ChildrenCallback() {
                    @Override
                    public void processResult(int rc, String path, Object ctx,
                                              List<String> children) {
                        // 0代表读取成功
                        System.out.println(rc);
                        // 节点的路径
                        System.out.println(path);
                        // 上下文参数对象
                        System.out.println(ctx);
                        // 子节点信息
                        for (String str : children) {
                            System.out.println(str);
                        }
                    }
                }, "I am Context");
        Thread.sleep(10000);
        System.out.println("结束");
    }
}
```



# 7 检查节点是否存在


```java
// 同步方法
exists(String path, boolean b)
    
// 异步方法
exists(String path, boolean b，AsyncCallback.StatCallback callBack,Object ctx)
// path -znode路径。
// b -是否使用连接对象中注册的监视器。
// callBack -异步回调接口。
// ctx -传递上下文参数
```



```java
package com.ali.zk;

import org.apache.zookeeper.AsyncCallback;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.Stat;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.concurrent.CountDownLatch;

public class ZKExists {
    String IP = "192.168.60.130:2181";
    ZooKeeper zookeeper;

    @Before
    public void before() throws Exception {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        // arg1:zookeeper服务器的ip地址和端口号
        // arg2:连接的超时时间 以毫秒为单位
        // arg3:监听器对象
        zookeeper = new ZooKeeper(IP, 5000, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                if (event.getState() == Event.KeeperState.SyncConnected) {
                    System.out.println("连接创建成功!");
                    countDownLatch.countDown();
                }
            }
        });
        // 使主线程阻塞等待
        countDownLatch.await();
    }

    @After
    public void after() throws Exception {
        zookeeper.close();
    }

    @Test
    public void exists1() throws Exception {
        // arg1:节点的路径
        Stat stat = zookeeper.exists("/exists1", false);
        System.out.println(stat.getVersion());
    }

    @Test
    public void exists2() throws Exception {
        // 异步使用方式
        zookeeper.exists("/exists1", false, new
                AsyncCallback.StatCallback() {
                    @Override
                    public void processResult(int rc, String path, Object ctx,
                                              Stat stat) {
                        // 0 判断成功
                        System.out.println(rc);
                        // 路径
                        System.out.println(path);
                        // 上下文参数
                        System.out.println(ctx);
                        // null 节点不存在
                        System.out.println(stat.getVersion());
                    }
                }, "I am Context");
        Thread.sleep(5000);
        System.out.println("结束");
    }
}
```

