**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Redis入门到实战教程，深度透析redis底层原理+redis分布式锁+企业解决方案</font>**](https://www.bilibili.com/video/BV1cr4y1671t/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)



安装完成Redis，我们就可以操作Redis，实现数据的CRUD了。这需要用到Redis客户端，包括：

+ **命令行客户端**
+ **图形化桌面客户端**
+ **编程客户端**



## 1 命令行客户端
+  **Redis安装完成后就自带了命令行客户端：**`**<font style="color:#E8323C;">redis-cli</font>**`**，使用方式如下：** 

```shell
redis-cli [options] [commonds]
```

+  **其中常见的**`**options**`**有：** 
    - `-h 127.0.0.1`：指定要连接的redis节点的IP地址，默认是127.0.0.1
    - `-p 6379`：指定要连接的redis节点的端口，默认是6379
    - `-a 132537`：指定redis的访问密码
+  **其中的**`**commonds**`**就是Redis的操作命令，例如：** 
    - `ping`：与redis服务端做心跳测试，服务端正常会返回`pong`
    - 不指定commond时，会进入`redis-cli`的交互控制台：

