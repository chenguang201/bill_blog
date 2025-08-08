**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">【尚硅谷】Kafka3.x教程（从入门到调优，深入全面）</font>**](https://www.bilibili.com/video/BV1vr4y1677k?p=10&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

  
Flink 是一个在大数据开发中非常常用的组件。可以用于 Kafka 的生产者，也可以用于Flink 的消费者。

![](images/60.png)

# 1 Flink 环境准备
1.  创建一个 maven 项目 flink-kafka 
2. 添加配置文件

```xml
<dependencies>
  <dependency>
    <groupId>org.apache.flink</groupId> 
    flink-java</artifactId> 
    <version>1.13.0</version>
  </dependency>
  
  <dependency>
    <groupId>org.apache.flink</groupId> 
    flink-streaming-java_2.12</artifactId> 
    <version>1.13.0</version>
  </dependency>
  
  <dependency> 
    <groupId>org.apache.flink</groupId> 
    flink-clients_2.12</artifactId> 
    <version>1.13.0</version>
  </dependency>

  <dependency>
    <groupId>org.apache.flink</groupId> 
    flink-connector-kafka_2.12</artifactId> 
    <version>1.13.0</version>
   </dependency>
  
</dependencies>
```

3. 将 log4j.properties 文件添加到 resources 里面，就能更改打印日志的级别为 error

```properties
log4j.rootLogger=error, stdout,R 
log4j.appender.stdout=org.apache.log4j.ConsoleAppender 
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout 
log4j.appender.stdout.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p --- [%50t] %-80c(line:%5L) : %m%n

log4j.appender.R=org.apache.log4j.RollingFileAppender 
log4j.appender.R.File=../log/agent.log 
log4j.appender.R.MaxFileSize=1024KB 
log4j.appender.R.MaxBackupIndex=1

log4j.appender.R.layout=org.apache.log4j.PatternLayout 
log4j.appender.R.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss,SSS} %5p --- [%50t] %-80c(line:%6L) : %m%n
```

4. 在 java 文件夹下创建包名为 `com.atguigu.flink`

# 2 Flink生产者
1. 在`com.atguigu.flink` 包下创建 java类：`FlinkKafkaProducer1` 

```java
package com.atguigu.flink;

import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment; 
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import java.util.ArrayList; 
import java.util.Properties;

public class FlinkKafkaProducer1 {
    public static void main(String[] args) throws Exception { 
        // 0 初始化 flink 环境 
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(); 
        env.setParallelism(3);
        // 1 读取集合中数据
        ArrayList<String> wordsList = new ArrayList<>(); 
        wordsList.add("hello");
        wordsList.add("world");
        DataStream<String> stream = env.fromCollection(wordsList);
		// 2 kafka 生产者配置信息
		Properties properties = new Properties(); 
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "hadoop102:9092");
		// 3 创建 kafka 生产者
		FlinkKafkaProducer<String> kafkaProducer = new FlinkKafkaProducer<>("first",new SimpleStringSchema(), properties);
		// 4 生产者和 flink 流关联
		stream.addSink(kafkaProducer);
		// 5 执行
		env.execute(); 
    }
}
```

2. 启动 Kafka 消费者

```bash
[atguigu@hadoop104 kafka]$ bin/kafka-console-consumer.sh --bootstrap-server hadoop102:9092 --topic first
```

3. 执行 FlinkKafkaProducer1 程序，观察 kafka 消费者控制台情况

# 3 Flink 消费者
1. 在 `com.atguigu.flink` 包下创建 java 类：`FlinkKafkaConsumer1`

```java
package com.atguigu.flink;

import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer; 
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import java.util.Properties;

public class FlinkKafkaConsumer1 {
    public static void main(String[] args) throws Exception {
        // 0 初始化 flink 环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(3);
        // 1 kafka 消费者配置信息
		Properties properties = new Properties(); 
        properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"hadoop102:9092");
		// 2 创建 kafka 消费者
		FlinkKafkaConsumer<String> kafkaConsumer = new FlinkKafkaConsumer<>("first",new SimpleStringSchema(), properties);
	   	// 3 消费者和 flink 流关联 
        env.addSource(kafkaConsumer).print();
       	// 4 执行
		env.execute(); 
    }
}
```

2. 启动 FlinkKafkaConsumer1 消费者
3.  启动 kafka 生产者

```bash
[atguigu@hadoop103 kafka]$ bin/kafka-console-producer.sh --bootstrap-server hadoop102:9092 - -topic first
```

4. 观察 IDEA 控制台数据打印

