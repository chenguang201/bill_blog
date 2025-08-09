**笔记来源：** [08.Jenkins软件介绍_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1kJ411p7mV?spm_id_from=333.788.videopod.episodes&vd_source=e8046ccbdc793e09a75eb61fe8e84a30&p=8)

------



# 1 Jenkins介绍
![](images/6.png)

Jenkins 是一款流行的开源持续集成（Continuous Integration）工具，广泛用于项目开发，具有自动化构建、测试和部署等功能。官网：[Jenkins](https://www.jenkins.io/)

Jenkins的特征：

+ 开源的Java语言开发持续集成工具，支持持续集成，持续部署。
+ 易于安装部署配置：可通过 yum 安装，或下载 war 包以及通过 docker 容器等快速实现安装部署，可方便web界面配置管理。 
+ 消息通知及测试报告：集成RSS/E-mail 通过 RSS 发布构建结果或当构建完成时通过 e-mail 通知，生成JUnit/TestNG测试报告。
+ 分布式构建：Jenkins支持能够让多台计算机一起构建/测试。 
+ 文件识别：Jenkins能够跟踪哪次构建生成哪些jar，哪次构建使用哪个版本的jar等。
+ 丰富的插件支持：支持扩展插件，你可以开发适合自己团队使用的工具，如 git，svn，maven，docker等。

# 2 Jenkins 安装和持续集成配置
持续集成流程说明

![画板](images/7.jpeg)

1. 首先，开发人员每天进行代码提交，提交到Git仓库
2. 然后，Jenkins作为持续集成工具，使用Git工具到Git仓库拉取代码到集成服务器，再配合JDK， Maven等软件完成代码编译，代码测试与审查，测试，打包等工作，在这个过程中每一步出错，都重新再执行一次整个流程。 
3. 最后，Jenkins 把生成的 jar 或 war 包分发到测试服务器或者生产服务器，测试人员或用户就可以访问应用。

**服务器列表：** 本此实验虚拟机统一采用CentOS7。CentOS7的安装流程参考一下文档：

[Win10 安装VirtualBox](https://www.yuque.com/chenguang201/java/en5ty7kwp9x4owvw)

[利用VirtualBox 安装Centos7](https://www.yuque.com/chenguang201/java/xnatznvgbk160et1)

| 名称      | IP地址          | 安装的软件                                    |
| ------- | ------------- | ---------------------------------------- |
| 代码托管服务器 | 192.168.3.100 | Gitlab-12.4.2                            |
| 持续集成服务器 | 192.168.3.101 | Jenkins-2.190.3，JDK1.8，Maven3.6.2，Git， SonarQube |
| 应用测试服务器 | 192.168.3.102 | JDK1.8，Tomcat8.5                         |


## 2.1 GitLab代码托管服务器安装
### 2.1.1 GitLab 简介
[The most-comprehensive AI-powered DevSecOps platform](https://about.gitlab.com/)

GitLab 是一个用于仓库管理系统的开源项目，使用Git作为代码管理工具，并在此基础上搭建起来的web服务。 

GitLab 和 GitHub 一样属于第三方基于Git开发的作品，免费且开源（基于MIT协议），与 Github 类似， 可以注册用户，任意提交你的代码，添加SSHKey等等。不同的是，**GitLab是可以部署到自己的服务器上，数据库等一切信息都掌握在自己手上，适合团队内部协作开发**， 你总不可能把团队内部的智慧总放在别人的服务器上吧？简单来说可把GitLab看作个人版的GitHub。

### 2.1.2 Linux安装GitLab流程
1. 安装相关依赖

   >```powershell
   >yum -y install policycoreutils openssh-server openssh-clients postfix
   >```


2. 启动ssh服务并设置为开机启动

   > ```powershell
   > systemctl enable sshd && sudo systemctl start sshd
   > ```


3. 设置postfix开机自启，并启动，postfix支持gitlab发信功能

   >```powershell
   >systemctl enable postfix && systemctl start postfix
   >```


4. 开放ssh以及http服务，然后重新加载防火墙列表，如果关闭防火墙就不需要做以下配置

   >```powershell
   >firewall-cmd --add-service=ssh --permanent
   >firewall-cmd --add-service=http --permanent
   >firewall-cmd --reload
   >
   ># 185. 关闭防火墙
   >systemctl stop firewalld.service
   >systemctl status firewalld.service
   >systemctl start firewalld.service
   >systemctl restart firewalld.service
   >```


5. 下载gitlab包，并且安装

   >```powershell
   ># 在线下载：
   >wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el6/gitlab-ce-12.4.2-ce.0.el6.x86_64.rpm
   >```
   >
   >[gitlab-ce-12.4.2-ce.0.el6.x86_64.rpm](https://www.yuque.com/attachments/yuque/0/2025/rpm/22334924/1736584521404-1a58e96c-b6fa-477e-868c-0f22b1cf21c6.rpm)
   >
   >```powershell
   ># 安装命令：
   >rpm -i gitlab-ce-12.4.2-ce.0.el6.x86_64.rpm
   >```


6. 修改gitlab配置

   >```powershell
   >vi /etc/gitlab/gitlab.rb
   ># 186. 修改gitlab访问地址和端口，默认为80，我们改为82
   >external_url 'http://192.168.33.100:82'
   >nginx['listen_port'] = 82
   >```


7. 重载配置及启动gitlab

   >```powershell
   >gitlab-ctl reconfigure
   >gitlab-ctl restart
   >```


8. 把端口添加到防火墙

   >```powershell
   >firewall-cmd --zone=public --add-port=82/tcp --permanent
   >firewall-cmd --reload
   >```


9. 访问：[http://192.168.3.100:82](http://192.168.3.100:82/)

   ![](images/8.png)

### 2.1.3 Gitlab的操作
接着上面的安装流程，在浏览器上访问页面后，直接可以改密码操作。

**创建组操作**： 使用管理员 root 创建组，一个组里面可以有多个项目分支，可以将开发添加到组里面进行设置权限，不同的组就是公司不同的开发项目或者服务模块，不同的组添加不同的开发即可实现对开发设置权限的管理。操作流程如下：

1. 用root 账户登录上服务

   ![](images/9.png)


2. 点击 creat a group 菜单。

   ![](images/10.png)


3. 填写对应的信息，点击确认。

   ![](images/11.png)

   ![](images/12.png)

**创建用户操作：** 创建用户的时候，可以选择Regular或Admin类型。流程如下：

1. 再次点击首页图标，回到首页，点击add people 选项![](images/13.png)


2. 填写所有的选项，然后点击确认。![](images/14.png)


3. 创建完用户后，立即修改密码![](images/15.png)


4. 输入密码，点击确认更改![](images/16.png)


5. 将用户添加进组中。选择某个用户组，进行Members管理组的成员![](images/17.png)


6. 搜索用户，同时给用户赋予不同的角色，不同的角色有不同的权限。![](images/18.png)![](images/19.png)

Gitlab用户在组里面有5种不同权限： 

+ Guest：可以创建issue、发表评论，不能读写版本库
+ Reporter：可以克隆代码，不能提交，QA、PM 可以赋予这个权限
+ Developer：可以克隆代码、开发、提交、push，普通开发可以赋予这个权限 
+ Maintainer：可以创建项目、添加tag、保护分支、添加项目成员、编辑项目，核心开发可以赋予这个权限 
+ Owner：可以设置项目访问权限 - Visibility Level、删除项目、迁移项目、管理组成员，开发组组长可以赋予这个权限



**创建项目操作流程：** 

1. 以刚才创建的新用户身份登录到Gitlab![](images/20.png)


2. 在用户组中创建新的项目![](images/21.png)


3. 填写对应的选项，然后点击确认![](images/22.png)


4. 此时项目就创建好了![](images/23.png)

注意一点：我们平时从Gitlab 或者其他代码托管平台拉取代码有两种方式：

+ HTTP方式：这种方式需要传统的username 和password。而这个username 和password就是我们需要登录代码托管平台时候的账号和密码。比如上面的就是zhangsan和xxxxxx。
+ SSH方式：这种方式是免密登录的方式，需要生成公钥和密钥。具体操作可以参考下面的文档。

[SSH 公钥设置 | Gitee 帮助中心](https://help.gitee.com/base/account/SSH%E5%85%AC%E9%92%A5%E8%AE%BE%E7%BD%AE)

### 2.1.4 源码上传到GitLab
下面来到IDEA开发工具，我们已经准备好一个简单的Web应用准备到集成部署。 我们要把源码上传到Gitlab的项目仓库中。

1. 项目结构说明：我们建立了一个非常简单的web应用，只有一个index.jsp页面，如果部署好，可以访问该页面就成功啦！![](images/24.png)


2. 开启本地代码和远程仓库的绑定。![](images/25.png)


3. 填写远程的仓库地址![](images/26.png)


4. 点击OK，发现需要个Token  ![](images/27.png)


5. 生成token

   ![](images/28.png)

6. 将生成的Token 输入在IDEA中，点击登录![](images/29.png)


7. 然后我们发现，GitLab的版本太老，不支持这种Token这种模式，点击右下角的Log in via GitLab![](images/30.png)
8. 然后重新输入用户名和密码的方式来登录，至此我们将远程仓库和本地仓库绑定好了![](images/31.png)


9. 开始进行add、commit、push等操作![](images/32.png)

我们发现代码已经被提交到远程仓库了。





