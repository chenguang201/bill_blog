**<font style="color:#DF2A3F;background-color:#FFFFFF;">笔记来源：</font>**[**<font style="color:#DF2A3F;background-color:#FFFFFF;">k8s（Kubernetes）集群编排工具helm3实战教程</font>**](https://www.bilibili.com/video/BV12D4y1Y7Z7/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)



# 29. <font style="color:#000000;">1 环境准备</font>
<font style="color:#000000;">需要准备一套k8s的集群，helm主要是k8s集群的包管理器，主要用来管理helm中的各种chart包。</font>

# 30. <font style="color:#000000;">2 传统服务部署到k8s集群的流程</font>
1. <font style="color:#000000;">拉取代码</font>
2. <font style="color:#000000;">打包编译</font>
3. <font style="color:#000000;">构建镜像</font>
4. <font style="color:#000000;">准备一堆相关部署的yaml文件(如:deployment、service、ingress等)</font>
5. <font style="color:#000000;">kubectl apply 部署到k8s集群</font>

<font style="color:#000000;">传统方式部署引发的问题</font>

1. <font style="color:#000000;">随着引用的增多，需要维护大量的yaml文件</font>
2. <font style="color:#000000;">不能根据一套yaml文件来创建多个环境，需要手动进行修改。  
</font><font style="color:#000000;">例如：一般环境都分为dev、预生产、生产环境，部署完了dev这套环境，后面再部署预生产和生产环境，还需要复制出两套，并手动修改才行。</font>

# 31. <font style="color:#000000;">3 什么是helm</font>
<font style="color:#000000;">Helm是Kubernetes 的包管理工具，可以方便地发现、共享和构建 Kubernetes 应用</font>

<font style="color:#000000;">helm是</font><font style="color:#000000;">k8s的包管理器</font><font style="color:#000000;">，</font><font style="color:#000000;">相当于centos系统中的yum工具</font><font style="color:#000000;">，可以</font><font style="color:#000000;">将一个服务相关的所有资源信息整合到一个chart包中</font><font style="color:#000000;">，并且可以使</font><font style="color:#000000;">用一套资源发布到多个环境中</font><font style="color:#000000;">， 可以将应用程序的所有资源和部署信息组合到单个部署包中。  
</font><font style="color:#000000;">就像Linux下的rpm包管理器，如yum/apt等，可以很方便的将之前打包好的yaml文件部署到kubernetes上。</font>

# 32. <font style="color:#000000;">4 helm的组件</font>
+ <font style="color:#000000;">Chart：就是helm的</font><font style="color:#000000;">一个整合后的chart包</font><font style="color:#000000;">，包含一个应用所有的kubernetes声明模版，类似于yum的rpm包或者apt的dpkg文件。</font>
    - <font style="color:#000000;">helm</font><font style="color:#000000;">将打包的应用程序部署到k8s</font><font style="color:#000000;">，并将它们构建成Chart。这些Chart将</font><font style="color:#000000;">所有预配置的应用程序资源</font><font style="color:#000000;">以及所有版本都</font><font style="color:#000000;">包含在一个易于管理的包中</font><font style="color:#000000;">。</font>
    - <font style="color:#000000;">Helm把</font><font style="color:#000000;">kubernetes资源(如deployments、services或ingress等) 打包到一个chart中,chart被保存到chart仓库</font><font style="color:#000000;">。通过chart仓库可用来存储和分享chart</font>
+ <font style="color:#000000;">Helm客户端：helm的客户端组件，负责和k8s apiserver通信</font>
+ <font style="color:#000000;">Repository：用于发布和存储chart包的仓库，类似yum仓库或docker仓库</font>
+ <font style="color:#000000;">Release：用chart包</font><font style="color:#000000;">部署的一个实例</font><font style="color:#000000;">。通过chart在k8s中部署的应用都会产生一个唯一的Release. 同一chart部署多次就会产生多个Release.  
</font><font style="color:#000000;">理解：将这些yaml部署完成后，</font><font style="color:#000000;">他也会记录部署时候的一个版本，维护了一个release版本状态</font><font style="color:#000000;">，通过Release这个实例，他会具体帮我们创建pod，deployment等资源</font>

# 33. <font style="color:#000000;">5 helm3和helm2的区别</font>
+ <font style="color:#000000;">helm3移除了Tiller组件 helm2中helm客户端和k8s通信是通过Tiller组件和k8s通信,helm3移除了Tiller组件,直接使用kubeconfig文件和k8s的apiserver通信</font>
+ <font style="color:#000000;">删除 release 命令变更</font>

```yaml
helm delete release-name --purge
helm uninstall release-name 
```

+ <font style="color:#000000;">查看 charts 信息命令变更</font>

```yaml
helm inspect release-name
helm show release-name
```

+ <font style="color:#000000;">拉取 charts 包命令变更</font>

```yaml
helm fetch chart-name
helm pull chart-name
```

+ <font style="color:#000000;">helm3中必须指定 release 名称，如果需要生成一个随机名称，需要加选项--generate-name，helm2中如果不指定release名称，可以自动生成一个随机名称</font>

```yaml
helm install ./mychart --generate-name
```

# 34. <font style="color:#000000;">6 </font><font style="color:#000000;">二进制方式安装helm3</font>
1. <font style="color:#000000;">查看k8s集群</font>

```yaml
kubectl get nodes
```

2. <font style="color:#000000;">下载helm依赖包</font>

```yaml
wget https://get.helm.sh/helm-v3.5.2-linux-amd64.tar.gz 
```

3. <font style="color:#000000;">解压</font>

```yaml
tar -zxf helm-v3.5.2-linux-amd64.tar.gz
```

4. <font style="color:#000000;">安装</font>

```yaml
mv linux-amd64/helm /usr/local/bin/
```

5. <font style="color:#000000;">查看</font>

```yaml
helm version
```

# 35. <font style="color:#000000;">7 </font><font style="color:#000000;">chart的目录结构</font>
<font style="color:#000000;">创建一个chart，指定chart名：mychart</font>

```yaml
helm create mychart
```

<font style="color:#000000;">查看chart的目录结构</font>

```yaml
tree mychart/
```

<font style="color:#000000;">如果没有 tree 命令</font>

```yaml
brew install tree  # mac 执行此命令安装即可
```

```yaml
mychart/ #chart包的名称
├── charts #存放子chart的目录，目录里存放这个chart依赖的所有子chart
├── Chart.yaml #保存chart的基本信息，包括名字、描述信息及版本等，这个变量文件都可以被templates目录下文件所引用
├── templates #模板文件目录，目录里面存放所有yaml模板文件，包含了所有部署应用的yaml文件
│   ├── deployment.yaml #创建deployment对象的模板文件
│   ├── _helpers.tpl #放置模板助手的文件，可以在整个chart中重复使用，是放一些templates目录下这些yaml都有可能会用的一些模板
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt #存放提示信息的文件,介绍chart帮助信息,helm install部署后展示给用户.如何使用chart等,是部署chart后给用户的提示信息
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests #用于测试的文件，测试完部署完chart后，如web，做一个链接，看看你是否部署正常
│   └──── test-connection.yaml
└── values.yaml #用于渲染模板的文件（变量文件，定义变量的值） 定义templates目录下的yaml文件可能引用到的变量
#values.yaml用于存储 templates 目录中模板文件中用到变量的值，这些变量定义都是为了让templates目录下yaml引用
```

![](images/1.png)<font style="color:#000000;">  
  
</font><font style="color:#000000;">Chart.yaml 文件字段说明</font>

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for Kubernetes

# 36. A chart can be either an 'application' or a 'library' chart.
#
# 37. Application charts are a collection of templates that can be packaged into versioned archives
# 38. to be deployed.
#
# 39. Library charts provide useful utilities or functions for the chart developer. They're included as
# 40. a dependency of application charts to inject those utilities and functions into the rendering
# 41. pipeline. Library charts do not define any templates and therefore cannot be deployed.
type: application

# 42. This is the chart version. This version number should be incremented each time you make changes
# 43. to the chart and its templates, including the app version.
# 44. Versions are expected to follow Semantic Versioning (https://semver.org/)
version: 0.1.0

# 45. This is the version number of the application being deployed. This version number should be
# 46. incremented each time you make changes to the application. Versions are not expected to
# 47. follow Semantic Versioning. They should reflect the version the application is using.
# 48. It is recommended to use it with quotes.
appVersion: "1.16.0"
```

<font style="color:#000000;">详细字段说明：</font>

+ <font style="color:#000000;">apiVersion：chart API 版本信息, 通常是 "v2" (必须) </font>
    - <font style="color:#000000;">appVersion字段与version 字段并没有直接的联系。这是指定应用版本的一种方式</font>
    - <font style="color:#000000;">对于部分仅支持helm3的chart，apiVersion应该指定为v2。对于可以同时支持helm3和helm2版本的chart,可以将其设置为v1</font>
+ <font style="color:#000000;">name：chart 的名称 (必须)</font>
+ <font style="color:#000000;">version：chart 包的版本 (必须)</font><font style="color:#000000;">，</font>
    - <font style="color:#000000;">version 字段用于指定 chart 包的版本号，后续更新 chart 包时候也需要同步更新这个字段</font>
+ <font style="color:#000000;">kubeVersion：指定kubernetes 版本(可选) 用于指定受支持的kubernetes版本,helm在安装时候会验证版本信息。</font>
+ <font style="color:#000000;">type：chart 类型（可选）， v2版新增了type 字段，用于区分chart类型,取值为application（默认）和library，如【应用类型chart】和【库类型chart】</font>
+ <font style="color:#000000;">description：对项目的描述 (可选)  
</font>

