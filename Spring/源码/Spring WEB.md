**<font style="color:#DF2A3F;">笔记来源：</font>**[**<font style="color:#DF2A3F;">黑马程序员Spring视频教程，深度讲解spring5底层原理</font>**](https://www.bilibili.com/video/BV1P44y1N7QG/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

# 1 HandlerMapping与HandlerAdapter
主要先看<font style="color:#E8323C;">RequestMappingHandlerMapping</font>与<font style="color:#E8323C;">RequestMappingHandlerAdapter，RequestMappingHandlerMapping</font>与<font style="color:#E8323C;">RequestMappingHandlerAdapter</font>俩是一对，分别用来

+ 处理 @RequestMapping 映射
+ 调用控制器方法、并处理方法参数与方法返回值

## 1.1 DispatcherServlet初始化
代码参考

```java
package com.itheima.a20;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
import org.springframework.web.servlet.HandlerExecutionChain;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.util.Map;

public class A20 {
    private static final Logger log = LoggerFactory.getLogger(A20.class);

    public static void main(String[] args) throws Exception {
        AnnotationConfigServletWebServerApplicationContext context =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);

        // 作用 解析 @RequestMapping 以及派生注解，生成路径与控制器方法的映射关系, 在初始化时就生成
        RequestMappingHandlerMapping handlerMapping = context.getBean(RequestMappingHandlerMapping.class);

        // 获取映射结果
        Map<RequestMappingInfo, HandlerMethod> handlerMethods = handlerMapping.getHandlerMethods();
        handlerMethods.forEach((k, v) -> {
            System.out.println(k + "=" + v);
        });

        // 请求来了，获取控制器方法  返回处理器执行链对象
        MockHttpServletRequest request = new MockHttpServletRequest("GET", "/test4");
        request.setParameter("name", "张三");
        request.addHeader("token", "某个令牌");
        MockHttpServletResponse response = new MockHttpServletResponse();
        HandlerExecutionChain chain = handlerMapping.getHandler(request);
        System.out.println(chain);

        System.out.println(">>>>>>>>>>>>>>>>>>>>>");
        // HandlerAdapter 作用: 调用控制器方法
        MyRequestMappingHandlerAdapter handlerAdapter = context.getBean(MyRequestMappingHandlerAdapter.class);
        handlerAdapter.invokeHandlerMethod(request, response, (HandlerMethod) chain.getHandler());

        // 检查响应
        byte[] content = response.getContentAsByteArray();
        System.out.println(new String(content, StandardCharsets.UTF_8));

        /*System.out.println(">>>>>>>>>>>>>>>>>>>>> 所有参数解析器");
        for (HandlerMethodArgumentResolver resolver : handlerAdapter.getArgumentResolvers()) {
            System.out.println(resolver);
        }

        System.out.println(">>>>>>>>>>>>>>>>>>>>> 所有返回值解析器");
        for (HandlerMethodReturnValueHandler handler : handlerAdapter.getReturnValueHandlers()) {
            System.out.println(handler);
        }*/

        /*
        学到了什么
            a. DispatcherServlet 是在第一次被访问时执行初始化, 也可以通过配置修改为 Tomcat 启动后就初始化
            b. 在初始化时会从 Spring 容器中找一些 Web 需要的组件, 如 HandlerMapping、HandlerAdapter 等

        本章介绍两个最为重要的组件
            a. RequestMappingHandlerAdapter, 以 @RequestMapping 作为映射路径
            b. RequestMappingHandlerAdapter, 调用 handler
            c. 控制器的具体方法会被当作 handler
                - handler 的参数和返回值多种多样
                - 需要解析方法参数, 由 HandlerMethodArgumentResolver 来做
                - 需要处理方法返回值, 由 HandlerMethodReturnValueHandler 来做
        */
    }
}
```

```java
package com.itheima.a20;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.ModelAndView;
import org.yaml.snakeyaml.Yaml;

@Controller
public class Controller1 {

    private static final Logger log = LoggerFactory.getLogger(Controller1.class);

    @GetMapping("/test1")
    public ModelAndView test1() throws Exception {
        log.debug("test1()");
        return null;
    }

    @PostMapping("/test2")
    public ModelAndView test2(@RequestParam("name") String name) {
        log.debug("test2({})", name);
        return null;
    }

    @PutMapping("/test3")
    public ModelAndView test3(@Token String token) {
        log.debug("test3({})", token);
        return null;
    }

    @RequestMapping("/test4")
//    @ResponseBody
    @Yml
    public User test4() {
        log.debug("test4");
        return new User("张三", 18);
    }

    public static class User {
        private String name;
        private int age;

        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public int getAge() {
            return age;
        }

        public void setName(String name) {
            this.name = name;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }

    public static void main(String[] args) {
        String str = new Yaml().dump(new User("张三", 18));
        System.out.println(str);
    }
}
```



```java
package com.itheima.a20;

import org.springframework.boot.autoconfigure.http.HttpMessageConverters;
import org.springframework.boot.autoconfigure.web.ServerProperties;
import org.springframework.boot.autoconfigure.web.servlet.DispatcherServletRegistrationBean;
import org.springframework.boot.autoconfigure.web.servlet.WebMvcProperties;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.List;

@Configuration
@ComponentScan
@PropertySource("classpath:application.properties")
@EnableConfigurationProperties({WebMvcProperties.class, ServerProperties.class})
public class WebConfig {
    // ⬅️内嵌 web 容器工厂
    @Bean
    public TomcatServletWebServerFactory tomcatServletWebServerFactory(ServerProperties serverProperties) {
        return new TomcatServletWebServerFactory(serverProperties.getPort());
    }

    // ⬅️创建 DispatcherServlet
    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    // ⬅️注册 DispatcherServlet, Spring MVC 的入口
    @Bean
    public DispatcherServletRegistrationBean dispatcherServletRegistrationBean(
            DispatcherServlet dispatcherServlet, WebMvcProperties webMvcProperties) {
        DispatcherServletRegistrationBean registrationBean = new DispatcherServletRegistrationBean(dispatcherServlet, "/");
        registrationBean.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
        return registrationBean;
    }

    // 如果用 DispatcherServlet 初始化时默认添加的组件, 并不会作为 bean, 给测试带来困扰
    // ⬅️1. 加入RequestMappingHandlerMapping
    @Bean
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
        return new RequestMappingHandlerMapping();
    }

    // ⬅️2. 继续加入RequestMappingHandlerAdapter, 会替换掉 DispatcherServlet 默认的 4 个 HandlerAdapter
    @Bean
    public MyRequestMappingHandlerAdapter requestMappingHandlerAdapter() {
        TokenArgumentResolver tokenArgumentResolver = new TokenArgumentResolver();
        YmlReturnValueHandler ymlReturnValueHandler = new YmlReturnValueHandler();
        MyRequestMappingHandlerAdapter handlerAdapter = new MyRequestMappingHandlerAdapter();
        handlerAdapter.setCustomArgumentResolvers(List.of(tokenArgumentResolver));
        handlerAdapter.setCustomReturnValueHandlers(List.of(ymlReturnValueHandler));
        return handlerAdapter;
    }

    public HttpMessageConverters httpMessageConverters() {
        return new HttpMessageConverters();
    }

    // ⬅️3. 演示 RequestMappingHandlerAdapter 初始化后, 有哪些参数、返回值处理器

    // ⬅️3.1 创建自定义参数处理器

    // ⬅️3.2 创建自定义返回值处理器

}
```



```java
package com.itheima.a20;

import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class MyRequestMappingHandlerAdapter extends RequestMappingHandlerAdapter {
    @Override
    public ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
        return super.invokeHandlerMethod(request, response, handlerMethod);
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. `DispatcherServlet` 是在第一次被访问时执行初始化, 也可以通过配置修改为 Tomcat 启动后就初始化
2. 在初始化时会从 Spring 容器中找一些 Web 需要的组件, 如 HandlerMapping、HandlerAdapter 等，并逐一调用它们的初始化
3. `RequestMappingHandlerMapping` 初始化时，会收集所有 @RequestMapping 映射信息，封装为 Map，其中
    - key 是 `RequestMappingInfo` 类型，包括请求路径、请求方法等信息
    - value 是 `HandlerMethod` 类型，包括控制器方法对象、控制器对象
    - 有了这个 Map，就可以在请求到达时，快速完成映射，找到 HandlerMethod 并与匹配的拦截器一起返回给 DispatcherServlet
4. `RequestMappingHandlerAdapter` 初始化时，会准备 HandlerMethod 调用时需要的各个组件，如：
    - `HandlerMethodArgumentResolver` 解析控制器方法参数
    - `HandlerMethodReturnValueHandler` 处理控制器方法返回值

## 1.2 自定义参数与返回值处理器
代码参考

```java
package com.itheima.a20;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// 例如经常需要用到请求头中的 token 信息, 用下面注解来标注由哪个参数来获取它
// token=令牌
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface Token {
}
```



```java
package com.itheima.a20;

import org.springframework.core.MethodParameter;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodArgumentResolver;
import org.springframework.web.method.support.ModelAndViewContainer;

public class TokenArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    // 是否支持某个参数
    public boolean supportsParameter(MethodParameter parameter) {
        Token token = parameter.getParameterAnnotation(Token.class);
        return token != null;
    }

    @Override
    // 解析参数
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
                                  NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        return webRequest.getHeader("token");
    }
}
```



```java
package com.itheima.a20;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Yml {
}
```



```java
package com.itheima.a20;

import org.springframework.core.MethodParameter;
import org.springframework.web.context.request.NativeWebRequest;
import org.springframework.web.method.support.HandlerMethodReturnValueHandler;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.yaml.snakeyaml.Yaml;

import javax.servlet.http.HttpServletResponse;

public class YmlReturnValueHandler implements HandlerMethodReturnValueHandler {
    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        Yml yml = returnType.getMethodAnnotation(Yml.class);
        return yml != null;
    }

    @Override                   //  返回值
    public void handleReturnValue(Object returnValue, MethodParameter returnType,
                                  ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        // 1. 转换返回结果为 yaml 字符串
        String str = new Yaml().dump(returnValue);

        // 2. 将 yaml 字符串写入响应
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        response.setContentType("text/plain;charset=utf-8");
        response.getWriter().print(str);

        // 3. 设置请求已经处理完毕
        mavContainer.setRequestHandled(true);
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. 体会参数解析器的作用
2. 体会返回值处理器的作用

# 52.  2 参数解析器
**<font style="color:#F5222D;">常见参数解析器</font>**

代码参考

```java
package com.itheima.a21;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.core.MethodParameter;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockPart;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.method.annotation.ExpressionValueMethodArgumentResolver;
import org.springframework.web.method.annotation.RequestHeaderMethodArgumentResolver;
import org.springframework.web.method.annotation.RequestParamMethodArgumentResolver;
import org.springframework.web.method.support.HandlerMethodArgumentResolverComposite;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.multipart.support.StandardServletMultipartResolver;
import org.springframework.web.servlet.HandlerMapping;
import org.springframework.web.servlet.mvc.method.annotation.*;

import javax.servlet.http.Cookie;
import javax.servlet.http.HttpServletRequest;
import java.nio.charset.StandardCharsets;
import java.util.Arrays;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/*
    目标: 解析控制器方法的参数值

    常见的参数处理器如下:
        org.springframework.web.method.annotation.RequestParamMethodArgumentResolver@abbc908
        org.springframework.web.method.annotation.RequestParamMapMethodArgumentResolver@44afefd5
        org.springframework.web.servlet.mvc.method.annotation.PathVariableMethodArgumentResolver@9a7a808
        org.springframework.web.servlet.mvc.method.annotation.PathVariableMapMethodArgumentResolver@72209d93
        org.springframework.web.servlet.mvc.method.annotation.MatrixVariableMethodArgumentResolver@2687f956
        org.springframework.web.servlet.mvc.method.annotation.MatrixVariableMapMethodArgumentResolver@1ded7b14
        org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor@29be7749
        org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor@5f84abe8
        org.springframework.web.servlet.mvc.method.annotation.RequestPartMethodArgumentResolver@4650a407
        org.springframework.web.method.annotation.RequestHeaderMethodArgumentResolver@30135202
        org.springframework.web.method.annotation.RequestHeaderMapMethodArgumentResolver@6a4d7f76
        org.springframework.web.servlet.mvc.method.annotation.ServletCookieValueMethodArgumentResolver@10ec523c
        org.springframework.web.method.annotation.ExpressionValueMethodArgumentResolver@53dfacba
        org.springframework.web.servlet.mvc.method.annotation.SessionAttributeMethodArgumentResolver@79767781
        org.springframework.web.servlet.mvc.method.annotation.RequestAttributeMethodArgumentResolver@78411116
        org.springframework.web.servlet.mvc.method.annotation.ServletRequestMethodArgumentResolver@aced190
        org.springframework.web.servlet.mvc.method.annotation.ServletResponseMethodArgumentResolver@245a060f
        org.springframework.web.servlet.mvc.method.annotation.HttpEntityMethodProcessor@6edaa77a
        org.springframework.web.servlet.mvc.method.annotation.RedirectAttributesMethodArgumentResolver@1e63d216
        org.springframework.web.method.annotation.ModelMethodProcessor@62ddd21b
        org.springframework.web.method.annotation.MapMethodProcessor@16c3ca31
        org.springframework.web.method.annotation.ErrorsMethodArgumentResolver@2d195ee4
        org.springframework.web.method.annotation.SessionStatusMethodArgumentResolver@2d6aca33
        org.springframework.web.servlet.mvc.method.annotation.UriComponentsBuilderMethodArgumentResolver@21ab988f
        org.springframework.web.servlet.mvc.method.annotation.PrincipalMethodArgumentResolver@29314cc9
        org.springframework.web.method.annotation.RequestParamMethodArgumentResolver@4e38d975
        org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor@35f8a9d3
 */
public class A21 {

    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(WebConfig.class);
        DefaultListableBeanFactory beanFactory = context.getDefaultListableBeanFactory();
        // 准备测试 Request
        HttpServletRequest request = mockRequest();

        // 要点1. 控制器方法被封装为 HandlerMethod
        HandlerMethod handlerMethod = new HandlerMethod(new Controller(), Controller.class.getMethod("test", String.class, String.class, int.class, String.class, MultipartFile.class, int.class, String.class, String.class, String.class, HttpServletRequest.class, User.class, User.class, User.class));

        // 要点2. 准备对象绑定与类型转换
        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null, null);

        // 要点3. 准备 ModelAndViewContainer 用来存储中间 Model 结果
        ModelAndViewContainer container = new ModelAndViewContainer();

        // 要点4. 解析每个参数值
        for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
            // 多个解析器组合
            HandlerMethodArgumentResolverComposite composite = new HandlerMethodArgumentResolverComposite();
            composite.addResolvers(
                    //                                          false 表示必须有 @RequestParam
                    new RequestParamMethodArgumentResolver(beanFactory, false),
                    new PathVariableMethodArgumentResolver(),
                    new RequestHeaderMethodArgumentResolver(beanFactory),
                    new ServletCookieValueMethodArgumentResolver(beanFactory),
                    new ExpressionValueMethodArgumentResolver(beanFactory),
                    new ServletRequestMethodArgumentResolver(),
                    new ServletModelAttributeMethodProcessor(false), // 必须有 @ModelAttribute
                    new RequestResponseBodyMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())),
                    new ServletModelAttributeMethodProcessor(true), // 省略了 @ModelAttribute
                    new RequestParamMethodArgumentResolver(beanFactory, true) // 省略 @RequestParam
            );

            String annotations = Arrays.stream(parameter.getParameterAnnotations()).map(a -> a.annotationType().getSimpleName()).collect(Collectors.joining());
            String str = annotations.length() > 0 ? " @" + annotations + " " : " ";
            parameter.initParameterNameDiscovery(new DefaultParameterNameDiscoverer());

            if (composite.supportsParameter(parameter)) {
                // 支持此参数
                Object v = composite.resolveArgument(parameter, container, new ServletWebRequest(request), factory);
//                System.out.println(v.getClass());
                System.out.println("[" + parameter.getParameterIndex() + "] " + str + parameter.getParameterType().getSimpleName() + " " + parameter.getParameterName() + "->" + v);
                System.out.println("模型数据为：" + container.getModel());
            } else {
                System.out.println("[" + parameter.getParameterIndex() + "] " + str + parameter.getParameterType().getSimpleName() + " " + parameter.getParameterName());
            }
        }

        /*
            学到了什么
                a. 每个参数处理器能干啥
                    1) 看是否支持某种参数
                    2) 获取参数的值
                b. 组合模式在 Spring 中的体现
                c. @RequestParam, @CookieValue 等注解中的参数名、默认值, 都可以写成活的, 即从 ${ } #{ }中获取
         */
    }

    private static HttpServletRequest mockRequest() {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setParameter("name1", "zhangsan");
        request.setParameter("name2", "lisi");
        request.addPart(new MockPart("file", "abc", "hello".getBytes(StandardCharsets.UTF_8)));
        Map<String, String> map = new AntPathMatcher().extractUriTemplateVariables("/test/{id}", "/test/123");
        System.out.println(map);
        request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, map);
        request.setContentType("application/json");
        request.setCookies(new Cookie("token", "123456"));
        request.setParameter("name", "张三");
        request.setParameter("age", "18");
        request.setContent("""
                    {
                        "name":"李四",
                        "age":20
                    }
                """.getBytes(StandardCharsets.UTF_8));

        return new StandardServletMultipartResolver().resolveMultipart(request);
    }

    static class Controller {
        public void test(
                @RequestParam("name1") String name1, // name1=张三
                String name2,                        // name2=李四
                @RequestParam("age") int age,        // age=18
                @RequestParam(name = "home", defaultValue = "${JAVA_HOME}") String home1, // spring 获取数据
                @RequestParam("file") MultipartFile file, // 上传文件
                @PathVariable("id") int id,               //  /test/124   /test/{id}
                @RequestHeader("Content-Type") String header,
                @CookieValue("token") String token,
                @Value("

$$

$$

{JAVA_HOME}") String home2, // spring 获取数据  
$$

$$

{} #{}
                HttpServletRequest request,          // request, response, session ...
                @ModelAttribute("abc") User user1,   // name=zhang&age=18
                User user2,                          // name=zhang&age=18
                @RequestBody User user3              // json
        ) {
        }
    }

    static class User {
        private String name;
        private int age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "User{" +
                   "name='" + name + '\'' +
                   ", age=" + age +
                   '}';
        }
    }
}
```



```java
package com.itheima.a21;

import org.springframework.context.annotation.Configuration;

@Configuration
public class WebConfig {
}
```

**<font style="color:#E8323C;">收获</font>**

1. 初步了解 RequestMappingHandlerAdapter 的调用过程
    1. 控制器方法被封装为 HandlerMethod
    2. 准备对象绑定与类型转换
    3. 准备 ModelAndViewContainer 用来存储中间 Model 结果
    4. 解析每个参数值
2. 解析参数依赖的就是各种参数解析器，它们都有两个重要方法
    - supportsParameter 判断是否支持方法参数
    -  resolveArgument 解析方法参数
3. 常见参数的解析
    - @RequestParam
    - 省略 @RequestParam
    - @RequestParam(defaultValue)
    - MultipartFile
    - @PathVariable
    - @RequestHeader
    - @CookieValue
    - @Value
    - HttpServletRequest 等
    - @ModelAttribute
    - 省略 @ModelAttribute
    - @RequestBody
4. 组合模式在 Spring 中的体现
5. @RequestParam, @CookieValue 等注解中的参数名、默认值, 都可以写成活的, 即从 ${ } #{ }中获取

# 3 参数名解析
**<font style="color:#F5222D;">两种方法获取参数名</font>**

代码参考

```java
package com.itheima.a22;

import org.springframework.core.LocalVariableTableParameterNameDiscoverer;
import org.springframework.core.StandardReflectionParameterNameDiscoverer;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.Arrays;

/*
    目标: 如何获取方法参数名, 注意把 a22 目录添加至模块的类路径
        1. a22 不在 src 是避免 idea 自动编译它下面的类
        2. spring boot 在编译时会加 -parameters
        3. 大部分 IDE 编译时都会加 -g
 */
public class A22 {

    public static void main(String[] args) throws NoSuchMethodException, ClassNotFoundException {
        // 1. 反射获取参数名
        Method foo = Bean2.class.getMethod("foo", String.class, int.class);
       	for (Parameter parameter : foo.getParameters()) {
            System.out.println(parameter.getName());
        }

    	//2. 基于 LocalVariableTable 本地变量表
       	LocalVariableTableParameterNameDiscoverer discoverer = new LocalVariableTableParameterNameDiscoverer();
       	String[] parameterNames = discoverer.getParameterNames(foo);
       	System.out.println(Arrays.toString(parameterNames));

        /*
            学到了什么
                a. 如果编译时添加了 -parameters 可以生成参数表, 反射时就可以拿到参数名
                b. 如果编译时添加了 -g 可以生成调试信息, 但分为两种情况
                    1. 普通类, 会包含局部变量表, 用 asm 可以拿到参数名
                    2. 接口, 不会包含局部变量表, 无法获得参数名 (这也是 MyBatis 在实现 Mapper 接口时为何要提供 @Param 注解来辅助获得参数名)
         */
    }

}
```

**<font style="color:#E8323C;">收获</font>**

1. 如果编译时添加了 -parameters 可以生成参数表, 反射时就可以拿到参数名
2. 如果编译时添加了 -g 可以生成调试信息, 但分为两种情况
    - 普通类, 会包含局部变量表, 用 asm 可以拿到参数名
    - 接口, 不会包含局部变量表, 无法获得参数名
        * 这也是 MyBatis 在实现 Mapper 接口时为何要提供 @Param 注解来辅助获得参数名

# 4 对象绑定与类型转换
## 4.1 底层第一套转换接口与实现


![](https://cdn.nlark.com/yuque/__mermaid_v3/695ac9e641d25b910f7d46af5e968ed9.svg)

+  Printer 把其它类型转为 String
+ Parser 把 String 转为其它类型
+ Formatter 综合 Printer 与 Parser 功能
+ Converter 把类型 S 转为类型 T
+ Printer、Parser、Converter 经过适配转换成 GenericConverter 放入 Converters 集合
+ FormattingConversionService 利用其它们实现转换

## 4.2 底层第二套转换接口
![](https://cdn.nlark.com/yuque/__mermaid_v3/dded866472a3682d5bd41ee320b9bfea.svg)

+ PropertyEditor 把 String 与其它类型相互转换
+ PropertyEditorRegistry 可以注册多个 PropertyEditor 对象
+ 与第一套接口直接可以通过 FormatterPropertyEditorAdapter 来进行适配

## 4.3 高层接口与实现
![](https://cdn.nlark.com/yuque/__mermaid_v3/a167628c55a4d9678e9a2dabc2697a2c.svg)

+ 它们都实现了 TypeConverter 这个高层转换接口，在转换时，会用到 TypeConverter Delegate 委派ConversionService 与 PropertyEditorRegistry 真正执行转换（Facade 门面模式）
    - 首先看是否有自定义转换器, @InitBinder 添加的即属于这种 (用了适配器模式把 Formatter 转为需要的 PropertyEditor)
    - 再看有没有 ConversionService 转换
    - 再利用默认的 PropertyEditor 转换
    - 最后有一些特殊处理
+ SimpleTypeConverter 仅做类型转换
+ BeanWrapperImpl 为 bean 的属性赋值，当需要时做类型转换，走 Property
+ DirectFieldAccessor 为 bean 的属性赋值，当需要时做类型转换，走 Field
+ ServletRequestDataBinder 为 bean 的属性执行绑定，当需要时做类型转换，根据 directFieldAccess 选择走 Property 还是 Field，具备校验与获取校验结果功能

## 4.4 类型转换与数据绑定
基本的类型转换与数据绑定用法



<font style="color:#000000;">SimpleTypeConverter</font>

```java
package com.itheima.a23;

import org.springframework.beans.SimpleTypeConverter;

import java.util.Date;

public class TestSimpleConverter {
    public static void main(String[] args) {
        // 仅有类型转换的功能
        SimpleTypeConverter typeConverter = new SimpleTypeConverter();
        Integer number = typeConverter.convertIfNecessary("13", int.class);
        Date date = typeConverter.convertIfNecessary("1999/03/04", Date.class);
        System.out.println(number);
        System.out.println(date);
    }
}
```





BeanWrapperImpl

```java
package com.itheima.a23;

import org.springframework.beans.BeanWrapperImpl;
import org.springframework.beans.SimpleTypeConverter;
import org.springframework.core.GenericTypeResolver;
import org.springframework.format.Formatter;
import org.springframework.format.support.FormatterPropertyEditorAdapter;
import org.springframework.format.support.FormattingConversionService;

import java.util.Date;

public class TestBeanWrapper {
    public static void main(String[] args) {
        // 利用反射原理, 为 bean 的属性赋值
        MyBean target = new MyBean();
        BeanWrapperImpl wrapper = new BeanWrapperImpl(target);
        wrapper.setPropertyValue("a", "10");
        wrapper.setPropertyValue("b", "hello");
        wrapper.setPropertyValue("c", "1999/03/04");
        System.out.println(target);
    }

    static class MyBean {
        private int a;
        private String b;
        private Date c;

        public int getA() {
            return a;
        }

        public void setA(int a) {
            this.a = a;
        }

        public String getB() {
            return b;
        }

        public void setB(String b) {
            this.b = b;
        }

        public Date getC() {
            return c;
        }

        public void setC(Date c) {
            this.c = c;
        }

        @Override
        public String toString() {
            return "MyBean{" +
                   "a=" + a +
                   ", b='" + b + '\'' +
                   ", c=" + c +
                   '}';
        }
    }
}
```



DirectFieldAccessor

```java
package com.itheima.a23;

import org.springframework.beans.DirectFieldAccessor;

import java.util.Date;

public class TestFieldAccessor {
    public static void main(String[] args) {
        // 利用反射原理, 为 bean 的属性赋值
        MyBean target = new MyBean();
        DirectFieldAccessor accessor = new DirectFieldAccessor(target);
        accessor.setPropertyValue("a", "10");
        accessor.setPropertyValue("b", "hello");
        accessor.setPropertyValue("c", "1999/03/04");
        System.out.println(target);
    }

    static class MyBean {
        private int a;
        private String b;
        private Date c;
        @Override
        public String toString() {
            return "MyBean{" +
                   "a=" + a +
                   ", b='" + b + '\'' +
                   ", c=" + c +
                   '}';
        }
    }
}
```

ServletRequestDataBinder

```java
package com.itheima.a23;

import org.springframework.beans.MutablePropertyValues;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.validation.DataBinder;
import org.springframework.web.bind.ServletRequestDataBinder;
import org.springframework.web.bind.ServletRequestParameterPropertyValues;

import java.util.Date;

public class TestServletDataBinder {

    public static void main(String[] args) {
        // web 环境下数据绑定
        MyBean target = new MyBean();
        ServletRequestDataBinder dataBinder = new ServletRequestDataBinder(target);
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setParameter("a", "10");
        request.setParameter("b", "hello");
        request.setParameter("c", "1999/03/04");

        dataBinder.bind(new ServletRequestParameterPropertyValues(request));

        System.out.println(target);
    }

    static class MyBean {
        private int a;
        private String b;
        private Date c;

        public int getA() {
            return a;
        }

        public void setA(int a) {
            this.a = a;
        }

        public String getB() {
            return b;
        }

        public void setB(String b) {
            this.b = b;
        }

        public Date getC() {
            return c;
        }

        public void setC(Date c) {
            this.c = c;
        }

        @Override
        public String toString() {
            return "MyBean{" +
                   "a=" + a +
                   ", b='" + b + '\'' +
                   ", c=" + c +
                   '}';
        }
    }
}
```



```java
package com.itheima.a23;

import org.springframework.beans.MutablePropertyValues;
import org.springframework.validation.DataBinder;

import java.util.Date;

public class TestDataBinder {

    public static void main(String[] args) {
        // 执行数据绑定
        MyBean target = new MyBean();
        DataBinder dataBinder = new DataBinder(target);
        dataBinder.initDirectFieldAccess();
        MutablePropertyValues pvs = new MutablePropertyValues();
        pvs.add("a", "10");
        pvs.add("b", "hello");
        pvs.add("c", "1999/03/04");
        dataBinder.bind(pvs);
        System.out.println(target);
    }

    static class MyBean {
        private int a;
        private String b;
        private Date c;

        @Override
        public String toString() {
            return "MyBean{" +
                   "a=" + a +
                   ", b='" + b + '\'' +
                   ", c=" + c +
                   '}';
        }
    }
}
```

## 4.5 数据绑定工厂
代码参考

```java
package com.itheima.a23;

import org.springframework.boot.convert.ApplicationConversionService;
import org.springframework.format.annotation.DateTimeFormat;
import org.springframework.format.support.DefaultFormattingConversionService;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.web.bind.ServletRequestParameterPropertyValues;
import org.springframework.web.bind.WebDataBinder;
import org.springframework.web.bind.annotation.InitBinder;
import org.springframework.web.bind.support.ConfigurableWebBindingInitializer;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.servlet.mvc.method.annotation.ServletRequestDataBinderFactory;

import java.util.Date;

public class TestServletDataBinderFactory {
    public static void main(String[] args) throws Exception {
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setParameter("birthday", "1999|01|02");
        request.setParameter("address.name", "西安");

        User target = new User();
        // "1. 用工厂, 无转换功能"
//        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null, null);
        // "2. 用 @InitBinder 转换"          PropertyEditorRegistry PropertyEditor
//        InvocableHandlerMethod method = new InvocableHandlerMethod(new MyController(), MyController.class.getMethod("aaa", WebDataBinder.class));
//        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(List.of(method), null);
        // "3. 用 ConversionService 转换"    ConversionService Formatter
//        FormattingConversionService service = new FormattingConversionService();
//        service.addFormatter(new MyDateFormatter("用 ConversionService 方式扩展转换功能"));
//        ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
//        initializer.setConversionService(service);
//        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null, initializer);
        // "4. 同时加了 @InitBinder 和 ConversionService"
//        InvocableHandlerMethod method = new InvocableHandlerMethod(new MyController(), MyController.class.getMethod("aaa", WebDataBinder.class));
//
//        FormattingConversionService service = new FormattingConversionService();
//        service.addFormatter(new MyDateFormatter("用 ConversionService 方式扩展转换功能"));
//        ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
//        initializer.setConversionService(service);
//
//        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(List.of(method), initializer);
        // "5. 使用默认 ConversionService 转换"
        ApplicationConversionService service = new ApplicationConversionService();
        ConfigurableWebBindingInitializer initializer = new ConfigurableWebBindingInitializer();
        initializer.setConversionService(service);

        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null, initializer);

        WebDataBinder dataBinder = factory.createBinder(new ServletWebRequest(request), target, "user");
        dataBinder.bind(new ServletRequestParameterPropertyValues(request));
        System.out.println(target);
    }

    static class MyController {
        @InitBinder
        public void aaa(WebDataBinder dataBinder) {
            // 扩展 dataBinder 的转换器
            dataBinder.addCustomFormatter(new MyDateFormatter("用 @InitBinder 方式扩展的"));
        }
    }

    public static class User {
        @DateTimeFormat(pattern = "yyyy|MM|dd")
        private Date birthday;
        private Address address;

        public Address getAddress() {
            return address;
        }

        public void setAddress(Address address) {
            this.address = address;
        }

        public Date getBirthday() {
            return birthday;
        }

        public void setBirthday(Date birthday) {
            this.birthday = birthday;
        }

        @Override
        public String toString() {
            return "User{" +
                   "birthday=" + birthday +
                   ", address=" + address +
                   '}';
        }
    }

    public static class Address {
        private String name;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        @Override
        public String toString() {
            return "Address{" +
                   "name='" + name + '\'' +
                   '}';
        }
    }
}
```



```java
package com.itheima.a23;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.GenericTypeResolver;
import org.springframework.format.Formatter;

import java.lang.reflect.Type;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.Locale;

public class MyDateFormatter implements Formatter<Date> {
    private static final Logger log = LoggerFactory.getLogger(MyDateFormatter.class);
    private final String desc;

    public MyDateFormatter(String desc) {
        this.desc = desc;
    }

    @Override
    public String print(Date date, Locale locale) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy|MM|dd");
        return sdf.format(date);
    }

    @Override
    public Date parse(String text, Locale locale) throws ParseException {
        log.debug(">>>>>> 进入了: {}", desc);
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy|MM|dd");
        return sdf.parse(text);
    }


}
```

**<font style="color:#F5222D;">收获</font>**

ServletRequestDataBinderFactory 的用法和扩展点

1. 可以解析控制器的 @InitBinder 标注方法作为扩展点，添加自定义转换器
    - 控制器私有范围
2. 可以通过 ConfigurableWebBindingInitializer 配置 ConversionService 作为扩展点，添加自定义转换器
    - 公共范围
3. 同时加了 @InitBinder 和 ConversionService 的转换优先级
    1. 优先采用 @InitBinder 的转换器
    2. 其次使用 ConversionService 的转换器
    3. 使用默认转换器
    4. 特殊处理（例如有参构造）

## 4.6 获取泛型参数
代码参考

```java
package com.itheima.a23.sub;

class BaseDao<T> {
    T findOne() {
        return null;
    }
}
```



```java
package com.itheima.a23.sub;

class EmployeeDao extends BaseDao {
}
```



```java
package com.itheima.a23.sub;

class Student {
}
```



```java
package com.itheima.a23.sub;

class StudentDao extends BaseDao<Student> {
}
```



```java
package com.itheima.a23.sub;

class Teacher {
}
```



```java
package com.itheima.a23.sub;

class TeacherDao extends BaseDao<Teacher> {
}
```



```java
package com.itheima.a23.sub;

import org.springframework.core.GenericTypeResolver;
import org.springframework.core.ResolvableType;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

public class TestGenericType {
    public static void main(String[] args) {
        // 小技巧
        // 1. java api
        System.out.println(">>>>>>>>>>>>>>>>>>>>>>>");
        Type type = TeacherDao.class.getGenericSuperclass();
        System.out.println(type);

        if (type instanceof ParameterizedType parameterizedType) {
            System.out.println(parameterizedType.getActualTypeArguments()[0]);
        }

        // 2. spring api 1
        System.out.println(">>>>>>>>>>>>>>>>>>>>>>>");
        Class<?> t = GenericTypeResolver.resolveTypeArgument(TeacherDao.class, BaseDao.class);
        System.out.println(t);

        // 3. spring api 2
        System.out.println(">>>>>>>>>>>>>>>>>>>>>>>");
        System.out.println(ResolvableType.forClass(TeacherDao.class).getSuperType().getGeneric().resolve());
    }

}
```

**<font style="color:#F5222D;">收获</font>**

1. java api 获取泛型参数
2. spring api 获取泛型参数

# 5 @ControllerAdvice之@InitBinder


**<font style="color:#E8323C;">准备@InitBinder</font>**

**准备@InitBinder**在整个 HandlerAdapter 调用过程中所处的位置



![](https://cdn.nlark.com/yuque/__mermaid_v3/9865e0947a3a93deeebbf2f35e93dcb7.svg)

+ RequestMappingHandlerAdapter 在图中缩写为 HandlerAdapter
+ HandlerMethodArgumentResolverComposite 在图中缩写为 ArgumentResolvers
+ HandlerMethodReturnValueHandlerComposite 在图中缩写为 ReturnValueHandlers



**<font style="color:#E8323C;">收获</font>**

1. RequestMappingHandlerAdapter 初始化时会解析 @ControllerAdvice 中的 @InitBinder 方法
2. RequestMappingHandlerAdapter 会以类为单位，在该类首次使用时，解析此类的 @InitBinder 方法
3. 以上两种 @InitBinder 的解析结果都会缓存来避免重复解析
4. 控制器方法调用时，会综合利用本类的 @InitBinder 方法和 @ControllerAdvice 中的 @InitBinder 方法创建绑定工厂



# 6 控制器方法执行流程


图1

![](https://cdn.nlark.com/yuque/__mermaid_v3/242ef4948bfe86d508c41c2530cb9b74.svg)

HandlerMethod 需要

+ bean 即是哪个 Controller
+ method 即是 Controller 中的哪个方法



ServletInvocableHandlerMethod 需要

+ WebDataBinderFactory 负责对象绑定、类型转换
+ ParameterNameDiscoverer 负责参数名解析
+ HandlerMethodArgumentResolverComposite 负责解析参数
+ HandlerMethodReturnValueHandlerComposite 负责处理返回值



图2

![](https://cdn.nlark.com/yuque/__mermaid_v3/b47a873826911f95fc6174ec91d411d7.svg)



图3

![](https://cdn.nlark.com/yuque/__mermaid_v3/6cb1bf6056fc792217ccb43bfb24a595.svg)



# 7 @ControllerAdvice之@ModelAttribute
代码参考

```java
package com.itheima.a26;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.http.HttpStatus;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.support.DefaultSessionAttributeStore;
import org.springframework.web.bind.support.WebDataBinderFactory;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.method.annotation.*;
import org.springframework.web.method.support.HandlerMethodArgumentResolverComposite;
import org.springframework.web.method.support.InvocableHandlerMethod;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.*;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Collections;
import java.util.List;

import static com.itheima.a26.WebConfig.*;

public class A26 {

    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(WebConfig.class);

        RequestMappingHandlerAdapter adapter = new RequestMappingHandlerAdapter();
        adapter.setApplicationContext(context);
        adapter.afterPropertiesSet();

        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setParameter("name", "张三");
        /*
            现在可以通过 ServletInvocableHandlerMethod 把这些整合在一起, 并完成控制器方法的调用, 如下
         */
        ServletInvocableHandlerMethod handlerMethod = new ServletInvocableHandlerMethod(
                new Controller1(), Controller1.class.getMethod("foo", User.class));

        ServletRequestDataBinderFactory factory = new ServletRequestDataBinderFactory(null, null);

        handlerMethod.setDataBinderFactory(factory);
        handlerMethod.setParameterNameDiscoverer(new DefaultParameterNameDiscoverer());
        handlerMethod.setHandlerMethodArgumentResolvers(getArgumentResolvers(context));

        ModelAndViewContainer container = new ModelAndViewContainer();

        // 获取模型工厂方法
        Method getModelFactory = RequestMappingHandlerAdapter.class.getDeclaredMethod("getModelFactory", HandlerMethod.class, WebDataBinderFactory.class);
        getModelFactory.setAccessible(true);
        ModelFactory modelFactory = (ModelFactory) getModelFactory.invoke(adapter, handlerMethod, factory);

        // 初始化模型数据
        modelFactory.initModel(new ServletWebRequest(request), container, handlerMethod);

        handlerMethod.invokeAndHandle(new ServletWebRequest(request), container);

        System.out.println(container.getModel());

        context.close();

        /*
            学到了什么
                a. 控制器方法是如何调用的
                b. 模型数据如何产生
                c. advice 之二, @ModelAttribute 补充模型数据
         */
    }

    public static HandlerMethodArgumentResolverComposite getArgumentResolvers(AnnotationConfigApplicationContext context) {
        HandlerMethodArgumentResolverComposite composite = new HandlerMethodArgumentResolverComposite();
        composite.addResolvers(
                new RequestParamMethodArgumentResolver(context.getDefaultListableBeanFactory(), false),
                new PathVariableMethodArgumentResolver(),
                new RequestHeaderMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ServletCookieValueMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ExpressionValueMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ServletRequestMethodArgumentResolver(),
                new ServletModelAttributeMethodProcessor(false),
                new RequestResponseBodyMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())),
                new ServletModelAttributeMethodProcessor(true),
                new RequestParamMethodArgumentResolver(context.getDefaultListableBeanFactory(), true)
        );
        return composite;
    }
}
```



```java
package com.itheima.a26;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;

@Configuration
public class WebConfig {

    @ControllerAdvice
    static class MyControllerAdvice {
        @ModelAttribute("a")
        public String aa() {
            return "aa";
        }
    }

    @Controller
    static class Controller1 {
        @ModelAttribute("b")
        public String aa() {
            return "bb";
        }

        @ResponseStatus(HttpStatus.OK)
        public ModelAndView foo(@ModelAttribute("u") User user) {
            System.out.println("foo");
            return null;
        }
    }

    static class User {
        private String name;

        public void setName(String name) {
            this.name = name;
        }

        public String getName() {

            return name;
        }

        @Override
        public String toString() {
            return "User{" +
                   "name='" + name + '\'' +
                   '}';
        }
    }
}
```

****

**准备 @ModelAttribute**在整个 HandlerAdapter 调用过程中所处的位置

![](https://cdn.nlark.com/yuque/__mermaid_v3/f031a47a35ba4f8321bc9b8b165e49f9.svg)



**<font style="color:#E8323C;">收获</font>**

1. RequestMappingHandlerAdapter 初始化时会解析 @ControllerAdvice 中的 @ModelAttribute 方法
2. RequestMappingHandlerAdapter 会以类为单位，在该类首次使用时，解析此类的 @ModelAttribute 方法
3. 以上两种 @ModelAttribute 的解析结果都会缓存来避免重复解析
4. 控制器方法调用时，会综合利用本类的 @ModelAttribute 方法和 @ControllerAdvice 中的 @ModelAttribute 方法创建模型工厂

# 8 返回值处理器
**<font style="color:#E8323C;">常见返回值处理器</font>**

代码参考

```java
package com.itheima.a27;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.View;
import org.springframework.web.servlet.mvc.method.annotation.*;
import org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator;
import org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver;
import org.springframework.web.util.UrlPathHelper;

import java.lang.reflect.Method;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Locale;

/*
    目标: 解析控制器方法的返回值
    常见的返回值处理器
        org.springframework.web.servlet.mvc.method.annotation.ModelAndViewMethodReturnValueHandler@4c9e38
        org.springframework.web.method.annotation.ModelMethodProcessor@5d1e09bc
        org.springframework.web.servlet.mvc.method.annotation.ViewMethodReturnValueHandler@4bdc8b5d
        org.springframework.web.servlet.mvc.method.annotation.ResponseBodyEmitterReturnValueHandler@3bcd426c
        org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBodyReturnValueHandler@5f14a673
        org.springframework.web.servlet.mvc.method.annotation.HttpEntityMethodProcessor@726a17c4
        org.springframework.web.servlet.mvc.method.annotation.HttpHeadersReturnValueHandler@5dc3fcb7
        org.springframework.web.servlet.mvc.method.annotation.CallableMethodReturnValueHandler@c4c0b41
        org.springframework.web.servlet.mvc.method.annotation.DeferredResultMethodReturnValueHandler@76911385
        org.springframework.web.servlet.mvc.method.annotation.AsyncTaskMethodReturnValueHandler@5467eea4
        org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor@160396db
        org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor@7a799159
        org.springframework.web.servlet.mvc.method.annotation.ViewNameMethodReturnValueHandler@40ab8a8
        org.springframework.web.method.annotation.MapMethodProcessor@6ff37443
        org.springframework.web.servlet.mvc.method.annotation.ServletModelAttributeMethodProcessor@65cc8228
 */
public class A27 {
    private static final Logger log = LoggerFactory.getLogger(A27.class);

    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(WebConfig.class);
        // 1. 测试返回值类型为 ModelAndView

        // 2. 测试返回值类型为 String 时, 把它当做视图名

        // 3. 测试返回值添加了 @ModelAttribute 注解时, 此时需找到默认视图名

        // 4. 测试返回值不加 @ModelAttribute 注解且返回非简单类型时, 此时需找到默认视图名

        // 5. 测试返回值类型为 ResponseEntity 时, 此时不走视图流程

        // 6. 测试返回值类型为 HttpHeaders 时, 此时不走视图流程

        // 7. 测试返回值添加了 @ResponseBody 注解时, 此时不走视图流程
        test7(context);

        /*
            学到了什么
                a. 每个返回值处理器能干啥
                    1) 看是否支持某种返回值
                    2) 返回值或作为模型、或作为视图名、或作为响应体 ...
                b. 组合模式在 Spring 中的体现 + 1
         */
    }
    private static void test7(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test7");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            if (!container.isRequestHandled()) {
                renderView(context, container, webRequest); // 渲染视图
            } else {
                for (String name : response.getHeaderNames()) {
                    System.out.println(name + "=" + response.getHeader(name));
                }
                System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
            }
        }
    }

    private static void test6(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test6");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            if (!container.isRequestHandled()) {
                renderView(context, container, webRequest); // 渲染视图
            } else {
                for (String name : response.getHeaderNames()) {
                    System.out.println(name + "=" + response.getHeader(name));
                }
                System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
            }
        }
    }

    private static void test5(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test5");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        ServletWebRequest webRequest = new ServletWebRequest(request, response);
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            if (!container.isRequestHandled()) {
                renderView(context, container, webRequest); // 渲染视图
            } else {
                System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
            }
        }
    }

    private static void test4(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test4");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setRequestURI("/test4");
        UrlPathHelper.defaultInstance.resolveAndCacheLookupPath(request);
        ServletWebRequest webRequest = new ServletWebRequest(request, new MockHttpServletResponse());
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            renderView(context, container, webRequest); // 渲染视图
        }
    }

    private static void test3(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test3");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        MockHttpServletRequest request = new MockHttpServletRequest();
        request.setRequestURI("/test3");
        UrlPathHelper.defaultInstance.resolveAndCacheLookupPath(request);
        ServletWebRequest webRequest = new ServletWebRequest(request, new MockHttpServletResponse());
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            renderView(context, container, webRequest); // 渲染视图
        }
    }

    private static void test2(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test2");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        ServletWebRequest webRequest = new ServletWebRequest(new MockHttpServletRequest(), new MockHttpServletResponse());
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            renderView(context, container, webRequest); // 渲染视图
        }
    }

    private static void test1(AnnotationConfigApplicationContext context) throws Exception {
        Method method = Controller.class.getMethod("test1");
        Controller controller = new Controller();
        Object returnValue = method.invoke(controller); // 获取返回值

        HandlerMethod methodHandle = new HandlerMethod(controller, method);

        ModelAndViewContainer container = new ModelAndViewContainer();
        HandlerMethodReturnValueHandlerComposite composite = getReturnValueHandler();
        ServletWebRequest webRequest = new ServletWebRequest(new MockHttpServletRequest(), new MockHttpServletResponse());
        if (composite.supportsReturnType(methodHandle.getReturnType())) { // 检查是否支持此类型的返回值
            composite.handleReturnValue(returnValue, methodHandle.getReturnType(), container, webRequest);
            System.out.println(container.getModel());
            System.out.println(container.getViewName());
            renderView(context, container, webRequest); // 渲染视图
        }
    }

    public static HandlerMethodReturnValueHandlerComposite getReturnValueHandler() {
        HandlerMethodReturnValueHandlerComposite composite = new HandlerMethodReturnValueHandlerComposite();
        composite.addHandler(new ModelAndViewMethodReturnValueHandler());
        composite.addHandler(new ViewNameMethodReturnValueHandler());
        composite.addHandler(new ServletModelAttributeMethodProcessor(false));
        composite.addHandler(new HttpEntityMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())));
        composite.addHandler(new HttpHeadersReturnValueHandler());
        composite.addHandler(new RequestResponseBodyMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())));
        composite.addHandler(new ServletModelAttributeMethodProcessor(true));
        return composite;
    }

    @SuppressWarnings("all")
    private static void renderView(AnnotationConfigApplicationContext context, ModelAndViewContainer container,
                                   ServletWebRequest webRequest) throws Exception {
        log.debug(">>>>>> 渲染视图");
        FreeMarkerViewResolver resolver = context.getBean(FreeMarkerViewResolver.class);
        String viewName = container.getViewName() != null ? container.getViewName() : new DefaultRequestToViewNameTranslator().getViewName(webRequest.getRequest());
        log.debug("没有获取到视图名, 采用默认视图名: {}", viewName);
        // 每次渲染时, 会产生新的视图对象, 它并非被 Spring 所管理, 但确实借助了 Spring 容器来执行初始化
        View view = resolver.resolveViewName(viewName, Locale.getDefault());
        view.render(container.getModel(), webRequest.getRequest(), webRequest.getResponse());
        System.out.println(new String(((MockHttpServletResponse) webRequest.getResponse()).getContentAsByteArray(), StandardCharsets.UTF_8));
    }

    static class Controller {
        private static final Logger log = LoggerFactory.getLogger(Controller.class);

        public ModelAndView test1() {
            log.debug("test1()");
            ModelAndView mav = new ModelAndView("view1");
            mav.addObject("name", "张三");
            return mav;
        }

        public String test2() {
            log.debug("test2()");
            return "view2";
        }

        @ModelAttribute
//        @RequestMapping("/test3")
        public User test3() {
            log.debug("test3()");
            return new User("李四", 20);
        }

        public User test4() {
            log.debug("test4()");
            return new User("王五", 30);
        }

        public HttpEntity<User> test5() {
            log.debug("test5()");
            return new HttpEntity<>(new User("赵六", 40));
        }

        public HttpHeaders test6() {
            log.debug("test6()");
            HttpHeaders headers = new HttpHeaders();
            headers.add("Content-Type", "text/html");
            return headers;
        }

        @ResponseBody
        public User test7() {
            log.debug("test7()");
            return new User("钱七", 50);
        }
    }

    // 必须用 public 修饰, 否则 freemarker 渲染其 name, age 属性时失败
    public static class User {
        private String name;
        private int age;

        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "User{" +
                   "name='" + name + '\'' +
                   ", age=" + age +
                   '}';
        }
    }
}

```



```java
package com.itheima.a27;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.view.AbstractUrlBasedView;
import org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer;
import org.springframework.web.servlet.view.freemarker.FreeMarkerView;
import org.springframework.web.servlet.view.freemarker.FreeMarkerViewResolver;

@Configuration
public class WebConfig {

    @Bean
    public FreeMarkerConfigurer freeMarkerConfigurer() {
        FreeMarkerConfigurer configurer = new FreeMarkerConfigurer();
        configurer.setDefaultEncoding("utf-8");
        configurer.setTemplateLoaderPath("classpath:templates");
        return configurer;
    }

    @Bean // FreeMarkerView 在借助 Spring 初始化时，会要求 web 环境才会走 setConfiguration, 这里想办法去掉了 web 环境的约束
    public FreeMarkerViewResolver viewResolver(FreeMarkerConfigurer configurer) {
        FreeMarkerViewResolver resolver = new FreeMarkerViewResolver() {
            @Override
            protected AbstractUrlBasedView instantiateView() {
                FreeMarkerView view = new FreeMarkerView() {
                    @Override
                    protected boolean isContextRequired() {
                        return false;
                    }
                };
                view.setConfiguration(configurer.getConfiguration());
                return view;
            }
        };
        resolver.setContentType("text/html;charset=utf-8");
        resolver.setPrefix("/");
        resolver.setSuffix(".ftl");
        resolver.setExposeSpringMacroHelpers(false);
        return resolver;
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. 常见的返回值处理器
    - ModelAndView，分别获取其模型和视图名，放入 ModelAndViewContainer
    - 返回值类型为 String 时，把它当做视图名，放入 ModelAndViewContainer
    - 返回值添加了 @ModelAttribute 注解时，将返回值作为模型，放入 ModelAndViewContainer，此时需找到默认视图名
    - 返回值省略 @ModelAttribute 注解且返回非简单类型时，将返回值作为模型，放入 ModelAndViewContainer，此时需找到默认视图名
    - 返回值类型为 ResponseEntity 时，此时走 MessageConverter，并设置 ModelAndViewContainer.requestHandled 为 true
    - 返回值类型为 HttpHeaders 时，会设置 ModelAndViewContainer.requestHandled 为 true
    - 返回值添加了 @ResponseBody 注解时，此时走 MessageConverter，并设置 ModelAndViewContainer.requestHandled 为 true
2. 组合模式在 Spring 中的体现 + 1

# 9 MessageConverter
**<font style="color:#E8323C;">MessageConverter的作用</font>**

代码参考

```java
package com.itheima.a28;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonProperty;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.http.converter.xml.MappingJackson2XmlHttpMessageConverter;
import org.springframework.mock.http.MockHttpInputMessage;
import org.springframework.mock.http.MockHttpOutputMessage;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.HttpMediaTypeNotAcceptableException;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.List;

public class A28 {
    public static void main(String[] args) throws IOException, NoSuchMethodException, HttpMediaTypeNotAcceptableException {
//        test1();
//        test2();
//        test3();
        test4();

        /*
            学到了什么
                a. MessageConverter 的作用, @ResponseBody 是返回值处理器解析的, 但具体转换工作是 MessageConverter 做的
                b. 如何选择 MediaType
                    - 首先看 @RequestMapping 上有没有指定
                    - 其次看 request 的 Accept 头有没有指定
                    - 最后按 MessageConverter 的顺序, 谁能谁先转换
         */

    }

    private static void test4() throws IOException, HttpMediaTypeNotAcceptableException, NoSuchMethodException {
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        ServletWebRequest webRequest = new ServletWebRequest(request, response);

        request.addHeader("Accept", "application/xml");
        response.setContentType("application/json");

        RequestResponseBodyMethodProcessor processor = new RequestResponseBodyMethodProcessor(
                List.of(
                        new MappingJackson2HttpMessageConverter(), new MappingJackson2XmlHttpMessageConverter()
                ));
        processor.handleReturnValue(
                new User("张三", 18),
                new MethodParameter(A28.class.getMethod("user"), -1),
                new ModelAndViewContainer(),
                webRequest
        );
        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
    }

    @ResponseBody
    @RequestMapping(produces = "application/json")
    public User user() {
        return null;
    }

    private static void test3() throws IOException {
        MockHttpInputMessage message = new MockHttpInputMessage("""
                {
                    "name":"李四",
                    "age":20
                }
                """.getBytes(StandardCharsets.UTF_8));
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        if (converter.canRead(User.class, MediaType.APPLICATION_JSON)) {
            Object read = converter.read(User.class, message);
            System.out.println(read);
        }
    }

    private static void test2() throws IOException {
        MockHttpOutputMessage message = new MockHttpOutputMessage();
        MappingJackson2XmlHttpMessageConverter converter = new MappingJackson2XmlHttpMessageConverter();
        if (converter.canWrite(User.class, MediaType.APPLICATION_XML)) {
            converter.write(new User("李四", 20), MediaType.APPLICATION_XML, message);
            System.out.println(message.getBodyAsString());
        }
    }

    public static void test1() throws IOException {
        MockHttpOutputMessage message = new MockHttpOutputMessage();
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        if (converter.canWrite(User.class, MediaType.APPLICATION_JSON)) {
            converter.write(new User("张三", 18), MediaType.APPLICATION_JSON, message);
            System.out.println(message.getBodyAsString());
        }
    }

    public static class User {
        private String name;
        private int age;

        @JsonCreator
        public User(@JsonProperty("name") String name, @JsonProperty("age") int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return "User{" +
                   "name='" + name + '\'' +
                   ", age=" + age +
                   '}';
        }
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. MessageConverter 的作用
    - @ResponseBody 是返回值处理器解析的
    - 但具体转换工作是 MessageConverter 做的
2. 如何选择 MediaType
    - 首先看 @RequestMapping 上有没有指定
    - 其次看 request 的 Accept 头有没有指定
    - 最后按 MessageConverter 的顺序, 谁能谁先转换

# 10 @ControllerAdvice之ResponseBodyAdvice
ResponseBodyAdvice增强

代码参考

```java
package com.itheima.a29;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.method.ControllerAdviceBean;
import org.springframework.web.method.annotation.ExpressionValueMethodArgumentResolver;
import org.springframework.web.method.annotation.RequestHeaderMethodArgumentResolver;
import org.springframework.web.method.annotation.RequestParamMethodArgumentResolver;
import org.springframework.web.method.support.HandlerMethodArgumentResolverComposite;
import org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite;
import org.springframework.web.method.support.ModelAndViewContainer;
import org.springframework.web.servlet.mvc.method.annotation.*;

import java.nio.charset.StandardCharsets;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;

public class A29 {

    // {"name":"王五","age":18}
    // {"code":xx, "msg":xx, data: {"name":"王五","age":18} }
    public static void main(String[] args) throws Exception {
        AnnotationConfigApplicationContext context =
                new AnnotationConfigApplicationContext(WebConfig.class);

        ServletInvocableHandlerMethod handlerMethod = new ServletInvocableHandlerMethod(
                context.getBean(WebConfig.MyController.class),
                WebConfig.MyController.class.getMethod("user")
        );
        handlerMethod.setDataBinderFactory(new ServletRequestDataBinderFactory(Collections.emptyList(), null));
        handlerMethod.setParameterNameDiscoverer(new DefaultParameterNameDiscoverer());
        handlerMethod.setHandlerMethodArgumentResolvers(getArgumentResolvers(context));
        handlerMethod.setHandlerMethodReturnValueHandlers(getReturnValueHandlers(context));

        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        ModelAndViewContainer container = new ModelAndViewContainer();
        handlerMethod.invokeAndHandle(new ServletWebRequest(request, response), container);

        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));

        /*
            学到了什么
                a. advice 之三, ResponseBodyAdvice 返回响应体前包装
         */
    }

    public static HandlerMethodArgumentResolverComposite getArgumentResolvers(AnnotationConfigApplicationContext context) {
        HandlerMethodArgumentResolverComposite composite = new HandlerMethodArgumentResolverComposite();
        composite.addResolvers(
                new RequestParamMethodArgumentResolver(context.getDefaultListableBeanFactory(), false),
                new PathVariableMethodArgumentResolver(),
                new RequestHeaderMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ServletCookieValueMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ExpressionValueMethodArgumentResolver(context.getDefaultListableBeanFactory()),
                new ServletRequestMethodArgumentResolver(),
                new ServletModelAttributeMethodProcessor(false),
                new RequestResponseBodyMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())),
                new ServletModelAttributeMethodProcessor(true),
                new RequestParamMethodArgumentResolver(context.getDefaultListableBeanFactory(), true)
        );
        return composite;
    }

    public static HandlerMethodReturnValueHandlerComposite getReturnValueHandlers(AnnotationConfigApplicationContext context) {
        // 添加 advice
        List<ControllerAdviceBean> annotatedBeans = ControllerAdviceBean.findAnnotatedBeans(context);
        List<Object> collect = annotatedBeans.stream().filter(b -> ResponseBodyAdvice.class.isAssignableFrom(b.getBeanType()))
                .collect(Collectors.toList());

        HandlerMethodReturnValueHandlerComposite composite = new HandlerMethodReturnValueHandlerComposite();
        composite.addHandler(new ModelAndViewMethodReturnValueHandler());
        composite.addHandler(new ViewNameMethodReturnValueHandler());
        composite.addHandler(new ServletModelAttributeMethodProcessor(false));
        composite.addHandler(new HttpEntityMethodProcessor(List.of(new MappingJackson2HttpMessageConverter())));
        composite.addHandler(new HttpHeadersReturnValueHandler());
        composite.addHandler(new RequestResponseBodyMethodProcessor(List.of(new MappingJackson2HttpMessageConverter()), collect));
        composite.addHandler(new ServletModelAttributeMethodProcessor(true));
        return composite;
    }
}
```





```

```java
package com.itheima.a29;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.annotation.JsonProperty;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class Result {
    private int code;
    private String msg;
    private Object data;

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }

    @JsonCreator
    private Result(@JsonProperty("code") int code, @JsonProperty("data") Object data) {
        this.code = code;
        this.data = data;
    }

    private Result(int code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    public static Result ok() {
        return new Result(200, null);
    }

    public static Result ok(Object data) {
        return new Result(200, data);
    }

    public static Result error(String msg) {
        return new Result(500, "服务器内部错误:" + msg);
    }
}
```



```java
package com.itheima.a29;

import org.springframework.context.annotation.Configuration;
import org.springframework.core.MethodParameter;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

@Configuration
public class WebConfig {

    @ControllerAdvice
    static class MyControllerAdvice implements ResponseBodyAdvice<Object> {
        // 满足条件才转换
        public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
            if (returnType.getMethodAnnotation(ResponseBody.class) != null ||
                AnnotationUtils.findAnnotation(returnType.getContainingClass(), ResponseBody.class) != null) {
//                returnType.getContainingClass().isAnnotationPresent(ResponseBody.class)) {
                return true;
            }
            return false;
        }

        // 将 User 或其它类型统一为 Result 类型
        public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
            if (body instanceof Result) {
                return body;
            }
            return Result.ok(body);
        }
    }

    // @Controller
    // @ResponseBody
    @RestController
    public static class MyController {
        public User user() {
            return new User("王五", 18);
        }
    }

    public static class User {
        private String name;
        private int age;

        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }
}
```



**ResponseBodyAdvice 增强**在整个 HandlerAdapter 调用过程中所处的位置

![](https://cdn.nlark.com/yuque/__mermaid_v3/806ead2fe68d467e2653ca271c64b149.svg)

**<font style="color:#E8323C;">收获</font>**

1. ResponseBodyAdvice 返回响应体前包装

# 11 异常解析器
**<font style="color:#E8323C;">ExceptionHandlerExceptionResolver</font>**

代码参考

```java
package com.itheima.a30;

import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.sql.SQLException;
import java.util.List;
import java.util.Map;

public class A30 {
    public static void main(String[] args) throws NoSuchMethodException {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.setMessageConverters(List.of(new MappingJackson2HttpMessageConverter()));
        resolver.afterPropertiesSet();

        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();
        // 1.测试 json
//        HandlerMethod handlerMethod = new HandlerMethod(new Controller1(), Controller1.class.getMethod("foo"));
//        Exception e = new ArithmeticException("被零除");
//        resolver.resolveException(request, response, handlerMethod, e);
//        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
        // 2.测试 mav
//        HandlerMethod handlerMethod = new HandlerMethod(new Controller2(), Controller2.class.getMethod("foo"));
//        Exception e = new ArithmeticException("被零除");
//        ModelAndView mav = resolver.resolveException(request, response, handlerMethod, e);
//        System.out.println(mav.getModel());
//        System.out.println(mav.getViewName());
        // 3.测试嵌套异常
//        HandlerMethod handlerMethod = new HandlerMethod(new Controller3(), Controller3.class.getMethod("foo"));
//        Exception e = new Exception("e1", new RuntimeException("e2", new IOException("e3")));
//        resolver.resolveException(request, response, handlerMethod, e);
//        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));

        // 4.测试异常处理方法参数解析
        HandlerMethod handlerMethod = new HandlerMethod(new Controller4(), Controller4.class.getMethod("foo"));
        Exception e = new Exception("e1");
        resolver.resolveException(request, response, handlerMethod, e);
        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
        /*
            学到了什么
                a. ExceptionHandlerExceptionResolver 能够重用参数解析器、返回值处理器，实现组件重用
                b. 能够支持嵌套异常
         */
    }

    static class Controller1 {
        public void foo() {

        }
        @ExceptionHandler
        @ResponseBody
        public Map<String, Object> handle(ArithmeticException e) {
            return Map.of("error", e.getMessage());
        }
    }

    static class Controller2 {
        public void foo() {

        }
        @ExceptionHandler
        public ModelAndView handle(ArithmeticException e) {
            return new ModelAndView("test2", Map.of("error", e.getMessage()));
        }
    }

    static class Controller3 {
        public void foo() {

        }
        @ExceptionHandler
        @ResponseBody
        public Map<String, Object> handle(IOException e3) {
            return Map.of("error", e3.getMessage());
        }
    }

    static class Controller4 {
        public void foo() {}
        @ExceptionHandler
        @ResponseBody
        public Map<String, Object> handler(Exception e, HttpServletRequest request) {
            System.out.println(request);
            return Map.of("error", e.getMessage());
        }
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. 它能够重用参数解析器、返回值处理器，实现组件重用
2. 它能够支持嵌套异常

# 12 @ControllerAdvice之@ExceptionHandler
**<font style="color:#E8323C;"> @ExceptionHandler</font>**

 代码参考

```java
package com.itheima.a31;

import com.itheima.a30.A30;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.mock.web.MockHttpServletRequest;
import org.springframework.mock.web.MockHttpServletResponse;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.List;
import java.util.Map;

public class A31 {
    public static void main(String[] args) throws NoSuchMethodException {
        MockHttpServletRequest request = new MockHttpServletRequest();
        MockHttpServletResponse response = new MockHttpServletResponse();

//        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
//        resolver.setMessageConverters(List.of(new MappingJackson2HttpMessageConverter()));
//        resolver.afterPropertiesSet();

        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(WebConfig.class);
        ExceptionHandlerExceptionResolver resolver = context.getBean(ExceptionHandlerExceptionResolver.class);

        HandlerMethod handlerMethod = new HandlerMethod(new Controller5(), Controller5.class.getMethod("foo"));
        Exception e = new Exception("e1");
        resolver.resolveException(request, response, handlerMethod, e);
        System.out.println(new String(response.getContentAsByteArray(), StandardCharsets.UTF_8));
    }

    static class Controller5 {
        public void foo() {

        }
    }
}
```



```java
package com.itheima.a31;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver;

import java.util.List;
import java.util.Map;

@Configuration
public class WebConfig {
    @ControllerAdvice
    static class MyControllerAdvice {
        @ExceptionHandler
        @ResponseBody
        public Map<String, Object> handle(Exception e) {
            return Map.of("error", e.getMessage());
        }
    }

    @Bean
    public ExceptionHandlerExceptionResolver resolver() {
        ExceptionHandlerExceptionResolver resolver = new ExceptionHandlerExceptionResolver();
        resolver.setMessageConverters(List.of(new MappingJackson2HttpMessageConverter()));
        return resolver;
    }
}
```



**<font style="color:#E8323C;">收获</font>**

1. ExceptionHandlerExceptionResolver 初始化时会解析 @ControllerAdvice 中的 @ExceptionHandler 方法
2. ExceptionHandlerExceptionResolver 会以类为单位，在该类首次处理异常时，解析此类的 @ExceptionHandler 方法
3. 以上两种 @ExceptionHandler 的解析结果都会缓存来避免重复解析

# 13 Tomcat异常处理
我们知道 @ExceptionHandler 只能处理发生在 mvc 流程中的异常，例如控制器内、拦截器内，那么如果是 Filter 出现了异常，如何进行处理呢？



在 Spring Boot 中，是这么实现的：

  1. 因为内嵌了 Tomcat 容器，因此可以配置 Tomcat 的错误页面，Filter 与 错误页面之间是通过请求转发跳转的，可以在这里做手脚

  2. 先通过 ErrorPageRegistrarBeanPostProcessor 这个后处理器配置错误页面地址，默认为 `/error` 也可以通过 `${server.error.path}` 进行配置

  3. 当 Filter 发生异常时，不会走 Spring 流程，但会走 Tomcat 的错误处理，于是就希望转发至 `/error` 这个地址

    - 当然，如果没有 @ExceptionHandler，那么最终也会走到 Tomcat 的错误处理

  4. Spring Boot 又提供了一个 BasicErrorController，它就是一个标准 @Controller，@RequestMapping 配置为 `/error`，所以处理异常的职责就又回到了 Spring

  5. 异常信息由于会被 Tomcat 放入 request 作用域，因此 BasicErrorController 里也能获取到

  6. 具体异常信息会由 DefaultErrorAttributes 封装好

  7. BasicErrorController 通过 Accept 头判断需要生成哪种 MediaType 的响应

    - 如果要的不是 text/html，走 MessageConverter 流程
    - 如果需要 text/html，走 mvc 流程，此时又分两种情况
    - 配置了 ErrorViewResolver，根据状态码去找 View
    - 没配置或没找到，用 BeanNameViewResolver 根据一个固定为 error 的名字找到 View，即所谓的 WhitelabelErrorView

>  _评价_
>
> 一个错误处理搞得这么复杂，就问恶心不？
>

**<font style="color:#E8323C;">错误页处理</font>**

关键代码

```java
package com.itheima.a32;

import org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext;
import org.springframework.web.method.HandlerMethod;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.servlet.HandlerExceptionResolver;
import org.springframework.web.servlet.mvc.method.RequestMappingInfo;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;

import java.lang.reflect.Field;
import java.util.List;

public class A32 {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        AnnotationConfigServletWebServerApplicationContext context =
                new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);
        RequestMappingHandlerMapping handlerMapping = context.getBean(RequestMappingHandlerMapping.class);
        handlerMapping.getHandlerMethods().forEach((RequestMappingInfo k, HandlerMethod v) -> {
            System.out.println("映射路径:" + k + "\t方法信息:" + v);
        });
    }

    /*
        学到了什么
            a. tomcat 的错误页处理手段
            b. Spring Boot 中 BasicErrorController 如何工作
     */

}
```



```java
package com.itheima.a32;

import org.springframework.boot.autoconfigure.web.ErrorProperties;
import org.springframework.boot.autoconfigure.web.servlet.DispatcherServletRegistrationBean;
import org.springframework.boot.autoconfigure.web.servlet.error.BasicErrorController;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.ErrorPage;
import org.springframework.boot.web.server.ErrorPageRegistrar;
import org.springframework.boot.web.server.ErrorPageRegistrarBeanPostProcessor;
import org.springframework.boot.web.servlet.error.DefaultErrorAttributes;
import org.springframework.boot.web.servlet.error.ErrorAttributes;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.View;
import org.springframework.web.servlet.ViewResolver;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter;
import org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping;
import org.springframework.web.servlet.view.BeanNameViewResolver;

import javax.servlet.RequestDispatcher;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.util.List;
import java.util.Map;

@Configuration
public class WebConfig {
    @Bean
    public TomcatServletWebServerFactory servletWebServerFactory() {
        return new TomcatServletWebServerFactory();
    }

    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean
    public DispatcherServletRegistrationBean servletRegistrationBean(DispatcherServlet dispatcherServlet) {
        DispatcherServletRegistrationBean registrationBean = new DispatcherServletRegistrationBean(dispatcherServlet, "/");
        registrationBean.setLoadOnStartup(1);
        return registrationBean;
    }

    @Bean // @RequestMapping
    public RequestMappingHandlerMapping requestMappingHandlerMapping() {
        return new RequestMappingHandlerMapping();
    }

    @Bean // 注意默认的 RequestMappingHandlerAdapter 不会带 jackson 转换器
    public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
        RequestMappingHandlerAdapter handlerAdapter = new RequestMappingHandlerAdapter();
        handlerAdapter.setMessageConverters(List.of(new MappingJackson2HttpMessageConverter()));
        return handlerAdapter;
    }

    @Bean // 修改了 Tomcat 服务器默认错误地址
    public ErrorPageRegistrar errorPageRegistrar() { // 出现错误，会使用请求转发 forward 跳转到 error 地址
        return webServerFactory -> webServerFactory.addErrorPages(new ErrorPage("/error"));
    }

    @Bean
    public ErrorPageRegistrarBeanPostProcessor errorPageRegistrarBeanPostProcessor() {
        return new ErrorPageRegistrarBeanPostProcessor();
    }

    @Controller
    public static class MyController {
        @RequestMapping("test")
        public ModelAndView test() {
            int i = 1 / 0;
            return null;
        }

        /*@RequestMapping("/error")
        @ResponseBody
        public Map<String, Object> error(HttpServletRequest request) {
            Throwable e = (Throwable) request.getAttribute(RequestDispatcher.ERROR_EXCEPTION);
            return Map.of("error", e.getMessage());
        }*/
    }

    @Bean
    public BasicErrorController basicErrorController() {
        ErrorProperties errorProperties = new ErrorProperties();
        errorProperties.setIncludeException(true);
        return new BasicErrorController(new DefaultErrorAttributes(), errorProperties);
    }

    @Bean
    public View error() {
        return new View() {
            @Override
            public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
                System.out.println(model);
                response.setContentType("text/html;charset=utf-8");
                response.getWriter().print("""
                        <h3>服务器内部错误</h3>
                        """);
            }
        };
    }

    @Bean
    public ViewResolver viewResolver() {
        return new BeanNameViewResolver();
    }
}
```

# 14 BeanNameUrlHandlerMapping 与 SimpleControllerHandlerAdapter
**<font style="color:#E8323C;">本组映射器和适配器</font>**

关键代码

```java
@Bean
public BeanNameUrlHandlerMapping beanNameUrlHandlerMapping() {
    return new BeanNameUrlHandlerMapping();
}

@Bean
public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
    return new SimpleControllerHandlerAdapter();
}

@Bean("/c3")
public Controller controller3() {
    return (request, response) -> {
        response.getWriter().print("this is c3");
        return null;
    };
}
```



**<font style="color:#E8323C;">收获</font>**

1. BeanNameUrlHandlerMapping，以 / 开头的 bean 的名字会被当作映射路径
2. 这些 bean 本身当作 handler，要求实现 Controller 接口
3. SimpleControllerHandlerAdapter，调用 handler
4. 模拟实现这组映射器和适配器

# 15 RouterFunctionMapping与HandlerFunctionAdapter
**<font style="color:#E8323C;">本组映射器和适配器</font>**

关键代码

```java
@Bean
public RouterFunctionMapping routerFunctionMapping() {
    return new RouterFunctionMapping();
}

@Bean
public HandlerFunctionAdapter handlerFunctionAdapter() {
    return new HandlerFunctionAdapter();
}

@Bean
public RouterFunction<ServerResponse> r1() {
    //           ⬇️映射条件   ⬇️handler
    return route(GET("/r1"), request -> ok().body("this is r1"));
}
```

**<font style="color:#E8323C;">收获</font>**

1. RouterFunctionMapping, 通过 RequestPredicate 条件映射
2. handler 要实现 HandlerFunction 接口
3. HandlerFunctionAdapter, 调用 handler

# 16 SimpleUrlHandlerMapping与HttpRequestHandlerAdapter
**<font style="color:#F5222D;">本组映射器和适配器</font>**

代码参考

```java
package org.springframework.boot.autoconfigure.web.servlet;

import org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext;
import org.springframework.web.HttpRequestHandler;
import org.springframework.web.servlet.function.HandlerFunction;

public class A35 {
    public static void main(String[] args) {
        AnnotationConfigServletWebServerApplicationContext context
                = new AnnotationConfigServletWebServerApplicationContext(WebConfig.class);

        /*
            学到了什么
            静态资源处理
                a. SimpleUrlHandlerMapping 映射路径
                b. ResourceHttpRequestHandler 作为静态资源 handler
                c. HttpRequestHandlerAdapter, 调用此 handler

            欢迎页处理
                a. WelcomePageHandlerMapping, 映射欢迎页(即只映射 '/')
                    - 它内置了 handler ParameterizableViewController 作用是不执行逻辑, 仅根据视图名找视图
                    - 视图名固定为 forward:index.html       /**
                b. SimpleControllerHandlerAdapter, 调用 handler
                    - 转发至 /index.html
                    - 处理 /index.html 又会走上面的静态资源处理流程

        */

        /*
            小结
                a. HandlerMapping 负责建立请求与控制器之间的映射关系
                    - RequestMappingHandlerMapping (与 @RequestMapping 匹配)
                    - WelcomePageHandlerMapping    (/)
                    - BeanNameUrlHandlerMapping    (与 bean 的名字匹配 以 / 开头)
                    - RouterFunctionMapping        (函数式 RequestPredicate, HandlerFunction)
                    - SimpleUrlHandlerMapping      (静态资源 通配符 /** /img/**)
                    之间也会有顺序问题, boot 中默认顺序如上
                b. HandlerAdapter 负责实现对各种各样的 handler 的适配调用
                    - RequestMappingHandlerAdapter 处理：@RequestMapping 方法
                           参数解析器、返回值处理器体现了组合模式
                    - SimpleControllerHandlerAdapter 处理：Controller 接口
                    - HandlerFunctionAdapter 处理：HandlerFunction 函数式接口
                    - HttpRequestHandlerAdapter 处理：HttpRequestHandler 接口 (静态资源处理)
                    这也是典型适配器模式体现
                c. ResourceHttpRequestHandler.setResourceResolvers 这是典型责任链模式体现
         */
    }
}
```

```java
package org.springframework.boot.autoconfigure.web.servlet;

import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.cache.concurrent.ConcurrentMapCache;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;
import org.springframework.web.servlet.DispatcherServlet;
import org.springframework.web.servlet.handler.SimpleUrlHandlerMapping;
import org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter;
import org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter;
import org.springframework.web.servlet.resource.CachingResourceResolver;
import org.springframework.web.servlet.resource.EncodedResourceResolver;
import org.springframework.web.servlet.resource.PathResourceResolver;
import org.springframework.web.servlet.resource.ResourceHttpRequestHandler;

import javax.annotation.PostConstruct;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.util.List;
import java.util.Map;
import java.util.zip.GZIPOutputStream;

@Configuration
public class WebConfig {
    @Bean // ⬅️内嵌 web 容器工厂
    public TomcatServletWebServerFactory servletWebServerFactory() {
        return new TomcatServletWebServerFactory(8080);
    }

    @Bean // ⬅️创建 DispatcherServlet
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }

    @Bean // ⬅️注册 DispatcherServlet, Spring MVC 的入口
    public DispatcherServletRegistrationBean servletRegistrationBean(DispatcherServlet dispatcherServlet) {
        return new DispatcherServletRegistrationBean(dispatcherServlet, "/");
    }

    @Bean
    public SimpleUrlHandlerMapping simpleUrlHandlerMapping(ApplicationContext context) {
        SimpleUrlHandlerMapping handlerMapping = new SimpleUrlHandlerMapping();
        Map<String, ResourceHttpRequestHandler> map = context.getBeansOfType(ResourceHttpRequestHandler.class);
        handlerMapping.setUrlMap(map);
        System.out.println(map);
        return handlerMapping;
    }

    @Bean
    public HttpRequestHandlerAdapter httpRequestHandlerAdapter() {
        return new HttpRequestHandlerAdapter();
    }

    /*
        /index.html
        /r1.html
        /r2.html

        /**
     */

    @Bean("/**")
    public ResourceHttpRequestHandler handler1() {
        ResourceHttpRequestHandler handler = new ResourceHttpRequestHandler();
        handler.setLocations(List.of(new ClassPathResource("static/")));
        handler.setResourceResolvers(List.of(
                new CachingResourceResolver(new ConcurrentMapCache("cache1")),
                new EncodedResourceResolver(),
                new PathResourceResolver()
        ));
        return handler;
    }

    /*
        /img/1.jpg
        /img/2.jpg
        /img/3.jpg

        /img/**
     */

    @Bean("/img/**")
    public ResourceHttpRequestHandler handler2() {
        ResourceHttpRequestHandler handler = new ResourceHttpRequestHandler();
        handler.setLocations(List.of(new ClassPathResource("images/")));
        return handler;
    }

    @Bean
    public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext context) {
        Resource resource = context.getResource("classpath:static/index.html");
        return new WelcomePageHandlerMapping(null, context, resource, "/**");
        // Controller 接口
    }

    @Bean
    public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
        return new SimpleControllerHandlerAdapter();
    }

    @PostConstruct
    @SuppressWarnings("all")
    public void initGzip() throws IOException {
        Resource resource = new ClassPathResource("static");
        File dir = resource.getFile();
        for (File file : dir.listFiles(pathname -> pathname.getName().endsWith(".html"))) {
            System.out.println(file);
            try (FileInputStream fis = new FileInputStream(file); GZIPOutputStream fos = new GZIPOutputStream(new FileOutputStream(file.getAbsoluteFile() + ".gz"))) {
                byte[] bytes = new byte[8 * 1024];
                int len;
                while ((len = fis.read(bytes)) != -1) {
                    fos.write(bytes, 0, len);
                }
            }
        }
    }
}
```

**<font style="color:#E8323C;">收获</font>**

1. SimpleUrlHandlerMapping 不会在初始化时收集映射信息，需要手动收集
2. SimpleUrlHandlerMapping 映射路径
3. ResourceHttpRequestHandler 作为静态资源 handler
4. HttpRequestHandlerAdapter, 调用此 handler

**<font style="color:#F5222D;">静态资源解析优化</font>**

关键代码

```java
@Bean("/**")
public ResourceHttpRequestHandler handler1() {
    ResourceHttpRequestHandler handler = new ResourceHttpRequestHandler();
    handler.setLocations(List.of(new ClassPathResource("static/")));
    handler.setResourceResolvers(List.of(
        	// ⬇️缓存优化
            new CachingResourceResolver(new ConcurrentMapCache("cache1")),
        	// ⬇️压缩优化
            new EncodedResourceResolver(),
        	// ⬇️原始资源解析
            new PathResourceResolver()
    ));
    return handler;
}
```

**<font style="color:#F5222D;">收获</font>**

1. 责任链模式体现
2. 压缩文件需要手动生成







**<font style="color:#F5222D;">欢迎页</font>**

关键代码



```java
@Bean
public WelcomePageHandlerMapping welcomePageHandlerMapping(ApplicationContext context) {
    Resource resource = context.getResource("classpath:static/index.html");
    return new WelcomePageHandlerMapping(null, context, resource, "/**");
}

@Bean
public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
    return new SimpleControllerHandlerAdapter();
}
```

**<font style="color:#F5222D;">收获</font>**

1. 欢迎页支持静态欢迎页与动态欢迎页
2. WelcomePageHandlerMapping 映射欢迎页（即只映射 '/'）
    - 它内置的 handler ParameterizableViewController 作用是不执行逻辑，仅根据视图名找视图
    - 视图名固定为 forward:index.html
3. SimpleControllerHandlerAdapter, 调用 handler
    - 转发至 /index.html
    - 处理 /index.html 又会走上面的静态资源处理流程



**<font style="color:#F5222D;">映射器与适配器小结</font>**

1. HandlerMapping 负责建立请求与控制器之间的映射关系
    - RequestMappingHandlerMapping(与 @RequestMapping 匹配)
    - WelcomePageHandlerMapping (/)
    - BeanNameUrlHandlerMapping (与 bean 的名字匹配 以 / 开头)
    - RouterFunctionMapping (函数式 RequestPredicate, HandlerFunction)
    -  SimpleUrlHandlerMapping(静态资源 通配符 /** /img/**)
    - 之间也会有顺序问题, boot 中默认顺序如上
2. HandlerAdapter 负责实现对各种各样的 handler 的适配调用
    - RequestMappingHandlerAdapter 处理：@RequestMapping 方法
    - 参数解析器、返回值处理器体现了组合模式
    - SimpleControllerHandlerAdapter 处理：Controller 接口
    - HandlerFunctionAdapter 处理：HandlerFunction 函数式接口
    - HttpRequestHandlerAdapter 处理：HttpRequestHandler 接口 (静态资源处理)
    - 这也是典型适配器模式体现

# 17 mvc 处理流程
当浏览器发送一个请求 `http://localhost:8080/hello` 后，请求到达服务器，其处理流程是：

1. 服务器提供了`DispatcherServlet`，它使用的是标准`Servlet`技术
    - 路径：默认映射路径为 `/`，即会匹配到所有请求 URL，可作为请求的统一入口，也被称之为**前控制器**
        * `jsp` 不会匹配到 `DispatcherServlet`
        * 其它有路径的 `Servlet` 匹配优先级也高于 `DispatcherServlet`
    - 创建：在 Boot 中，由 `DispatcherServletAutoConfiguration` 这个自动配置类提供 `DispatcherServlet` 的 `bean`
    - 初始化：`DispatcherServlet` 初始化时会优先到容器里寻找各种组件，作为它的成员变量
        * `HandlerMapping`，初始化时记录映射关系
        * `HandlerAdapter`，初始化时准备参数解析器、返回值处理器、消息转换器
        * `HandlerExceptionResolver`，初始化时准备参数解析器、返回值处理器、消息转换器
        * `ViewResolver`

2.  `DispatcherServlet` 会利用 `RequestMappingHandlerMapping`查找控制器方法

    - 例如根据 `/hello` 路径找到 `@RequestMapping("/hello")` 对应的控制器方法
    - 控制器方法会被封装为 `HandlerMethod` 对象，并结合匹配到的拦截器一起返回给 `DispatcherServlet`
    - `HandlerMethod` 和拦截器合在一起称为 `HandlerExecutionChain`（调用链）对象
3. `DispatcherServlet`接下来会：
    1. 调用拦截器的 `preHandle` 方法
    2. `RequestMappingHandlerAdapter` 调用`handle` 方法，准备数据绑定工厂、模型工厂、`ModelAndViewContainer` 、将 `HandlerMethod` 完善为 `ServletInvocableHandlerMethod`
        * `@ControllerAdvice` 全局增强点1️⃣：补充模型数据
        * `@ControllerAdvice` 全局增强点2️⃣：补充自定义类型转换器
        * 使用 `HandlerMethodArgumentResolver` 准备参数
            + `@ControllerAdvice` 全局增强点3️⃣：`RequestBody` 增强
        * 调用 `ServletInvocableHandlerMethod` 
        * 使用 `HandlerMethodReturnValueHandler` 处理返回值
            + `@ControllerAdvice` 全局增强点4️⃣：`ResponseBody` 增强
        * 根据 `ModelAndViewContainer` 获取 `ModelAndView`
            + 如果返回的 `ModelAndView` 为 null，不走第 4 步视图解析及渲染流程
                - 例如，有的返回值处理器调用了 `HttpMessageConverter` 来将结果转换为 JSON，这时 `ModelAndView` 就为 null
            + 如果返回的 `ModelAndView` 不为 null，会在第 4 步走视图解析及渲染流程
    3. 调用拦截器的 `postHandle` 方法
    4. 处理异常或视图渲染
        * 如果 1~3 出现异常，走 `ExceptionHandlerExceptionResolver` 处理异常流程
            +  `@ControllerAdvice` 全局增强点5️⃣：`@ExceptionHandler` 异常处理
        * 正常，走视图解析及渲染流程
    5. 调用拦截器的 `afterCompletion` 方法





