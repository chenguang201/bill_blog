前提

1. 准备好K8S集群
2. 创建好Spring Boot工程



# 1 将K8S中的configmap的值读取到Java对象中
参考官方文档

[Spring Cloud Kubernetes](https://docs.spring.io/spring-cloud-kubernetes/docs/current/reference/html/)

操作步骤

1. 加入POM依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-parent</artifactId>
        <version>2.6.6</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>com.cg</groupId>
    demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>
    <description>Demo project for Spring Boot</description>
    <properties>
        <java.version>8</java.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            spring-cloud-starter-kubernetes-client-all</artifactId>
            <version>2.1.3</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>

```

2. 编写YAML

```yaml
spring:
  application:
    name: spring-cloud-kubenetes-test
  cloud:
    kubernetes:
      config:
        # 是否启用kubernetes configmap 配置
        enabled: true
        name: spring-cloud-kubenetes-test
        namespace: test
```

```yaml

```

3. 编写启动类
4. 编写业务类
5. 打包镜像
6. 编写K8S的Pod
7. 编写K8S的configmap
8. 测试

