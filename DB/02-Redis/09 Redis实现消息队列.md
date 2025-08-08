**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Redis入门到实战教程，深度透析redis底层原理+redis分布式锁+企业解决方案</font>**](https://www.bilibili.com/video/BV1cr4y1671t/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

## 1 认识消息队列
什么是消息队列：字面意思就是存放消息的队列。最简单的消息队列模型包括3个角色：

+ **消息队列**：存储和管理消息，也被称为消息代理（Message Broker）
+ **生产者**：发送消息到消息队列
+ **消费者**：从消息队列获取消息并处理消息

![](images/61.png)



使用队列的好处在于**解耦：**所谓解耦，举一个生活中的例子就是：快递员(生产者)把快递放到快递柜里边(Message Queue)去，我们(消费者)从快递柜里边去拿东西，这就是一个异步，如果耦合，那么这个快递员相当于直接把快递交给你，这事固然好，但是万一你不在家，那么快递员就会一直等你，这就浪费了快递员的时间，所以这种思想在我们日常开发中，是非常有必要的。

这种场景在我们秒杀中就变成了：我们下单之后，利用redis去进行校验下单条件，再通过队列把消息发送出去，然后再启动一个线程去消费这个消息，完成解耦，同时也加快我们的响应速度。

这里我们可以使用一些现成的mq，比如kafka，rabbitmq等等，但是呢，如果没有安装mq，我们也可以直接使用redis提供的mq方案，降低我们的部署和学习成本。

## 2 基于List实现消息队列
**基于List结构模拟消息队列**

消息队列（Message Queue），字面意思就是存放消息的队列。而Redis的list数据结构是一个双向链表，很容易模拟出队列效果。

队列是入口和出口不在一边，因此我们可以利用：LPUSH 结合 RPOP、或者 RPUSH 结合 LPOP来实现。  
不过要注意的是，当队列中没有消息时RPOP或LPOP操作会返回null，并不像JVM的阻塞队列那样会阻塞并等待消息。因此这里应该使用BRPOP或者BLPOP来实现阻塞效果。

![](images/62.png)



基于List的消息队列有哪些优缺点？  
**优点**：

+ 利用Redis存储，不受限于JVM内存上限
+ 基于Redis的持久化机制，数据安全性有保证
+ 可以满足消息有序性

**缺点**：

+ 无法避免消息丢失
+ 只支持单消费者

## 3 基于PubSub的消息队列
PubSub（发布订阅）是Redis2.0版本引入的消息传递模型。顾名思义，消费者可以订阅一个或多个channel，生产者向对应channel发送消息后，所有订阅者都能收到相关消息。

**SUBSCRIBE channel [channel] **：订阅一个或多个频道  
**PUBLISH channel msg **：向一个频道发送消息  
**PSUBSCRIBE pattern[pattern] **：订阅与pattern格式匹配的所有频道

![](images/63.png)



基于PubSub的消息队列有哪些优缺点？  
**优点：**

+ 采用发布订阅模型，支持多生产、多消费

**缺点：**

+ 不支持数据持久化
+ 无法避免消息丢失
+ 消息堆积有上限，超出时数据丢失

## 4 基于Stream的消息队列
Stream 是 Redis 5.0 引入的一种新数据类型，可以实现一个功能非常完善的消息队列。

发送消息的命令：

![](images/64.png)



例如：

![](images/65.png)



读取消息的方式之一：XREAD

![](images/66.png)



例如，使用XREAD读取第一个消息：

![](images/67.png)



XREAD阻塞方式，读取最新的消息：

![](images/68.png)



在业务开发中，我们可以循环的调用XREAD阻塞方式来查询最新消息，从而实现持续监听队列的效果，伪代码如下

```java
while (true){
    //尝试读取队列中的消息，最多阻塞2秒
    Object msg = redis.execute("XREAD COUNT 1 BLOCK 2000 STREAMS users $");
    //没读取到，跳过下面的逻辑
    if(msg == null){
        continue;
    }
    //处理消息
    handleMessage(msg);
}
```

注意：当我们指定起始ID为$时，代表读取最新的消息，如果我们处理一条消息的过程中，又有超过1条以上的消息到达队列，则下次获取时也只能获取到最新的一条，会出现漏读消息的问题

STREAM类型消息队列的XREAD命令特点：

+ 消息可回溯
+ 一个消息可以被多个消费者读取
+ 可以阻塞读取
+ 有消息漏读的风险

## 5 基于Stream的消息队列-消费者组
消费者组（Consumer Group）：将多个消费者划分到一个组中，监听同一个队列。具备下列特点：

![](images/69.png)



**创建消费者组**

```java
XGROUP CREATE key groupName ID [MKSTREAM]
    
//key：队列名称
//groupName：消费者组名称
//ID：起始ID标示，$代表队列中最后一个消息，0则代表队列中第一个消息
//MKSTREAM：队列不存在时自动创建队列
```



**删除指定的消费者组**

```java
XGROUP DESTORY key groupName
```



**给指定的消费者组添加消费者**

```java
XGROUP CREATECONSUMER key groupname consumername
```



**删除消费者组中的指定消费者**

```java
XGROUP DELCONSUMER key groupname consumername
```



从消费者组读取消息：

```java
XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...]
//group：消费组名称
//consumer：消费者名称，如果消费者不存在，会自动创建一个消费者
//count：本次查询的最大数量
//BLOCK milliseconds：当没有消息时最长等待时间
//NOACK：无需手动ACK，获取到消息后自动确认
//STREAMS key：指定队列名称
//ID：获取消息的起始ID： 
```

消费者监听消息的基本思路：

```java
while(true){
    // 尝试监听队列，使用阻塞模式，最大等待时长为2000ms
    Object msg = redis.call("XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 >")
    if(msg == null){
        // 没监听到消息，重试
        continue;
    }
    try{
        //处理消息，完成后要手动确认ACK，ACK代码在handleMessage中编写
        handleMessage(msg);
    } catch(Exception e){
        while(true){
            //0表示从pending-list中的第一个消息开始，如果前面都ACK了，那么这里就不会监听到消息
            Object msg = redis.call("XREADGROUP GROUP g1 c1 COUNT 1 STREAMS s1 0");
            if(msg == null){
                //null表示没有异常消息，所有消息均已确认，结束循环
                break;
            }
            try{
                //说明有异常消息，再次处理
                handleMessage(msg);
            } catch(Exception e){
                //再次出现异常，记录日志，继续循环
                log.error("..");
                continue;
            }
        }
    }
}
```

STREAM类型消息队列的XREADGROUP命令特点：

+ 消息可回溯
+ 可以多消费者争抢消息，加快消费速度
+ 可以阻塞读取
+ 没有消息漏读的风险
+ 有消息确认机制，保证消息至少被消费一次

最后我们来个小对比

![](images/70.png)

## 6 Stream结构作为消息队列实现异步秒杀下单
需求：

+ 创建一个Stream类型的消息队列，名为stream.orders
+ 修改之前的秒杀下单Lua脚本，在认定有抢购资格后，直接向stream.orders中添加消息，内容包含voucherId、userId、orderId
+ 项目启动时，开启一个线程任务，尝试获取stream.orders中的消息，完成下单\



修改lua表达式

```lua
-- 订单id
local voucherId = ARGV[1]
-- 用户id
local userId = ARGV[2]
-- 新增orderId，但是变量名用id就好，因为VoucherOrder实体类中的orderId就是用id表示的
local id = ARGV[3]
-- 优惠券key
local stockKey = 'seckill:stock:' .. voucherId
-- 订单key
local orderKey = 'seckill:order:' .. voucherId
-- 判断库存是否充足
if (tonumber(redis.call('get', stockKey)) <= 0) then
  return 1
end
-- 判断用户是否下单
if (redis.call('sismember', orderKey, userId) == 1) then
  return 2
end
-- 扣减库存
redis.call('incrby', stockKey, -1)
-- 将userId存入当前优惠券的set集合
redis.call('sadd', orderKey, userId)
-- 将下单数据保存到消息队列中
redis.call("sadd", 'stream.orders', '*', 'userId', userId, 'voucherId', voucherId, 'id', id)
return 0
```



VoucherOrderServiceImpl

```java
private class VoucherOrderHandler implements Runnable {

    @Override
    public void run() {
        while (true) {
            try {
                // 1.获取消息队列中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 >
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),
                    StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                    StreamOffset.create("stream.orders", ReadOffset.lastConsumed())
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有消息，继续下一次循环
                    continue;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("s1", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理订单异常", e);
                //处理异常消息
                handlePendingList();
            }
        }
    }

    private void handlePendingList() {
        while (true) {
            try {
                // 1.获取pending-list中的订单信息 XREADGROUP GROUP g1 c1 COUNT 1 BLOCK 2000 STREAMS s1 0
                List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                    Consumer.from("g1", "c1"),
                    StreamReadOptions.empty().count(1),
                    StreamOffset.create("stream.orders", ReadOffset.from("0"))
                );
                // 2.判断订单信息是否为空
                if (list == null || list.isEmpty()) {
                    // 如果为null，说明没有异常消息，结束循环
                    break;
                }
                // 解析数据
                MapRecord<String, Object, Object> record = list.get(0);
                Map<Object, Object> value = record.getValue();
                VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(value, new VoucherOrder(), true);
                // 3.创建订单
                createVoucherOrder(voucherOrder);
                // 4.确认消息 XACK
                stringRedisTemplate.opsForStream().acknowledge("s1", "g1", record.getId());
            } catch (Exception e) {
                log.error("处理pendding订单异常", e);
                try{
                    Thread.sleep(20);
                }catch(Exception e){
                    e.printStackTrace();
                }
            }
        }
    }
}
```



