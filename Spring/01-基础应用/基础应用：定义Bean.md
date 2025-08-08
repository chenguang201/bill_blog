# 1 用id和class定义一个bean


基于xml创建一个spring的bean

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="order" class="com.ali.testspring.Order">
    </bean>
</beans>
```

其中id是一个字符串，用于表示bean，是唯一的，class是类的全限定名。



在创建一个`services.xml`和`daos.xml`

  
services.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <!-- services -->

    <bean id="service" class="com.ali.testspring.Service">
        <property name="dao1" ref="dao1"/>
        <property name="dao2" ref="dao2"/>
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for services go here -->

</beans>
```



daos.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="dao1"
          class="com.ali.testspring.Dao1">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <bean id="dao2" class="com.ali.testspring.Dao2">
        <!-- additional collaborators and configuration for this bean go here -->
    </bean>

    <!-- more bean definitions for data access objects go here -->

</beans>
```



在测试类里：

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");

System.out.println(context.getBean("dao1"));
System.out.println(context.getBean("dao2"));
System.out.println(context.getBean("service"));
```

我们发现，虽然这三个bean不在一个xml里面，但是service仍然是可以引用dao1和dao2的。



# 2 用factory-method定义一个bean


看第一个示例

在ClientService中定义了一个静态工厂方法，返回一个clientService对象。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="clientService" class="com.ali.testspring.ClientService" factory-method="createInstance">
    </bean>
</beans>
```



```java
public class ClientService {
    private static ClientService clientService = new ClientService();
    private ClientService() {}

    public static ClientService createInstance() {
        return clientService;
    }
}
```



第二个示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userService" class="com.ali.testspring.UserService">
        <!-- inject any dependencies required by this locator bean -->
    </bean>

    <!-- the bean to be created via the factory bean -->
    <bean id="userDao"
          factory-bean="userService"
          factory-method="createUserDao"/>

</beans>
```

```java
public class UserService {
    private static  UserDao userDao=new UserDao();
    private UserDao createUserDao(){
        return userDao;
    }
}
```

