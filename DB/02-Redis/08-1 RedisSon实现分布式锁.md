**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Redis入门到实战教程，深度透析redis底层原理+redis分布式锁+企业解决方案</font>**](https://www.bilibili.com/video/BV1cr4y1671t/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

## 1 Redission功能介绍
基于SETNX实现的分布式锁存在下面的问题：

**重入问题**：重入问题是指 获得锁的线程可以再次进入到相同的锁的代码块中，可重入锁的意义在于防止死锁，比如HashTable这样的代码中，他的方法都是使用synchronized修饰的，假如他在一个方法内，调用另一个方法，那么此时如果是不可重入的，不就死锁了吗？所以可重入锁他的主要意义是防止死锁，我们的synchronized和Lock锁都是可重入的。

**不可重试**：是指目前的分布式只能尝试一次，我们认为合理的情况是：当线程在获得锁失败后，他应该能再次尝试获得锁。

**超时释放：**我们在加锁时增加了过期时间，这样的我们可以防止死锁，但是如果卡顿的时间超长，虽然我们采用了lua表达式防止删锁的时候，误删别人的锁，但是毕竟没有锁住，有安全隐患

**主从一致性：** 如果Redis提供了主从集群，当我们向集群写数据时，主机需要异步的将数据同步给从机，而万一在同步过去之前，主机宕机了，就会出现死锁问题。

![](images/51.png)



那么什么是Redission呢

Redisson是一个在Redis的基础上实现的Java驻内存数据网格（In-Memory Data Grid）。它不仅提供了一系列的分布式的Java常用对象，还提供了许多分布式服务，其中就包含了各种分布式锁的实现。

Redission提供了分布式锁的多种多样的功能

1. <font style="color:rgb(76, 73, 72);">可重入锁(Reentrant Lock)</font>
2. <font style="color:rgb(76, 73, 72);">公平锁(Fair Lock)</font>
3. <font style="color:rgb(76, 73, 72);">联锁(MultiLock)</font>
4. <font style="color:rgb(76, 73, 72);">红锁(RedLock)</font>
5. <font style="color:rgb(76, 73, 72);">读写锁(ReadWriteLock)</font>
6. <font style="color:rgb(76, 73, 72);">信号量(Semaphore)</font>
7. <font style="color:rgb(76, 73, 72);">可过期性信号量(PermitExpirableSemaphore)</font>
8. <font style="color:rgb(76, 73, 72);">闭锁(CountDownLatch)</font>

## 2 Redission快速入门
引入依赖：

```xml
<dependency>
  <groupId>org.redisson</groupId>
  redisson</artifactId>
  <version>3.13.6</version>
</dependency>
```



配置Redisson客户端：

```java
@Configuration
public class RedissonConfig {

    @Bean
    public RedissonClient redissonClient(){
        // 配置
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.150.101:6379")
            .setPassword("123321");
        // 创建RedissonClient对象
        return Redisson.create(config);
    }
}
```



如何使用Redission的分布式锁

```java
@Resource
private RedissionClient redissonClient;

@Test
void testRedisson() throws Exception{
    //获取锁(可重入)，指定锁的名称
    RLock lock = redissonClient.getLock("anyLock");
    //尝试获取锁，参数分别是：获取锁的最大等待时间(期间会重试)，锁自动释放时间，时间单位
    boolean isLock = lock.tryLock(1,10,TimeUnit.SECONDS);
    //判断获取锁成功
    if(isLock){
        try{
            System.out.println("执行业务");          
        }finally{
            //释放锁
            lock.unlock();
        }
    }   
}
```



在 VoucherOrderServiceImpl注入RedissonClient

```java
@Resource
private RedissonClient redissonClient;

@Override
public Result seckillVoucher(Long voucherId) {
        // 1.查询优惠券
        SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
        // 2.判断秒杀是否开始
        if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
            // 尚未开始
            return Result.fail("秒杀尚未开始！");
        }
        // 3.判断秒杀是否已经结束
        if (voucher.getEndTime().isBefore(LocalDateTime.now())) {
            // 尚未开始
            return Result.fail("秒杀已经结束！");
        }
        // 4.判断库存是否充足
        if (voucher.getStock() < 1) {
            // 库存不足
            return Result.fail("库存不足！");
        }
        Long userId = UserHolder.getUser().getId();
        //创建锁对象 这个代码不用了，因为我们现在要使用分布式锁
        //SimpleRedisLock lock = new SimpleRedisLock("order:" + userId, stringRedisTemplate);
        RLock lock = redissonClient.getLock("lock:order:" + userId);
        //获取锁对象
        boolean isLock = lock.tryLock();
       
		//加锁失败
        if (!isLock) {
            return Result.fail("不允许重复下单");
        }
        try {
            //获取代理对象(事务)
            IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);
        } finally {
            //释放锁
            lock.unlock();
        }
 }
```

## 3 redission可重入锁原理
在Lock锁中，他是借助于底层的一个voaltile的一个state变量来记录重入的状态的，

+ 比如当前没有人持有这把锁，那么state=0，
+ 假如有人持有这把锁，那么state=1，
    - 如果持有这把锁的人再次持有这把锁，那么state就会+1 ，
+ 如果是对于synchronized而言，他在c语言代码中会有一个count，原理和state类似，也是重入一次就加一，释放一次就-1 ，直到减少成0时，表示当前这把锁没有被人持有。

在redission中，我们的也支持支持可重锁

+ <font style="color:rgb(76, 73, 72);">在分布式锁中，它采用hash结构来存储锁，其中外层key表示这把锁是否存在，内层key则记录当前这把锁被哪个线程持有</font>

<font style="color:rgb(76, 73, 72);">method1在方法内部调用method2，method1和method2出于同一个线程，那么method1已经拿到一把锁了，想进入method2中拿另外一把锁，必然是拿不到的，于是就出现了死锁</font>

```java
@Resource
private RedissonClient redissonClient;

private RLock lock;

@BeforeEach
void setUp() {
    lock = redissonClient.getLock("lock");
}

@Test
void method1() {
    boolean success = lock.tryLock();
    if (!success) {
        log.error("获取锁失败，1");
        return;
    }
    try {
        log.info("获取锁成功");
        method2();
    } finally {
        log.info("释放锁，1");
        lock.unlock();
    }
}

void method2() {
    RLock lock = redissonClient.getLock("lock");
    boolean success = lock.tryLock();
    if (!success) {
        log.error("获取锁失败，2");
        return;
    }
    try {
        log.info("获取锁成功，2");
    } finally {
        log.info("释放锁，2");
        lock.unlock();
    }
}
```

<font style="color:rgb(76, 73, 72);">所以我们需要额外判断，method1和method2是否处于同一线程，如果是同一个线程，则可以拿到锁，但是state会</font><font style="color:rgb(244, 116, 102);">+1</font><font style="color:rgb(76, 73, 72);">，之后执行method2中的方法，释放锁，释放锁的时候也只是将state进行</font><font style="color:rgb(244, 116, 102);">-1</font><font style="color:rgb(76, 73, 72);">，只有减至0，才会真正释放锁</font>

<font style="color:rgb(76, 73, 72);">由于我们需要额外存储一个state，所以用字符串型</font><font style="color:rgb(244, 116, 102);">SET NX EX</font><font style="color:rgb(76, 73, 72);">是不行的，需要用到</font><font style="color:rgb(244, 116, 102);">Hash</font><font style="color:rgb(76, 73, 72);">结构，但是</font><font style="color:rgb(244, 116, 102);">Hash</font><font style="color:rgb(76, 73, 72);">结构又没有</font><font style="color:rgb(244, 116, 102);">NX</font><font style="color:rgb(76, 73, 72);">这种方法，所以我们需要将原有的逻辑拆开，进行手动判断</font>

![](images/52.png)

<font style="color:rgb(76, 73, 72);">为了保证原子性，所以流程图中的业务逻辑也是需要我们用Lua来实现的</font>

<font style="color:rgb(76, 73, 72);">获取锁的逻辑</font>

```lua
local key = KEYS[1]; -- 锁的key
local threadId = ARGV[1]; -- 线程唯一标识
local releaseTime = ARGV[2]; -- 锁的自动释放时间
-- 锁不存在
if (redis.call('exists', key) == 0) then
    -- 获取锁并添加线程标识，state设为1
    redis.call('hset', key, threadId, '1');
    -- 设置锁有效期
    redis.call('expire', key, releaseTime);
    return 1; -- 返回结果
end;
-- 锁存在，判断threadId是否为自己
if (redis.call('hexists', key, threadId) == 1) then
    -- 锁存在，重入次数 +1，这里用的是hash结构的incrby增长
    redis.call('hincrby', key, thread, 1);
    -- 设置锁的有效期
    redis.call('expire', key, releaseTime);
    return 1; -- 返回结果
end;
return 0; -- 代码走到这里，说明获取锁的不是自己，获取锁失败
```

<font style="color:rgb(76, 73, 72);">释放锁的逻辑</font>

```lua
local key = KEYS[1];
local threadId = ARGV[1];
local releaseTime = ARGV[2];
-- 如果锁不是自己的
if (redis.call('HEXISTS', key, threadId) == 0) then
    return nil; -- 直接返回
end;
-- 锁是自己的，锁计数-1，还是用hincrby，不过自增长的值为-1
local count = redis.call('hincrby', key, threadId, -1);
-- 判断重入次数为多少
if (count > 0) then
    -- 大于0，重置有效期
    redis.call('expire', key, releaseTime);
    return nil;
else
    -- 否则直接释放锁
    redis.call('del', key);
    return nil;
end;
```

<font style="color:rgb(76, 73, 72);">获取锁源码</font>：<font style="color:rgb(76, 73, 72);">查看源码，跟我们的实现方式几乎一致</font>

```java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    this.internalLockLeaseTime = unit.toMillis(leaseTime);
    return this.evalWriteAsync(this.getName(), LongCodec.INSTANCE, command, "if (redis.call('exists', KEYS[1]) == 0) then redis.call('hincrby', KEYS[1], ARGV[2], 1); redis.call('pexpire', KEYS[1], ARGV[1]); return nil; end; if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then redis.call('hincrby', KEYS[1], ARGV[2], 1); redis.call('pexpire', KEYS[1], ARGV[1]); return nil; end; return redis.call('pttl', KEYS[1]);", Collections.singletonList(this.getName()), this.internalLockLeaseTime, this.getLockName(threadId));
}
```

<font style="color:rgb(76, 73, 72);">释放锁源码</font>

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return this.evalWriteAsync(this.getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN, "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then return nil;end; local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); if (counter > 0) then redis.call('pexpire', KEYS[1], ARGV[2]); return 0; else redis.call('del', KEYS[1]); redis.call('publish', KEYS[2], ARGV[1]); return 1; end; return nil;", Arrays.asList(this.getName(), this.getChannelName()), LockPubSub.UNLOCK_MESSAGE, this.internalLockLeaseTime, this.getLockName(threadId));
}
```

## 4 redission锁重试和WatchDog机制
![](images/53.png)

**源码流程**

```java
package com.hmdp;

import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.boot.test.context.SpringBootTest;

import javax.annotation.Resource;
import java.util.concurrent.TimeUnit;

@Slf4j
@SpringBootTest
class RedissonTest {

    @Resource
    private RedissonClient redissonClient;

    private RLock lock;

    @BeforeEach
    void setUp() {
        lock = redissonClient.getLock("order");
    }

    @Test
    void method1() throws InterruptedException {
        // 尝试获取锁
        boolean isLock = lock.tryLock(1L, TimeUnit.SECONDS); //从这里开始
        if (!isLock) {
            log.error("获取锁失败 .... 1");
            return;
        }
        try {
            log.info("获取锁成功 .... 1");
            method2();
            log.info("开始执行业务 ... 1");
        } finally {
            log.warn("准备释放锁 .... 1");
            lock.unlock();
        }
    }
    void method2() {
        // 尝试获取锁
        boolean isLock = lock.tryLock();
        if (!isLock) {
            log.error("获取锁失败 .... 2");
            return;
        }
        try {
            log.info("获取锁成功 .... 2");
            log.info("开始执行业务 ... 2");
        } finally {
            log.warn("准备释放锁 .... 2");
            lock.unlock();
        }
    }
}

```

```java
public boolean tryLock(long waitTime, TimeUnit unit) throws InterruptedException {
        return this.tryLock(waitTime, -1L, unit);
}
```

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
        long time = unit.toMillis(waitTime);
        long current = System.currentTimeMillis();
        long threadId = Thread.currentThread().getId();
    	//1. 先看这个获取锁的流程
        Long ttl = this.tryAcquire(waitTime, leaseTime, unit, threadId);
        //判断ttl是否为null
        if (ttl == null) {
            return true;
        } else {
            //计算当前时间与获取锁时间的差值，让等待时间减去这个值
            time -= System.currentTimeMillis() - current;
            //如果消耗时间太长了，直接返回false，获取锁失败
            if (time <= 0L) {
                this.acquireFailed(waitTime, unit, threadId);
                return false;
            } else {
                //等待时间还有剩余，再次获取当前时间
                current = System.currentTimeMillis();
                //订阅别人释放锁的信号
                RFuture<RedissonLockEntry> subscribeFuture = this.subscribe(threadId);
                //在剩余时间内，等待这个信号
                if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {
                    if (!subscribeFuture.cancel(false)) {
                        subscribeFuture.onComplete((res, e) -> {
                            if (e == null) {
                                //取消订阅
                                this.unsubscribe(subscribeFuture, threadId);
                            }

                        });
                    }
                    //剩余时间内没等到，返回false
                    this.acquireFailed(waitTime, unit, threadId);
                    return false;
                } else {
                    try {
                        //如果剩余时间内等到了别人释放锁的信号，再次计算当前剩余最大等待时间
                        time -= System.currentTimeMillis() - current;
                        if (time <= 0L) {
                            //如果剩余时间为负数，则直接返回false
                            this.acquireFailed(waitTime, unit, threadId);
                            boolean var20 = false;
                            return var20;
                        } else {
                            boolean var16;
                            do {
                                //如果剩余时间等到了，dowhile循环重试获取锁
                                long currentTime = System.currentTimeMillis();
                                ttl = this.tryAcquire(waitTime, leaseTime, unit, threadId);
                                if (ttl == null) {
                                    var16 = true;
                                    return var16;
                                }

                                time -= System.currentTimeMillis() - currentTime;
                                if (time <= 0L) {
                                    this.acquireFailed(waitTime, unit, threadId);
                                    var16 = false;
                                    return var16;
                                }

                                currentTime = System.currentTimeMillis();
                                if (ttl >= 0L && ttl < time) {
                                    ((RedissonLockEntry)subscribeFuture.getNow()).getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                                } else {
                                    ((RedissonLockEntry)subscribeFuture.getNow()).getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
                                }

                                time -= System.currentTimeMillis() - currentTime;
                            } while(time > 0L);

                            this.acquireFailed(waitTime, unit, threadId);
                            var16 = false;
                            return var16;
                        }
                    } finally {
                        this.unsubscribe(subscribeFuture, threadId);
                    }
                }
            }
        }
    }
```

```java
private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
    return (Long)this.get(this.tryAcquireAsync(waitTime, leaseTime, unit, threadId));
}
```



```java
private <T> RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        if (leaseTime != -1L) {
            return this.tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_LONG);
        } else {
            //这个是获取锁的流程
            RFuture<Long> ttlRemainingFuture = this.tryLockInnerAsync(waitTime, this.commandExecutor.getConnectionManager().getCfg().getLockWatchdogTimeout(), TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);
            ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
                if (e == null) {
                    if (ttlRemaining == null) {
                        //这个是更新过期时间的调度任务，牵扯到WatchDog，可以看看
                        this.scheduleExpirationRenewal(threadId);
                    }

                }
            });
            return ttlRemainingFuture;
        }
    }
```

```java
 <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        this.internalLockLeaseTime = unit.toMillis(leaseTime);
        return this.evalWriteAsync(this.getName(), LongCodec.INSTANCE, command, "if (redis.call('exists', KEYS[1]) == 0) then redis.call('hincrby', KEYS[1], ARGV[2], 1); redis.call('pexpire', KEYS[1], ARGV[1]); return nil; end; if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then redis.call('hincrby', KEYS[1], ARGV[2], 1); redis.call('pexpire', KEYS[1], ARGV[1]); return nil; end; return redis.call('pttl', KEYS[1]);", Collections.singletonList(this.getName()), this.internalLockLeaseTime, this.getLockName(threadId));
    }
```

接下里`scheduleExpirationRenewal`

```java
private void scheduleExpirationRenewal(long threadId) {
        RedissonLock.ExpirationEntry entry = new RedissonLock.ExpirationEntry();
        RedissonLock.ExpirationEntry oldEntry = (RedissonLock.ExpirationEntry)EXPIRATION_RENEWAL_MAP.putIfAbsent(this.getEntryName(), entry);
        if (oldEntry != null) {
            oldEntry.addThreadId(threadId);
        } else {
            entry.addThreadId(threadId);

            //看看这行， 如果是第一次进入，则跟新有效期
            this.renewExpiration();
        }

    }
```

```java
    private void renewExpiration() {
        RedissonLock.ExpirationEntry ee = (RedissonLock.ExpirationEntry)EXPIRATION_RENEWAL_MAP.get(this.getEntryName());
        if (ee != null) {
            Timeout task = this.commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
                public void run(Timeout timeout) throws Exception {
                    RedissonLock.ExpirationEntry ent = (RedissonLock.ExpirationEntry)RedissonLock.EXPIRATION_RENEWAL_MAP.get(RedissonLock.this.getEntryName());
                    if (ent != null) {
                        Long threadId = ent.getFirstThreadId();
                        if (threadId != null) {
                            //看看这行
                            RFuture<Boolean> future = RedissonLock.this.renewExpirationAsync(threadId);
                            future.onComplete((res, e) -> {
                                if (e != null) {
                                    RedissonLock.log.error("Can't update lock " + RedissonLock.this.getName() + " expiration", e);
                                } else {
                                    if (res) {
                                        //然后调用自己，递归重置有效期
                                        RedissonLock.this.renewExpiration();
                                    }

                                }
                            });
                        }
                    }
                }
                //这是一个定时任务，目前是延期过期时间，internalLockLeaseTime是之前WatchDog默认有效期30秒，那这里就是 30 / 3 = 10秒之后，才会执行
            }, this.internalLockLeaseTime / 3L, TimeUnit.MILLISECONDS);
            ee.setTimeout(task);
        }
    }
```

```java
 protected RFuture<Boolean> renewExpirationAsync(long threadId) //重点看lua脚本，先判断锁是不是自己的，然后更新有效时间
        return this.evalWriteAsync(this.getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN, "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then redis.call('pexpire', KEYS[1], ARGV[1]); return 1; end; return 0;", Collections.singletonList(this.getName()), this.internalLockLeaseTime, this.getLockName(threadId));
    }
```

<font style="color:rgb(76, 73, 72);">那么之前的重置有效期的行为该怎么终止呢？当然是释放锁的时候会终止</font>

```java
public RFuture<Void> unlockAsync(long threadId) {
        RPromise<Void> result = new RedissonPromise();
		//释放锁
        RFuture<Boolean> future = this.unlockInnerAsync(threadId);
        future.onComplete((opStatus, e) -> {
            //看看这行
            this.cancelExpirationRenewal(threadId);
            if (e != null) {
                result.tryFailure(e);
            } else if (opStatus == null) {
                IllegalMonitorStateException cause = new IllegalMonitorStateException("attempt to unlock lock, not locked by current thread by node id: " + this.id + " thread-id: " + threadId);
                result.tryFailure(cause);
            } else {
                result.trySuccess((Object)null);
            }
        });
        return result;
    }
```

```java
 void cancelExpirationRenewal(Long threadId) {
        RedissonLock.ExpirationEntry task = (RedissonLock.ExpirationEntry)EXPIRATION_RENEWAL_MAP.get(this.getEntryName());
        if (task != null) {
            if (threadId != null) {
                task.removeThreadId(threadId);
            }

            if (threadId == null || task.hasNoThreads()) 
            	//获取之前的定时任务
                Timeout timeout = task.getTimeout();
                if (timeout != null) {
                    timeout.cancel();
                }
            	
                EXPIRATION_RENEWAL_MAP.remove(this.getEntryName());
            }

        }
    }
```

<font style="color:rgb(76, 73, 72);">Redisson分布式锁的原理：</font>

+ <font style="color:rgb(76, 73, 72);">可重入：利用Hash结构，记录线程标识与重入次数</font>
+ <font style="color:rgb(76, 73, 72);">可重试：利用信号量和PubSub功能实现等待，唤醒，获取锁失败的重试机制</font>
+ <font style="color:rgb(76, 73, 72);">超时续约：利用WatchDog，每隔一段时间（releaseTime / 3），重置超时时间</font>

## 5 redission锁的MutiLock原理
为了提高redis的可用性，我们会搭建集群或者主从，现在以主从为例

此时我们去写命令，写在主机上， 主机会将数据同步给从机，但是假设在主机还没有来得及把数据写入到从机去的时候，此时主机宕机，哨兵会发现主机宕机，并且选举一个slave变成master，而此时新的master中实际上并没有锁信息，此时锁信息就已经丢掉了。

![](images/54.png)

为了解决这个问题，redission提出来了MutiLock锁，使用这把锁咱们就不使用主从了，每个节点的地位都是一样的， 这把锁加锁的逻辑需要写入到每一个主丛节点上，只有所有的服务器都写入成功，此时才是加锁成功，假设现在某个节点挂了，那么他去获得锁的时候，只要有一个节点拿不到，都不能算是加锁成功，就保证了加锁的可靠性。

![](images/55.png)

那么MutiLock 加锁原理是什么呢？笔者画了一幅图来说明

当我们去设置了多个锁时，redission会将多个锁添加到一个集合中，然后用while循环去不停去尝试拿锁，但是会有一个总共的加锁时间，这个时间是用需要加锁的个数 * 1500ms ，假设有3个锁，那么时间就是4500ms，假设在这4500ms内，所有的锁都加锁成功， 那么此时才算是加锁成功，如果在4500ms有线程加锁失败，则会再次去进行重试.

![](images/56.png)

```java
@Configuration
public class RedissonConfig {
    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://192.168.137.130:6379")
                .setPassword("root");
        return Redisson.create(config);
    }

    @Bean
    public RedissonClient redissonClient2() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://92.168.137.131:6379")
                .setPassword("root");
        return Redisson.create(config);
    }

    @Bean
    public RedissonClient redissonClient3() {
        Config config = new Config();
        config.useSingleServer().setAddress("redis://92.168.137.132:6379")
                .setPassword("root");
        return Redisson.create(config);
    }
}
```

```java
@Resource
private RedissonClient redissonClient;
@Resource
private RedissonClient redissonClient2;
@Resource
private RedissonClient redissonClient3;

private RLock lock;

@BeforeEach
void setUp() {
    RLock lock1 = redissonClient.getLock("lock");
    RLock lock2 = redissonClient2.getLock("lock");
    RLock lock3 = redissonClient3.getLock("lock");
    lock = redissonClient.getMultiLock(lock1, lock2, lock3);
}

@Test
void method1() {
    boolean success = lock.tryLock();
    redissonClient.getMultiLock();
    if (!success) {
        log.error("获取锁失败，1");
        return;
    }
    try {
        log.info("获取锁成功");
        method2();
    } finally {
        log.info("释放锁，1");
        lock.unlock();
    }
}

void method2() {
    RLock lock = redissonClient.getLock("lock");
    boolean success = lock.tryLock();
    if (!success) {
        log.error("获取锁失败，2");
        return;
    }
    try {
        log.info("获取锁成功，2");
    } finally {
        log.info("释放锁，2");
        lock.unlock();
    }
}
```

源码分析：

<font style="color:rgb(76, 73, 72);">当我们没有传入锁对象来创建联锁的时候，则会抛出一个异常，反之则将我们传入的可变参数锁对象封装成一个集合</font>

```java
public RedissonMultiLock(RLock... locks) {
    if (locks.length == 0) {
        throw new IllegalArgumentException("Lock objects are not defined");
    } else {
        this.locks.addAll(Arrays.asList(locks));
    }
}
```

<font style="color:rgb(76, 73, 72);">联锁的tryLock</font>

```java
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long newLeaseTime = -1L;
    //如果传入了释放时间
    if (leaseTime != -1L) {
        //再判断一下是否有等待时间
        if (waitTime == -1L) {
            //如果没传等待时间，不重试，则只获得一次
            newLeaseTime = unit.toMillis(leaseTime);
        } else {
            //想要重试，耗时较久，万一释放时间小于等待时间，则会有问题，所以这里将等待时间乘以二
            newLeaseTime = unit.toMillis(waitTime) * 2L;
        }
    }
    //获取当前时间
    long time = System.currentTimeMillis();
    //剩余等待时间
    long remainTime = -1L;
    if (waitTime != -1L) {
        remainTime = unit.toMillis(waitTime);
    }
    //锁等待时间，与剩余等待时间一样    
    long lockWaitTime = this.calcLockWaitTime(remainTime);
    //锁失败的限制，源码返回是的0
    int failedLocksLimit = this.failedLocksLimit();
    //已经获取成功的锁
    List<RLock> acquiredLocks = new ArrayList(this.locks.size());
    //迭代器，用于遍历
    ListIterator<RLock> iterator = this.locks.listIterator();

    while(iterator.hasNext()) {
        RLock lock = (RLock)iterator.next();

        boolean lockAcquired;
        try {
            //没有等待时间和释放时间，调用空参的tryLock
            if (waitTime == -1L && leaseTime == -1L) {
                lockAcquired = lock.tryLock();
            } else {
                //否则调用带参的tryLock
                long awaitTime = Math.min(lockWaitTime, remainTime);
                lockAcquired = lock.tryLock(awaitTime, newLeaseTime, TimeUnit.MILLISECONDS);
            }
        } catch (RedisResponseTimeoutException var21) {
            this.unlockInner(Arrays.asList(lock));
            lockAcquired = false;
        } catch (Exception var22) {
            lockAcquired = false;
        }
        //判断获取锁是否成功
        if (lockAcquired) {
            //成功则将锁放入成功锁的集合
            acquiredLocks.add(lock);
        } else {
            //如果获取锁失败
            //判断当前锁的数量，减去成功获取锁的数量，如果为0，则所有锁都成功获取，跳出循环
            if (this.locks.size() - acquiredLocks.size() == this.failedLocksLimit()) {
                break;
            }
            //否则将拿到的锁都释放掉
            if (failedLocksLimit == 0) {
                this.unlockInner(acquiredLocks);
                //如果等待时间为-1，则不想重试，直接返回false
                if (waitTime == -1L) {
                    return false;
                }

                failedLocksLimit = this.failedLocksLimit();
                //将已经拿到的锁都清空
                acquiredLocks.clear();
                //将迭代器往前迭代，相当于重置指针，放到第一个然后重试获取锁
                while(iterator.hasPrevious()) {
                    iterator.previous();
                }
            } else {
                --failedLocksLimit;
            }
        }
        //如果剩余时间不为-1，很充足
        if (remainTime != -1L) {
            //计算现在剩余时间
            remainTime -= System.currentTimeMillis() - time;
            time = System.currentTimeMillis();
            //如果剩余时间为负数，则获取锁超时了
            if (remainTime <= 0L) {
                //将之前已经获取到的锁释放掉，并返回false
                this.unlockInner(acquiredLocks);
                //联锁成功的条件是：每一把锁都必须成功获取，一把锁失败，则都失败
                return false;
            }
        }
    }
    //如果设置了锁的有效期
    if (leaseTime != -1L) {
        List<RFuture<Boolean>> futures = new ArrayList(acquiredLocks.size());
        //迭代器用于遍历已经获取成功的锁
        Iterator var24 = acquiredLocks.iterator();

        while(var24.hasNext()) {
            RLock rLock = (RLock)var24.next();
            //设置每一把锁的有效期
            RFuture<Boolean> future = ((RedissonLock)rLock).expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
            futures.add(future);
        }

        var24 = futures.iterator();

        while(var24.hasNext()) {
            RFuture<Boolean> rFuture = (RFuture)var24.next();
            rFuture.syncUninterruptibly();
        }
    }
    //但如果没设置有效期，则会触发WatchDog机制，自动帮我们设置有效期，所以大多数情况下，我们不需要自己设置有效期
    return true;
}
```

小结

1. <font style="color:rgb(76, 73, 72);">不可重入Redis分布式锁</font>
    - <font style="color:rgb(76, 73, 72);">原理：利用SETNX的互斥性；利用EX避免死锁；释放锁时判断线程标识</font>
    - <font style="color:rgb(76, 73, 72);">缺陷：不可重入、无法重试、锁超时失效</font>
2. <font style="color:rgb(76, 73, 72);">可重入Redis分布式锁</font>
    - <font style="color:rgb(76, 73, 72);">原理：利用Hash结构，记录线程标识与重入次数；利用WatchDog延续锁时间；利用信号量控制锁重试等待</font>
    - <font style="color:rgb(76, 73, 72);">缺陷：Redis宕机引起锁失效问题</font>
3. <font style="color:rgb(76, 73, 72);">Redisson的multiLock</font>
    - <font style="color:rgb(76, 73, 72);">原理：多个独立的Redis节点，必须在所有节点都获取重入锁，才算获取锁成功</font>

  


