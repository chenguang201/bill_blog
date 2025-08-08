**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Redis入门到实战教程，深度透析redis底层原理+redis分布式锁+企业解决方案</font>**](https://www.bilibili.com/video/BV1cr4y1671t/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

基于CentOS7下的Redis集群教程，包括：

+ 单机安装Redis
+ Redis主从
+ Redis分片集群

# 1 单机安装Redis
首先需要安装Redis所需要的依赖：

```shell
yum install -y gcc tcl
```

然后将课前资料提供的Redis安装包上传到虚拟机的任意目录：  
![](images/105.png)

例如，我放到了/tmp目录：  
![](images/106.png)

解压缩：

```shell
tar -xzf redis-6.2.4.tar.gz
```

解压后：

![](images/107.png)



进入redis目录：

```shell
cd redis-6.2.4
```



运行编译命令：

```shell
make && make install
```

如果没有出错，应该就安装成功了。



然后修改redis.conf文件中的一些配置：

```properties
# 39. 绑定地址，默认是127.0.0.1，会导致只能在本地访问。修改为0.0.0.0则可以在任意IP访问
bind 0.0.0.0
# 40. 保护模式，关闭保护模式
protected-mode no
# 41. 数据库数量，设置为1
databases 1
```



启动Redis：

```shell
redis-server redis.conf
```



停止redis服务：

```shell
redis-cli shutdown
```



# 2 Redis主从集群
## 2.1 集群结构
我们搭建的主从集群结构如图：  
![](images/108.png)

共包含三个节点，一个主节点，两个从节点。

这里我们会在同一台虚拟机中开启3个redis实例，模拟主从集群，信息如下：

| IP | PORT | 角色 |
| :---: | :---: | :---: |
| 192.168.150.101 | 7001 | master |
| 192.168.150.101 | 7002 | slave |
| 192.168.150.101 | 7003 | slave |


## 2.2.准备实例和配置
要在同一台虚拟机开启3个实例，必须准备三份不同的配置文件和目录，配置文件所在目录也就是工作目录。

1）创建目录

我们创建三个文件夹，名字分别叫7001、7002、7003：

```shell
# 42. 进入/tmp目录
cd /tmp
# 43. 创建目录
mkdir 7001 7002 7003
```

如图：  
![](images/109.png)

2）恢复原始配置

修改redis-6.2.4/redis.conf文件，将其中的持久化模式改为默认的RDB模式，AOF保持关闭状态。

```properties
# 44. 开启RDB
# 45. save ""
save 3600 1
save 300 100
save 60 10000

# 46. 关闭AOF
appendonly no
```



3）拷贝配置文件到每个实例目录

然后将redis-6.2.4/redis.conf文件拷贝到三个目录中（在/tmp目录执行下列命令）：

```shell
# 47. 方式一：逐个拷贝
cp redis-6.2.4/redis.conf 7001
cp redis-6.2.4/redis.conf 7002
cp redis-6.2.4/redis.conf 7003

# 48. 方式二：管道组合命令，一键拷贝
echo 7001 7002 7003 | xargs -t -n 1 cp redis-6.2.4/redis.conf
```



4）修改每个实例的端口、工作目录

修改每个文件夹内的配置文件，将端口分别修改为7001、7002、7003，将rdb文件保存位置都修改为自己所在目录（在/tmp目录执行下列命令）：

```shell
sed -i -e 's/6379/7001/g' -e 's/dir .\//dir \/tmp\/7001\//g' 7001/redis.conf
sed -i -e 's/6379/7002/g' -e 's/dir .\//dir \/tmp\/7002\//g' 7002/redis.conf
sed -i -e 's/6379/7003/g' -e 's/dir .\//dir \/tmp\/7003\//g' 7003/redis.conf
```



5）修改每个实例的声明IP

虚拟机本身有多个IP，为了避免将来混乱，我们需要在redis.conf文件中指定每一个实例的绑定ip信息，格式如下：

```properties
# 49. redis实例的声明 IP
replica-announce-ip 192.168.150.101
```



每个目录都要改，我们一键完成修改（在/tmp目录执行下列命令）：

```shell
# 50. 逐一执行
sed -i '1a replica-announce-ip 192.168.150.101' 7001/redis.conf
sed -i '1a replica-announce-ip 192.168.150.101' 7002/redis.conf
sed -i '1a replica-announce-ip 192.168.150.101' 7003/redis.conf

# 51. 或者一键修改
printf '%s
' 7001 7002 7003 | xargs -I{} -t sed -i '1a replica-announce-ip 192.168.150.101' {}/redis.conf
```



## 2.3 启动
为了方便查看日志，我们打开3个ssh窗口，分别启动3个redis实例，启动命令：

```shell
# 52. 第1个
redis-server 7001/redis.conf
# 53. 第2个
redis-server 7002/redis.conf
# 54. 第3个
redis-server 7003/redis.conf
```



启动后：

![](images/110.png)

如果要一键停止，可以运行下面命令：

```shell
printf '%s
' 7001 7002 7003 | xargs -I{} -t redis-cli -p {} shutdown
```



## 2.4 开启主从关系
现在三个实例还没有任何关系，要配置主从可以使用replicaof 或者slaveof（5.0以前）命令。

有临时和永久两种模式：

+  修改配置文件（永久生效） 
    - 在redis.conf中添加一行配置：`<font style="color:#E8323C;">slaveof <masterip> <masterport></font>`
+  使用redis-cli客户端连接到redis服务，执行slaveof命令（重启后失效）： 

```shell
slaveof <masterip> <masterport>
```

**注意**：在5.0以后新增命令replicaof，与salveof效果一致。

这里我们为了演示方便，使用方式二。

通过redis-cli命令连接7002，执行下面命令：

```shell
# 55. 连接 7002
redis-cli -p 7002
# 56. 执行slaveof
slaveof 192.168.150.101 7001
```



通过redis-cli命令连接7003，执行下面命令：

```shell
# 57. 连接 7003
redis-cli -p 7003
# 58. 执行slaveof
slaveof 192.168.150.101 7001
```



然后连接 7001节点，查看集群状态：

```shell
# 59. 连接 7001
redis-cli -p 7001
# 60. 查看状态
info replication
```



结果：  
![](images/111.png)

## 2.5 测试
执行下列操作以测试：

+  利用redis-cli连接7001，执行`<font style="color:#E8323C;">set num 123</font>` 
+  利用redis-cli连接7002，执行`<font style="color:#E8323C;">get num</font>`，再执行`<font style="color:#E8323C;">set num 666</font>` 
+  利用redis-cli连接7003，执行`<font style="color:#E8323C;">get num</font>`，再执行`<font style="color:#E8323C;">set num 888</font>`<font style="color:#E8323C;"> </font>

可以发现，只有在7001这个master节点上可以执行写操作，7002和7003这两个slave节点只能执行读操作。

# 3 搭建哨兵集群
## 3.1 集群结构
这里我们搭建一个三节点形成的Sentinel集群，来监管之前的Redis主从集群。如图：



![](images/112.png)

三个sentinel实例信息如下：

| 节点 | IP | PORT |
| --- | :---: | :---: |
| s1 | 192.168.150.101 | 27001 |
| s2 | 192.168.150.101 | 27002 |
| s3 | 192.168.150.101 | 27003 |


## 3.2 准备实例和配置
要在同一台虚拟机开启3个实例，必须准备三份不同的配置文件和目录，配置文件所在目录也就是工作目录。

我们创建三个文件夹，名字分别叫s1、s2、s3：

```shell
# 61. 进入/tmp目录
cd /tmp
# 62. 创建目录
mkdir s1 s2 s3
```



如图：

![](images/113.png)



然后我们在s1目录创建一个sentinel.conf文件，添加下面的内容：

```properties
port 27001
sentinel announce-ip 192.168.150.101
sentinel monitor mymaster 192.168.150.101 7001 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
dir "/tmp/s1"
```

解读：

+ `port 27001`：是当前sentinel实例的端口
+ `sentinel monitor mymaster 192.168.150.101 7001 2`：指定主节点信息 
    - `mymaster`：主节点名称，自定义，任意写
    - `192.168.150.101 7001`：主节点的ip和端口
    - `2`：选举master时的quorum值



然后将s1/sentinel.conf文件拷贝到s2、s3两个目录中（在/tmp目录执行下列命令）：

```shell
# 63. 方式一：逐个拷贝
cp s1/sentinel.conf s2
cp s1/sentinel.conf s3
# 64. 方式二：管道组合命令，一键拷贝
echo s2 s3 | xargs -t -n 1 cp s1/sentinel.conf
```



修改s2、s3两个文件夹内的配置文件，将端口分别修改为27002、27003：

```shell
sed -i -e 's/27001/27002/g' -e 's/s1/s2/g' s2/sentinel.conf
sed -i -e 's/27001/27003/g' -e 's/s1/s3/g' s3/sentinel.conf
```



## 3.3 启动
为了方便查看日志，我们打开3个ssh窗口，分别启动3个redis实例，启动命令：

```shell
# 65. 第1个
redis-sentinel s1/sentinel.conf
# 66. 第2个
redis-sentinel s2/sentinel.conf
# 67. 第3个
redis-sentinel s3/sentinel.conf
```



启动后：

![](images/114.png)



## 3.4 测试
尝试让master节点7001宕机，查看sentinel日志：  
![](images/115.png)



查看7003的日志：

![](images/116.png)



查看7002的日志：  
![](images/117.png)



# 4 搭建分片集群
## 4.1 集群结构
分片集群需要的节点数量较多，这里我们搭建一个最小的分片集群，包含3个master节点，每个master包含一个slave节点，结构如下：



![](images/118.png)



这里我们会在同一台虚拟机中开启6个redis实例，模拟分片集群，信息如下：

| IP | PORT | 角色 |
| :---: | :---: | :---: |
| 192.168.150.101 | 7001 | master |
| 192.168.150.101 | 7002 | master |
| 192.168.150.101 | 7003 | master |
| 192.168.150.101 | 8001 | slave |
| 192.168.150.101 | 8002 | slave |
| 192.168.150.101 | 8003 | slave |


## 4.2 准备实例和配置
删除之前的7001、7002、7003这几个目录，重新创建出7001、7002、7003、8001、8002、8003目录：

```shell
# 68. 进入/tmp目录
cd /tmp
# 69. 删除旧的，避免配置干扰
rm -rf 7001 7002 7003
# 70. 创建目录
mkdir 7001 7002 7003 8001 8002 8003
```



在/tmp下准备一个新的redis.conf文件，内容如下：

```properties
port 6379
# 71. 开启集群功能
cluster-enabled yes
# 72. 集群的配置文件名称，不需要我们创建，由redis自己维护
cluster-config-file /tmp/6379/nodes.conf
# 73. 节点心跳失败的超时时间
cluster-node-timeout 5000
# 74. 持久化文件存放目录
dir /tmp/6379
# 75. 绑定地址
bind 0.0.0.0
# 76. 让redis后台运行
daemonize yes
# 77. 注册的实例ip
replica-announce-ip 192.168.150.101
# 78. 保护模式
protected-mode no
# 79. 数据库数量
databases 1
# 80. 日志
logfile /tmp/6379/run.log
```



将这个文件拷贝到每个目录下：

```shell
# 81. 进入/tmp目录
cd /tmp
# 82. 执行拷贝
echo 7001 7002 7003 8001 8002 8003 | xargs -t -n 1 cp redis.conf
```



修改每个目录下的redis.conf，将其中的6379修改为与所在目录一致：

```shell
# 83. 进入/tmp目录
cd /tmp
# 84. 修改配置文件
printf '%s
' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t sed -i 's/6379/{}/g' {}/redis.conf
```



## 4.3 启动
因为已经配置了后台启动模式，所以可以直接启动服务：

```shell
# 85. 进入/tmp目录
cd /tmp
# 86. 一键启动所有服务
printf '%s
' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t redis-server {}/redis.conf
```



通过ps查看状态：

```shell
ps -ef | grep redis
```



发现服务都已经正常启动：  
![](images/119.png)



如果要关闭所有进程，可以执行命令：

```shell
ps -ef | grep redis | awk '{print $2}' | xargs kill
```



或者（推荐这种方式）：

```shell
printf '%s
' 7001 7002 7003 8001 8002 8003 | xargs -I{} -t redis-cli -p {} shutdown
```



## 4.4 创建集群
虽然服务启动了，但是目前每个服务之间都是独立的，没有任何关联。

我们需要执行命令来创建集群，在Redis5.0之前创建集群比较麻烦，5.0之后集群管理命令都集成到了redis-cli中。



1）Redis5.0之前

Redis5.0之前集群命令都是用redis安装包下的src/redis-trib.rb来实现的。因为redis-trib.rb是有ruby语言编写的所以需要安装ruby环境。

```shell
# 87. 安装依赖
yum -y install zlib ruby rubygems
gem install redis
```



然后通过命令来管理集群：

```shell
# 88. 进入redis的src目录
cd /tmp/redis-6.2.4/src
# 89. 创建集群
./redis-trib.rb create --replicas 1 192.168.150.101:7001 192.168.150.101:7002 192.168.150.101:7003 192.168.150.101:8001 192.168.150.101:8002 192.168.150.101:8003
```



2）Redis5.0以后

我们使用的是Redis6.2.4版本，集群管理以及集成到了redis-cli中，格式如下：

```shell
redis-cli --cluster create --cluster-replicas 1 192.168.150.101:7001 192.168.150.101:7002 192.168.150.101:7003 192.168.150.101:8001 192.168.150.101:8002 192.168.150.101:8003
```

命令说明：

+ `redis-cli --cluster`或者`./redis-trib.rb`：代表集群操作命令
+ `create`：代表是创建集群
+ `--replicas 1`或者`--cluster-replicas 1` ：指定集群中每个master的副本个数为1，此时`节点总数 ÷ (replicas + 1)` 得到的就是master的数量。因此节点列表中的前n个就是master，其它节点都是slave节点，随机分配到不同master

运行后的样子：  
![](images/120.png)



这里输入yes，则集群开始创建：  
![](images/121.png)



通过命令可以查看集群状态：

```shell
redis-cli -p 7001 cluster nodes
```

![](images/122.png)



## 4.5 测试
尝试连接7001节点，存储一个数据：

```shell
# 90. 连接
redis-cli -p 7001
# 91. 存储数据
set num 123
# 92. 读取数据
get num
# 93. 再次存储
set a 1
```

结果悲剧了：  
![](images/123.png)

集群操作时，需要给`redis-cli`加上`-c`参数才可以：

```shell
redis-cli -c -p 7001
```

这次可以了：  
![](images/124.png)

