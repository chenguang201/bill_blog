1. 下载VirtualBox，参考链接

[Win10 安装VirtualBox](https://www.yuque.com/chenguang201/java/en5ty7kwp9x4owvw)

2. 安装 Centos7, 并配置网络。

[https://juejin.cn/post/7294555030726688795?searchId=2025010416432564BE54467EFA2EDDE05B](https://juejin.cn/post/7294555030726688795?searchId=2025010416432564BE54467EFA2EDDE05B)

3. 在宿主机上和虚拟机上分别 ping 一下对方的 IP。此时遇到一个问题，在宿主机上可以ping 通 虚拟机的IP。但是在虚拟机上ping 不同宿主机的IP。虚拟机上的防火墙已经关闭。解决方案如下：

[如何解决虚拟机ping不同主机，主机可以ping通虚拟机_虚拟机winxp怎ping通宿主以外的主机-CSDN博客](https://blog.csdn.net/snow_7/article/details/80555135)

# 191. 操作流程
## 1 创建服务
1. 点击新建按钮，创建 centos 服务

![](images/11.png)

2. 选择对应的镜像和其他参数

![](images/12.png)

3. 设置内存和CPU 大小

![](images/13.png)

4. 设置硬盘大小

![](images/14.png)

5. 预览新建的虚拟机配置

![](images/15.png)

6. 创建完成

![](images/16.png)

## 2 开始装机
1. 点击正常启动

![](images/17.png)

2. 选择第二个选项

![](images/18.png)

3. 选择语言，点击继续

![](images/19.png)

4. 选择安装信息摘要
    1. 本地化

![](images/20.png)

    2. 选择软件

![](images/21.png)

![](images/22.png)

    3. 选择安装位置和网络主机名

![](images/23.png)

![](images/24.png)

5. 开始安装，设置root密码和创建用户

![](images/25.png)

![](images/26.png)

![](images/27.png)

6. 重启

![](images/28.png)

7. 初始设置

![](images/29.png)

8. 用admin 用户登录

![](images/30.png)

9. 查看虚拟机ip，并 ping 主机看是否能ping 通。发现网络并不通

![](images/31.png)

## 3 设置网络
1. 点击设置

![](images/32.png)

2. 选择网络，并选择桥接模式

![](images/33.png)

3. 开始设置静态网络

```powershell
su root
cd /etc/sysconfig/network-scripts/
vi ifcfg-enp0s3
```

```powershell
TYPE=Etherneti
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=enp0s3
UUID=91f8a4a6-0ff0-412a-8f84-b48e4f615efb
DEVICE=enp0s3
ONBOOT=yes

IPADDR=192.168.3.101
GATEWAY=192.168.3.1
NETMASK=255.255.255.0
DNS1=8.8.8.8
```

![](images/34.png)

![](images/35.png)

4. 重启网络，并重新ping 宿主机，如果ping 不通，请注意关闭防火墙

```powershell
systemctl restart network
```

![](images/36.png)

## 4 发现 yum 无法用的问题
现象：

![](images/37.png)

解决方法：

```powershell
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum clean all    
yum  makecache
```

然后正常安装。

参考文档：

[Cannot find a valid baseurl for repo: base/7/x86_64-CSDN博客](https://blog.csdn.net/2302_80164634/article/details/144829880)

