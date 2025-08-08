文章来自：

[四十五图，一万五千字！一文让你走出迷雾玩转Maven！ - 掘金](https://juejin.cn/post/7238823745828405308?utm_source=gold_browser_extension)

上面所说到的一些知识，仅仅只是 Maven 的基本操作，而它作为 Java 项目管理占有率最高的工具，还提供了一系列高阶功能，例如属性管理、多模块开发、聚合工程等，不过这里先来说说依赖冲突。

## 2.1 依赖冲突
依赖冲突是指：在Maven项目中，当多个依赖包，引入了同一份类库的不同版本时，可能会导致编译错误或运行时异常。这种情况下，想要解决依赖冲突，可以靠升级/降级某些依赖项的版本，从而让不同依赖引入的同一类库，保持一致的版本号。

另外，还可以通过<font style="color:#DF2A3F;">隐藏依赖</font>、或者<font style="color:#DF2A3F;">排除特定的依赖项</font>来解决问题。但是想搞明白这些，首先得理解Maven中的依赖传递性，一起来看看。

### 2.1.1 依赖的传递性
先来看个例子：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599016232-51d539e6-9326-40e0-baaf-261a6f70dd66.webp)

目前的工程中，仅导入了一个`spring-web`依赖，可是从下面的依赖树来看，`web`还间接依赖于`beans``core`包。而`core`包又依赖于`jcl`包，此时就出现了依赖传递，所谓的依赖传递是指：当引入的一个包，如果依赖于其他包（类库），当前的工程就必须再把其他包引入进来。

这相当于无限套娃，而这类“套娃”引入的包，被称为间接性依赖。与之对应的是直接性依赖，即：当前工程的`pom.xml`中，直接通过 GAV 坐标引入的包。既然如此，那么一个工程内的依赖包，就必然会出现层级，如：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599016737-142d6638-081a-4a9f-aa0a-63e06583d0cc.webp)

在这里我们仅引入了一个`spring-boot-starter-test`坐标，但当打开依赖树时，会发现这一个包，依赖于其他许多包，而它所依赖的包又依赖于其他包……，如此不断套娃，最深套到了五层。而不同的包，根据自己所处的层级不同，会被划分为1、2、3、4……级。

### 2.1.2 自动解决冲突问题
Maven 作为 Apache 旗下的产品，而且还经过这么多个版本迭代，对于依赖冲突问题，难道官方想不到吗？

必然想到了，所以在绝对大多数情况下，依赖冲突问题并不需要我们考虑，Maven工具会自动解决，怎么解决的呢？就是基于前面所说的依赖层级，下面来详细说说。

**<font style="color:#DF2A3F;">① 层级优先原则</font>**：Maven会根据依赖树的层级，来自动剔除相同的包，层级越浅，优先级越高。这是啥意思呢？同样来看个例子：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599017849-70636f0e-1adf-46ff-97a0-682b4865fbb9.webp)

我们又通过 GAV 坐标导入了`spring-web`包，根据前面所说，`web`依赖于`beans``core`。而`beans`包又依赖于`core`包，此时注意，这里出现了两个`core`包，前者的层级为 2 ，后者的层级为 3 ，所以 Maven 会自动将后者剔除，这点从图中也可明显看出，层级为 3 的`core`直接变灰了。

**<font style="color:#DF2A3F;">② 声明优先原则</font>**：上条原则是基于层级深度，来自动剔除冲突的依赖，那假设同级出现两个相同的依赖怎么办？来看例子：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599020866-a69d980e-c218-40db-bbc4-8d2f653198bb.webp)

此时用 GAV 引入了`web``jdbc`两个包，来看右边的依赖树，`web`依赖于`beans``core`包。`jdbc`也依赖于这两个包，此时相同层级出现了依赖冲突，可从结果上来看，后面`jdbc`所依赖的两个包被剔除了，能明显看到一句：omitted for duplicate（重复时省略），这又是为啥呢？因为根据声明优先原则，同层级出现包冲突时，先声明的会覆盖后声明的，为此后者会被剔除。

**<font style="color:#DF2A3F;">③配置优先原则</font>**：此时问题又又来了，既然相同层级出现同版本的类库，前面的会覆盖后面的，可是当相同层级，出现不同版本的包呢？依旧来看例子：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599022159-f3988d39-c61a-408b-bb57-d76bf997982b.webp)

此时 pom 引入了两个`web`包，前者版本为`5.1.8`，后者为`5.1.2`，这两个包的层级都是 1，可是看右边的依赖树，此时会发现，`5.1.8`压根没引进来啊！为啥？这就是配置优先原则，同级出现不同版本的相同类库时，后配置的会覆盖先配置的。

所以大家发现了嘛？在很多时候，并不需要我们考虑依赖冲突问题，Maven会依据上述三条原则，帮我们智能化自动剔除冲突的依赖，其他包都会共享留下来的类库，只有当出现无法解决的冲突时，这才需要咱们手动介入。

通常来说，Maven如果无法自动解决冲突问题，会在构建过程中抛出异常并提供相关信息，这时大家可以根据给出的信息，手动排除指定依赖。

### 2.1.3 主动排除依赖
所谓的排除依赖，即是指从一个依赖包中，排除掉它依赖的其他包，如果出现了 Maven 无法自动解决的冲突，就可以基于这种手段进行处理，例如：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    spring-web</artifactId>
    <version>5.1.8.RELEASE</version>
    <exclusions>
        <!-- 排除web包依赖的beans包 -->
        <exclusion>
            <groupId>org.springframework</groupId>
            spring-beans</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599023108-96d73cc7-1de4-4098-a5a9-1145627285e8.webp)

从图中结果可以明显看出，通过这种方式，可以手动移除包所依赖的其他包。当出现冲突时，通过这种方式将冲突的两个包，移除掉其中一个即可。

其实还有种叫做“隐藏依赖”的手段，不过这种手段是用于多工程聚合项目，所以先讲清楚“多模块/工程”项目，接着再讲“隐藏依赖”。

## 2.2 Maven分模块开发
现如今，一个稍具规模的完整项目，通常都要考虑接入多端，如 PC、WEB、APP 端等，那此时问题来了，每个端之间的逻辑，多少会存在细微差异，如果将所有代码融入在一个 Maven 工程里，这无疑会显得十分臃肿！为了解决这个问题，Maven 推出了分模块开发技术。

所谓的分模块开发，即是指创建多个 Maven 工程，组成一个完整项目。通常会先按某个维度划分出多个模块，接着为每个模块创建一个 Maven 工程，典型的拆分维度有三个：

+ **① 接入维度**：按不同的接入端，将项目划分为多个模块，如APP、WEB、小程序等；
+ **② 业务维度**：根据业务性质，将项目划分为一个个业务模块，如前台、后台、用户等；
+ **③ 功能维度**：共用代码做成基础模块，业务做成一个模块、API做成一个模块……。

当然，通常 ①、② 会和 ③ 混合起来用，比如典型的“先根据代码功能拆分，再根据业务维度拆分”。



相较于把所有代码揉在一起的“大锅饭”，多模块开发的好处特别明显：

+ ① 简化项目管理，拆成多个模块/子系统后，每个模块可以独立编译、打包、发布等；
+ ② 提高代码复用性，不同模块间可以相互引用，可以建立公共模块，减少代码冗余度；
+ ③ 方便团队协作，多人各司其职，负责不同的模块，Git 管理时也能减少交叉冲突；
+ ④ 构建管理度更高，更方便做持续集成，可以根据需要灵活配置整个项目的构建流程；
+ ……



不过Maven 2.0.9 才开始支持聚合工程，在最初的时期里，想要实现分模块开发，需要手动先建立一个空的Java项目（Empty Project）：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599023155-1980d50c-e524-48a4-88dc-ee616f1994bf.webp)

接着再在其中建立多个Maven Project：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599024241-2664cc6e-165c-4e2e-a50d-030fe638b3dd.webp)

然后再通过`mvn install`命令，将不同的 Maven 项目安装到本地仓库，其他工程才能通过 GAV 坐标引入。

这种传统方式特别吃力，尤其是多人开发时，另一个模块的代码更新了，必须手动去更新本地仓库的 jar 包；而且多个模块之间相互依赖时，构建起来额外的麻烦！正因如此，官方在后面推出了“聚合工程”，下面聊聊这个。

## 2.3 Maven聚合工程
所谓的聚合工程，即是指：一个项目允许创建多个子模块，多个子模块组成一个整体，可以统一进行项目的构建。不过想要弄明白聚合工程，得先清楚“父子工程”的概念：

+ 父工程：不具备任何代码、仅有`pom.xml`的空项目，用来定义公共依赖、插件和配置；
+ 子工程：编写具体代码的子项目，可以继承父工程的配置、依赖项，还可以独立拓展。

而 Maven 聚合工程，就是基于父子工程结构，来将一个完整项目，划分出不同的层次，这种方式可以很好的管理多模块之间的依赖关系，以及构建顺序，大大提高了开发效率、维护性。并且当一个子工程更新时，聚合工程可以保障同步更新其他存在关联的子工程！

### 2.3.1 聚合工程入门指南
理解聚合工程是个什么东东之后，接着来聊聊如何创建聚合工程，首先要创建一个空的 Maven项目，作为父工程，这时可以在 IDEA 创建 Maven 项目时，把打包方式选成 POM，也可以创建一个普通的 Maven 项目，然后把`src`目录删掉，再修改一下`pom.xml`：

```xml
<!-- 写在当前项目GAV坐标下面 -->
<packaging>pom</packaging>
```

这样就得到了一个父工程，接着可以在此基础上，继续创建子工程：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599024939-434e4a18-6acf-4e5f-bd5b-a0a50f81f41e.webp)

当点击Next后，大家会发现：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599024718-5f23e168-d1a1-4494-ab23-adfbf3e39eb5.webp)

这时无法手动指定G、V了，而是会从父工程中继承，最终效果如下：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599030451-6286e5bc-cf67-4ecf-91f9-e81bde2fe09a.webp)

这里我创建了两个子工程，所以父工程的`pom.xml`中，会用一个`<modules>`标签，来记录自己名下的子工程列表，而子工程的`pom`头，也多了一个`<parent>`标签包裹！大家看这个标签有没有眼熟感？大家可以去看一下 SpringBoot 项目，每个`pom.xml`文件的头，都是这样的。

这里提个问题：子工程下面能不能继续创建子工程？答案 Yes，你可以无限套娃下去，不过我的建议是：一个聚合项目，最多只能有三层，路径太深反而会出现稀奇古怪的问题。

### 2.3.2 聚合工程的依赖管理
前面搭建好了聚合工程，接着来看个问题：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599030948-11c23220-7620-4fe0-ad42-80cc8babf282.webp)

`zhuzi_001``002`两个子工程中，各自引入了三个依赖，可观察上图会发现，两者引入的依赖仅有一个不同，其余全部一模一样！所以这时，就出现了“依赖冗余”问题，那有没有好的方式解决呢？答案是有的，前面说过：<font style="color:#DF2A3F;">公共的依赖、配置、插件等，都可以配置在父工程里</font>，如下：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599030917-12131165-fa1e-452f-8fd6-a0e32134a3aa.webp)

当把公共的依赖定义在父工程中，此时观察图中右侧的依赖树，会发现两个子工程都继承了父依赖。

不过此时问题又来了！为了防止不同子工程引入不同版本的依赖，最好的做法是在父工程中，统一对依赖的版本进行控制，规定所有子工程都使用同一版本的依赖，怎么做到这点呢？可以使用`<dependencyManagement>`标签来管理，例如：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599036991-c5e314c2-b915-4591-ba0c-27e2fe3af5d7.webp)

在父工程中，`<dependencies>`里只定义了一个`webmvc`依赖，而`<dependencyManagement>`中定义了`druid``test``jdbc`三个依赖，这两个标签有何区别呢？

+ `<dependencies>`：定义强制性依赖，写在该标签里的依赖项，<font style="color:#DF2A3F;">子工程必须强制继承</font>；
+ `<dependencyManagement>`：定义可选性依赖，该标签里的依赖项，<font style="color:#DF2A3F;">子工程可选择使用</font>。

相信这样解释后，大家对于两个标签的区别，就能一清二楚了！同时注意，子工程在使用`<dependencyManagement>`中已有的依赖项时，不需要写`<version>`版本号，版本号在父工程中统一管理，这就满足了前面的需求。这样做的好处在于：以后为项目的技术栈升级版本时，不需要单独修改每个子工程的 POM，只需要修改父 POM 文件即可，大大提高了维护性！

### 2.3.3 聚合工程解决依赖冲突
之前传统的 Maven 项目会存在依赖冲突问题，那聚合工程中存不存在呢？当然存在，比如 `001`中引入了`jdbc``test`这两个包，而 `002` 中也引入了，这时假设把 `001` 工程打包到本地仓库，在 `002` 工程中引入时，此时依赖是不是又冲突了？Yes，怎么处理呢？先看例子：

在下图中，`001` 引入了`aop`包，接着通过`install`操作，把 `001` 工程打到了本地仓库。于是，在 `002` 工程中，引入了`web``zhuzi_001`这两个包。根据前面所说的依赖传递原则，`002` 在引入 `001` 时，由于 `001` 引用了别的包，所以 `002` 被迫也引入了其他包。

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599039776-e4cd9f88-3974-4eea-94a4-6e44bcb449a0.webp)

还是那句话，大多数情况下，Maven会基于那三条原则，自动帮你剔除重复的依赖，如上图右边的依赖树所示，Maven 自动剔除了重复依赖。这种结果显然是好现象，可是万一Maven不能自动剔除怎么办？这时就需要用到最开始所说的“隐藏依赖”技术了！

修改`001`的`pom.xml`，如下：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    spring-aop</artifactId>
    <version>5.1.8.RELEASE</version>
    <optional>true</optional>
</dependency>
```

眼尖的小伙应该能发现，此时多了一个`<optional>`标签，该标签即是“隐藏依赖”的开关：

+ true：开启隐藏，<font style="color:#DF2A3F;">当前依赖不会向其他工程传递，只保留给自己用</font>；
+ false：默认值，表示当前依赖会保持传递性，其他引入当前工程的项目会间接依赖。

此时重新把001打到本地仓库，再来看看依赖树关系：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599041731-aeeb7107-8eb8-4e7e-9e2c-25be55bc874d.webp)

当开启隐藏后，其他工程引入当前工程时，就不会再间接引入当前工程的隐藏依赖，因此来手动排除聚合工程中的依赖冲突问题。其他许多资料里，讲这块时，多少讲的有点令人迷糊，而相信看到这里，大家就一定理解了Maven依赖管理。

### 2.3.4 父工程的依赖传递
来思考一个问题，现在项目需要用到 Spring-Cloud-Alibaba 的多个依赖项，如`Nacos``Sentinel`……等，根据前面所说的原则，由于这些依赖项可能会在多个子工程用到，最好的方式是定义在父 POM 的`<dependencyManagement>`标签里，可是 CloudAlibaba 依赖这么多，一个个写未免太繁杂、冗余了吧？

面对上述问题时，该如何处理呢？如下：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    spring-cloud-alibaba-dependencies</artifactId>
    <version>${spring-cloud-alibaba.version}</version>
    <type>pom</type>
    <scope>import</scope>
</dependency>
```

`<scope>`标签取值为`import`的方式，通常会用在聚合工程的父工程中，不过必须配合`<type>pom</type>`使用，这是啥意思呢？这代表着：把`spring-cloud-alibaba-dependencies`的所有子依赖，作为当前项目的可选依赖向下传递。

而当前父工程下的所有子工程，在继承父 POM 时，也会将这些可选依赖继承过来，这时就可以直接选择使用某些依赖项啦，如：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

### 2.3.5 聚合工程的构建
前面说到过，Maven聚合工程可以对所有子工程进行统一构建，这是啥意思呢？如果是传统的分模块项目，需要挨个进行打包、测试、安装……等工作，而聚合工程则不同，来看IDEA提供的Maven辅助工具：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599044042-d425b8d1-0f2a-461d-a8f8-495200f32417.webp)

尾巴上带有`root`标识的工程，意味着这是一个父工程，在我们的案例中，有一个父、两个子，来看IDEA的工具，除开给两个子工程提供了 Lifecycle 命令外，还给父工程提供了一套Lifecycle 命令，这两者的区别在哪儿呢？当你双击父工程的某个 Lifecycle 命令，它找到父 POM 的`<modules>`标签，再根据其中的子工程列表，完成对整个聚合工程的构建工作。

大家可以去试一下，当你双击父工程 Lifecycle 下的`clean`，它会把你所有子工程的`target`目录删除。同理，执行其他命令时也一样，比如`install`命令，双击后它会把你所有的子工程，打包并安装到本地仓库，不过问题又又又来了！

假设这里`001`引用了`002`，`002`又引用了`001`，两者相互引用，Maven 会如何构建啊？到底该先打包`001`，还是该先打包`002`？我没去看过 Lifecycle 插件的源码，不过相信背后的逻辑，应该跟 Spring 解决依赖循环类似，感兴趣的小伙伴可以自行去研究。不过我这里声明一点：Maven 聚合工程的构建流程，跟`<modules>`标签里的书写顺序无关，它会自行去推断依赖关系，从而完成整个项目的构建。

### 2.3.6 聚合打包跳过测试
当大家要做项目发版时，就需要对整个聚合工程的每个工程打包（`jar`或`war`包），此时可以直接双击父工程里的`package`命令，但`test`命令在`package`之前，按照之前聊的生命周期原则，就会先执行`test`，再进行打包。

`test`阶段，会去找到所有子工程的`src/test/java`目录，并执行里面的测试用例，如果其中任何一个报错，就无法完成打包工作。而且就算不报错，执行所有测试用例也会特别耗时，这时该怎么办呢？可以选择跳过`test`阶段，在IDEA工具里的操作如下：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599045151-cc5fd9a8-594f-4b05-af2d-87e37707aaed.webp)

先选中`test`命令，接着点击上面的闪电图标，这时test就会画上横线，表示该阶段会跳过。如果你是在用`mvn`命令，那么打包跳过测试的命令如下：

```bash
mvn package –D skipTests
```

同时大家还可以在`pom.xml`里，配置插件来精准控制，比如跳过某个测试类不执行，配置规则如下：

```xml
<build>
    <plugins>
        <plugin>
            maven-surefire-plugin</artifactId>
            <version>2.22.1</version>
            <configuration>
                <skipTests>true</skipTests>
                <includes>
                    <!-- 指定要执行的测试用例 -->
                    <include>**/XXX*Test.java</include>
                </includes>
                <excludes>
                    <!-- 执行要跳过的测试用例 -->
                    <exclude>**/XXX*Test.java</exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

不过这个功能有点鸡肋，了解即可，通常不需要用到。

## 2.4 Maven属性
回到之前案例的父工程 POM 中，此时来思考一个问题：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599046312-446686d6-dd6c-45e0-ae0a-2dba92e5cfa8.webp)

虽然我们通过`<dependencyManagement>`标签，来控制了子工程中的依赖版本，可目前还有一个小问题：版本冗余！比如现在我想把 Spring 版本从`5.1.8`升级到`5.2.0`，虽然不需要去修改子工程的POM文件，可从上图中大家会发现，想升级 Spring 的版本，还需要修改多处地方！

咋办？总不能只升级其中一个依赖的版本吧？可如果全部都改一遍，无疑就太累了……，所以，这里我们可以通过 Maven 属性来做管理，我们可以在POM的`<properties>`标签中，自定义属性，如：

```xml
<properties>
    <spring.version>5.2.0.RELEASE</spring.version>
</properties>
```

而在POM的其他位置中，可以通过${}来引用该属性，例如：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599049969-8c8f531e-fd37-4987-95bd-c15f50650099.webp)

这样做的好处特别明显，现在我想升级Spring版本，只需要修改一处地方即可！

除开可以自定义属性外，Maven也会有很多内置属性，大体可分为四类：

| 类型 | 使用方式 |
| --- | --- |
| Maven内置属性 | 

$$

$$

{属性名}，如
$$

$$

{version} |
| 项目环境属性 | 

$$

$$

{setting.属性名}，如
$$

$$

{settings.localRepository} |
| Java环境变量 | 

$$

$$

{xxx.属性名}，如
$$

$$

{java.class.path} |
| 系统环境变量 | 

$$

$$

{env.属性名}，如
$$

$$

{env.USERNAME} |


不过这些用的也不多，同时不需要记，要用的时候，IDEA工具会有提示：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599050052-1aba22cf-12c9-418f-a0a9-0ce1651aa9bf.webp)

## 2.5 Maven多环境配置
实际工作会分为开发、测试、生产等环境，不同环境的配置信息也略有不同，而大家都知道，我们可以通过`spring.profiles.active`属性，来动态使用不同环境的配置，而Maven为何又整出一个多环境配置出来呢？想要搞清楚，得先搭建一个 SpringBoot 版的 Maven 聚合工程。

首先创建一个只有 POM 的父工程，但要注意，这里是 SpringBoot 版聚合项目，需稍微改造：

```xml
<!-- 先把Spring Boot Starter声明为父工程 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    spring-boot-starter-parent</artifactId>
    <version>2.1.5.RELEASE</version>
    <relativePath/>
</parent>

<!-- 当前父工程的GAV坐标 -->
<modelVersion>4.0.0</modelVersion>
<groupId>com.zhuzi</groupId>
maven_zhuzi</artifactId>
<version>1.0-SNAPSHOT</version>
<packaging>pom</packaging>

<!-- 配置JDK版本 -->
<properties>
    <java.version>8</java.version>
</properties>

<dependencies>
    <!-- 引入SpringBootWeb的Starter依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<build>
<plugins>
    <!-- 引入SpringBoot整合Maven的插件 -->
    <plugin>
        <groupId>org.springframework.boot</groupId>
        spring-boot-maven-plugin</artifactId>
    </plugin>
</plugins>
</build>
```

对比普通聚合工程的父 POM 来说，SpringBoot 版的聚合工程，需要先把`spring-boot-starter`声明成自己的“爹”，同时需要引入 SpringBoot 相关的插件，并且我在这里还引入了一个`spring-boot-starter-web`依赖。

接着来创建子工程，在创建时记得选 SpringBoot 模板来创建，不过创建后记得改造POM：

```xml
<!-- 声明父工程 -->
<parent>
    maven_zhuzi</artifactId>
    <groupId>com.zhuzi</groupId>
    <version>1.0-SNAPSHOT</version>
</parent>
<modelVersion>4.0.0</modelVersion>

<!-- 子工程的描述信息 -->
boot_zhuzi_001</artifactId>
<name>boot_zhuzi_001</name>
<description>Demo project for Spring Boot</description>
```

就只需要这么多，因为 SpringBoot 的插件、依赖包，在父工程中已经声明了，这里会继承过来。

接着来做Maven多环境配置，找到父工程的 POM 进行修改，如下：

```xml
<profiles>
    <!-- 开发环境 -->
    <profile>
        <id>dev</id>
        <properties>
            <profile.active>dev</profile.active>
        </properties>
    </profile>
    
    <!-- 生产环境 -->
    <profile>
        <id>prod</id>
        <properties>
            <profile.active>prod</profile.active>
        </properties>
        <!-- activeByDefault=true，表示打包时，默认使用这个环境 -->
        
            true</activeByDefault>
        </activation>
    </profile>
    
    <!-- 测试环境 -->
    <profile>
        <id>test</id>
        <properties>
            <profile.active>test</profile.active>
        </properties>
    </profile>
</profiles>
```

配置完这个后，刷新当前Maven工程，IDEA中就会出现这个：

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599054085-bd5705e5-4959-4ce0-afbf-581e1deb93d7.webp)

默认停留在prod上，这是因为POM中用``标签指定了，接着去到子工程的`application.yml`中，完成 Spring 的多环境配置，如下：

```yaml
# 187. 设置启用的环境
spring:
  profiles:
    active: ${profile.active}

---
# 188. 开发环境
spring:
  profiles: dev
server:
  port: 80
---
# 189. 生产环境
spring:
  profiles: prod
server:
  port: 81
---
# 190. 测试环境
spring:
  profiles: test
server:
  port: 82
---
```

这里可以通过文件来区分不同环境的配置信息，但我这里为了简单，就直接用`---`进行区分，这组配置大家应该很熟悉，也就是不同的环境中，使用不同的端口号，但唯一不同的是：以前`spring.profiles.active`属性会写上固定的值，而现在写的是`${profile.active}`，这是为什么呢？

这代表从`pom.xml`中，读取`profile.active`属性值的意思，而父POM中配了三组值：`dev``prod``test`，所以当前子工程的 POM，也会继承这组配置，而目前默认勾选在`prod`上，所以最终`spring.profiles.active=prod`，不过想要在`application.yml`读到`pom.xml`的值，还需在父POM中，加一个依赖和插件：

```xml
<!-- 开启 yml 文件的 ${} 取值支持 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    spring-boot-configuration-processor</artifactId>
    <version>2.1.5.RELEASE</version>
    <optional>true</optional>
</dependency>

<!-- 添加插件，将项目的资源文件复制到输出目录中 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    maven-resources-plugin</artifactId>
    <version>3.2.0</version>
    <configuration>
        <encoding>UTF-8</encoding>
        <useDefaultDelimiters>true</useDefaultDelimiters>
    </configuration>
</plugin>
```

最后来尝试启动子工程，操作流程如下：

+ ①在Maven工具的`Profiles`中勾选`dev`，并刷新当前项目；
+ ②接着找到子工程的启动类，并右键选择 Run ……启动子项目。

流程图如下

![](https://cdn.nlark.com/yuque/0/2023/webp/22334924/1703599055174-da7c890f-42be-4e3f-9c13-1c303a2fbf3f.webp)

先仔细看执行的结果，我来解释一下执行流程：

+ ①启动时，`pom.xml`根据勾选的`Profiles`，使用相应的`dev`环境配置；
+ ②`yml`中`${profile.active}`会读到`profile.active=dev`，使用`dev`配置组；
+ ③`application.yml`中的`dev`配置组，`server.port=80`，所以最终通过 80 端口启动。



看完这个流程，大家明白最开始那个问题了吗？Maven为何还整了一个多环境配置？

大家可能有种似懂非懂的感觉，这里来说明一下，先把环境换到微服务项目中，假设有20个微服务，此时项目要上线或测试，所以需要更改配置信息，比如把数据库地址换成测试、线上地址等，而不同环境的配置，相信大家一定用`application-dev.yml``application-prod.yml`……做好了区分。



但就算提前准备了不同环境的配置，可到了切换环境时，还需要挨个服务修改`spring.profiles.active`这个值，从`dev`改成`prod``test`，然后才能使用对应的配置进行打包，可这里有 20 个微服务啊，难道要手动改 20 次吗？

而在父 POM 中配置了 Maven 多环境后，这时`yml`会读取`pom.xml`中的值，来使用不同的配置文件，此时大家就只需要在 IDEA 工具的`Profiles`中，把钩子从`dev`换到`test``prod`，然后刷新一下 Maven，SpringBoot 就能动态的切换配置文件，这是不是妙极了？因此，这才是 Maven 多环境的正确使用姿势！

