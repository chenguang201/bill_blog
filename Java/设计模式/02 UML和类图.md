**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Java设计模式详解， 23种Java设计模式（图解+框架源码分析+实战）</font>**](https://www.bilibili.com/video/BV1Np4y1z7BU?p=2&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

# 1 UML 概念
UML：统一建模语言（Unified Modeling Language，UML）是用来设计软件的可视化建模语言。它的特点是简单、统一、图形化、能表达软件设计中的动态与静态信息。

UML 从目标系统的不同角度出发，定义了**用例图**、**类图**、**对象图**、**状态图**、**活动图**、**时序图**、**协作图**、**构件图**、**部署图**等 9 种图。

# 2 类图概述
类图（Class diagram）是显示了模型的静态结构，特别是模型中存在的类、类的内部结构以及它们与其他类的关系等。类图不显示暂时性的信息。类图是面向对象建模的主要组成部分。

[23 Markdown 类图](https://www.wenjiangs.com/doc/markdown-markdownclassdiagram)



类图的作用

+ 在软件工程中，类图是一种静态的结构图，描述了系统的类的集合，类的属性和类之间的关系，可以简化了人们对系统的理解；
+ 类图是系统分析和设计阶段的重要产物，是系统编码和测试的重要模型。



类图表示法：在UML类图中，类使用包含**类名**、**属性**（field） 和**方法**（method） 且带有分割线的矩形来表示，比如下图表示一个 Employee 类，它包含 name、age 和 address 这3个属性，以及 work() 方法。



![](https://cdn.nlark.com/yuque/__puml/b530c2605e285089209fc66c94701dd1.svg)

属性/方法名称前的`+`和`-`表示了这个属性/方法的可见性，UML类图中表示可见性的符号有三种：

+ `<font style="color:#E8323C;">+</font>`<font style="color:#E8323C;">：表示 public</font>
+ `<font style="color:#E8323C;">-</font>`<font style="color:#E8323C;">：表示 private</font>
+ `<font style="color:#E8323C;">#</font>`<font style="color:#E8323C;">：表示 protected</font>

属性的完整表示方式是： `<font style="color:#E8323C;">可见性 属性名称 ：类型 [=缺省值]</font>`。方法的完整表示方式是： `<font style="color:#E8323C;">可见性 名称(参数列表)[:返回类型]</font>`

注意：

1. 中括号中的内容表示是可选的
2. 也有将类型放在变量名前面，返回值类型放在方法名前面

举例说明：

![](https://cdn.nlark.com/yuque/__puml/e0363894ed99c6d0b06b98f0a25d7b75.svg)

上图Demo类定义了三个方法：

+ `method()`：修饰符为public，没有参数，没有返回值。
+ `method1()`：修饰符为 private，没有参数，返回值类型为 String。
+ `method2()`：修饰符为protected，接收两个参数，第一个参数类型为int，第二个参数类型为String，返回值类型是int。

# 3 类与类关系的表示方式
**关联关系是对象之间的一种引用关系**，用于表示一类对象与另一类对象之间的联系，如老师和学生、师傅和徒弟、丈夫和妻子等。关联关系是类与类之间最常用的一种关系，分为：

+ 一般关联关系、
+ 聚合关系
+ 组合关系。

## 2.1 一般关联关系
我们先介绍**一般关联**。一般关联又可以分为：

+ 单向关联
+ 双向关联
+ 自关联

**<font style="color:#E8323C;">单向关联：</font>**在UML类图中单向关联用一个**<font style="color:#DF2A3F;">带箭头的实线</font>**表示。上图表示每个顾客都有一个地址，这通过让Customer类持有一个类型为Address的成员变量类实现。

![](https://cdn.nlark.com/yuque/__puml/f6fbdd49d7fcf001ecb96a0f271c7b21.svg)

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">双向关联：</font>**从图中我们很容易看出，所谓的双向关联就是双方各自持有对方类型的成员变量。

![](https://cdn.nlark.com/yuque/__puml/422c639c609a2cd05b8bc7a804b81c7a.svg)

在UML类图中，双向关联用一个带双箭头的直线表示。上图中在 Customer 类中维护一个 List<Product>，表示一个顾客可以购买多个商品；在Product类中维护一个Customer类型的成员变量表示这个产品被哪个顾客所购买。

**<font style="color:#E8323C;"></font>**

**<font style="color:#E8323C;">自关：</font>**自关联在UML类图中用一个带有**<font style="color:#DF2A3F;">箭头且指向自身的线</font>**表示。上图的意思就是Node类包含类型为Node的成员变量，也就是“自己包含自己”。

![](https://cdn.nlark.com/yuque/__puml/502add4400eefe52af7f2851d9de9ea0.svg)

## 2.2 聚合关系
聚合关系是关联关系的一种，是强关联关系，是整体和部分之间的关系。**<font style="color:#DF2A3F;">聚合关系也是通过成员对象来实现的</font>**，其中成员对象是整体对象的一部分，但是成员对象可以脱离整体对象而独立存在。

例如，学校与老师的关系，学校包含老师，但如果学校停办了，老师依然存在。

在 UML 类图中，聚合关系可以**<font style="color:#DF2A3F;">用带空心菱形的实线</font>**来表示，菱形指向整体。下图所示是大学和教师的关系图：

![](https://cdn.nlark.com/yuque/__puml/c02820000215f1c94a4cff58d226193a.svg)

## 2.3 组合关系
组合表示类之间的整体与部分的关系，但它是一种更强烈的聚合关系。

在组合关系中，整体对象可以控制部分对象的生命周期，**<font style="color:#DF2A3F;">一旦整体对象不存在，部分对象也将不存在</font>**，部分对象不能脱离整体对象而存在。例如，头和嘴的关系，没有了头，嘴也就不存在了。  
在 UML 类图中，组合关系用**<font style="color:#DF2A3F;">带实心菱形的实线</font>**来表示，菱形指向整体。下图所示是头和嘴的关系图：

![](https://cdn.nlark.com/yuque/__puml/db4507e15bffd89b138b21ddca3c986c.svg)

## 2.4 依赖关系
依赖关系是一种使用关系，它是对象之间耦合度最弱的一种关联方式，是临时性的关联。<font style="color:#DF2A3F;">在代码中，某个类的方法通过局部变量、方法的参数或者对静态方法的调用来访问另一个类</font>（被依赖类）中的某些方法来完成一些职责。

在 UML 类图中，依赖关系使用**<font style="color:#DF2A3F;">带箭头的虚线</font>**来表示，箭头从使用类指向被依赖的类。下图所示是司机和汽车的关系图，司机驾驶汽车：

![](https://cdn.nlark.com/yuque/__puml/f451b0d4bc09f6219e4ddaa4d8e57dec.svg)

## 2.5 继承关系
继承关系是对象之间耦合度最大的一种关系，表示一般与特殊的关系，是父类与子类之间的关系，是一种继承关系。

在 UML 类图中，泛化关系用带**<font style="color:#DF2A3F;">空心三角箭头的实线</font>**来表示，箭头从子类指向父类。在代码实现时，使用面向对象的继承机制来实现泛化关系。例如，Student 类和 Teacher 类都是 Person 类的子类，其类图如下图所示：

![](https://cdn.nlark.com/yuque/__puml/734c7179df36719272c3d15484cf107f.svg)

## 2.6 实现关系
实现关系是接口与实现类之间的关系。在这种关系中，类实现了接口，类中的操作实现了接口中所声明的所有的抽象操作。在 UML 类图中，实现关系使用带**<font style="color:#DF2A3F;">空心三角箭头的虚线</font>**来表示，箭头从实现类指向接口。例如，汽车和船实现了交通工具，其类图如图 9 所示。

![](https://cdn.nlark.com/yuque/__puml/52c24aabee10d1aceeeacee26b798a74.svg)



