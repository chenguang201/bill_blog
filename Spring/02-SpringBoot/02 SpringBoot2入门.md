**笔记来源：**[**【尚硅谷】SpringBoot2零基础入门教程（spring boot2干货满满）**](https://www.bilibili.com/video/BV19K4y1L7MT/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

# 1 系统要求
+ Java 8
+ Maven 3.3+
+ IntelliJ IDEA 2019.1.2



Maven配置文件

新添内容：

```xml
<mirrors>
	<mirror>
		<id>nexus-aliyun</id>
		<mirrorOf>central</mirrorOf>
		<name>Nexus aliyun</name>
		<url>http://maven.aliyun.com/nexus/content/groups/public</url>
	</mirror>
</mirrors>

<profiles>
	<profile>
		<id>jdk-1.8</id>

		<activation>
			<activeByDefault>true</activeByDefault>
			<jdk>1.8</jdk>
		</activation>

		<properties>
			<maven.compiler.source>1.8</maven.compiler.source>
			<maven.compiler.target>1.8</maven.compiler.target>
			<maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
		</properties>
	</profile>
</profiles>
```



# 2 HelloWorld项目
需求：浏览发送 `/hello`  请求，响应 `Hello，Spring Boot 2`

1）创建maven工程

创建普通的 maven 项目，不用 war 项目，jar 项目即可

引入依赖

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	spring-boot-starter-parent</artifactId>
	<version>2.3.4.RELEASE</version>
</parent>

<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		spring-boot-starter-web</artifactId>
	</dependency>
</dependencies>
```

2）创建主程序

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class MainApplication {

    public static void main(String[] args) {
        SpringApplication.run(MainApplication.class, args);
    }
}
```

3）编写业务

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HelloController {
    @RequestMapping("/hello")
    public String handle01(){
        return "Hello, Spring Boot 2!";
    }
}
```

4）运行&测试

+ 运行 `MainApplication` 类
+ 浏览器输入 `http://localhost:8888/hello`，将会输出 `Hello, Spring Boot 2!`。

5）设置配置

maven工程的resource文件夹中创建application.properties文件。

```properties
# 1. 设置端口号
server.port=8888
```

[更多配置信息](https://docs.spring.io/spring-boot/docs/2.3.7.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-server)

6）打包部署

在pom.xml添加，这个插件会将项目在target下生成一个jar包

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-maven-plugin</artifactId>
		</plugin>
	</plugins>
</build>
```

在 IDEA 的 Maven 插件上点击运行`clean`  `package` 把 `helloworld` 工程项目的打包成jar包。

打包好的 jar 包被生成在 helloworld 工程项目的 target 文件夹内。

用 cmd 运行 `java -jar boot-01-helloworld-1.0-SNAPSHOT.jar` 既可以运行 helloworld 工程项目。

将jar包直接在目标服务器执行即可。

