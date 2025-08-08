# 1 获取Class对象
获取一个类对应的Class对象，有以下三种方式

+ 使用Class的forName方法，该方法需要一个字符串参数，该字符串是该类的全限定名
+ 调用该类的class属性来获取该类对应的Class对象
+ 调用该类某个对象的getClass()方法，该类是Object类的方法，所有的java对象都是可以调用的



具体的代码如下：

```java
public class Hello {

    public static void main(String[] args) throws ClassNotFoundException {
        Class<?> aClass = Class.forName("com.ali.reflect.Hello");
        Class<Hello> helloClass = Hello.class;
        Class<? extends Hello> aClass1 = new Hello().getClass();

        System.out.println(aClass);        //class com.ali.reflect.Hello
        System.out.println(helloClass);    //class com.ali.reflect.Hello
        System.out.println(aClass1);      //class com.ali.reflect.Hello
    }
}
```



# 2 从Class类中获取信息
从类对应的Class对象可以获取哪些信息呢？主要如下：

+ 类所包含的构造方法
+ 类所包含的普通方法
+ 类上用到的annotation
+ 类所包含的内部类
+ 类所在的外部类
+ 类所实现的接口
+ 类所继承的父类
+ 类的修饰符、所在包、类名等基本信息
+ 判断该类是否为接口、枚举、注解等类型



# 3 使用反射创建对象并操作对象
## 3.1 创建对象
使用Class对象来生成类的对象一般需要以下三步：

1. 获取该类的Class对象
2. 利用Class对象的getConstructor方法类获取指定的构造器
3. 利用Constructor的newIntance方法来创建一个类的对象



代码如下：

```java
public class Hello {
    public static void main(String[] args) throws {
        Class<?> aClass = Class.forName("com.ali.reflect.Hello");  //第一步
        Constructor<?> constructor = aClass.getConstructor();     //第二步 
        Object o = constructor.newInstance();                    //第三步
        System.out.println(o);                                  
    }
}
```

## 3.2  操作对象调用方法
使用Class对象来调用类里面对象的方法一般需要以下几步：

1. 获取该类的Class对象
2. 利用Class对象的getMethod方法获取到指定的方法，getMethod方法返回的是一个Method对象
3. 调用Method对象的invoke(Object obj,Object... args)方法，invoke方法的第一个参数obj是该方法的主调，后面的arg是执行方法所需要的实参

具体代码如下：

```java
public class Hello {

    public void info(){
        System.out.println("info");
    }

    public static void main(String[] args) {
        Class<?> aClass = Class.forName("com.ali.reflect.Hello");          //第一步，获取Class对象
        Method info = aClass.getMethod("info");                            //第二步，根据getMethod获取Method对象
        System.out.println(info);
        info.invoke(new Hello());                                          //第三步，调用invoke方法，主调者是Hello的对象，info方法没有参数，所以只传入一个参数
    }
}
```

## 3.3 访问成员变量
使用Class对象给类对象里面的成员变量设置一般需要如下几步：

1. 获取类对象的Class对象
2. 根据getDeclaredField方法获取到类对象的具体某个属性，getDeclaredField的返回值是一个Field对象
3. 调用Field对象的setAccessible方法，并设置为ture，其意义是根据反射访问该成员变量时取消访问权限检查
4. 调用get(Object obj)或者set(Object obj，xxx val)方法，get方法的意义是获取obj对象的该成员变量的值，set方法的意义是将obj对象的该成员变量设置为val

具体代码如下：

```java
public class Hello {

    private String name;

    public static void main(String[] args) {
        Class<?> aClass = Class.forName("com.ali.reflect.Hello");                    //第一步，获取Class对象
        Hello hello = new Hello();          
        Field name = aClass.getDeclaredField("name");                                //第二步，获取到name属性对象的Field对象
        name.setAccessible(true);                                                    //第三步，取消访问权限检查
        name.set(hello,"jack");                                                      //第四步，将hello对象的name属性设置为jack
        System.out.println(hello.name);
    }
}
```



## 3.4 利用反射生成JDK动态代理
java主要利用reflect包下的Proxy类和InvocationHandler接口来生成jdk动态代理类或动态代理对象

Proxy提供了两个方法来创建动态代理类和动态代理对象：

+  getProxyClass 创建一个动态代理类所对应的Class对象， 
    - 该代理类将实现第二个参数interfaces指定的多个接口。
    - 第一个参数ClassLoader指定生成动态代理类的类加载器
+  newProxyInstance 
    - 直接生成一个动态代理类对象
    - 该代理类将实现第二个参数interfaces指定的多个接口。
    - 指定代理对象里面的每一个方法时都会被替换执行InvacationHandler对象的invoke方法，也就是说代理对象里面有多少个方法，invoke方法就会被执行多少遍。



代码如下：

```java
//定义一个接口，接口里面有两个方法
public interface Persion {
    void walk();
    void sayHello(String name);
}

/**
  *自定义一个InvocationHandler，实现里面的invoke方法，将来生成的代理对象里面会实现接口Persion里面的两个方法方法，
  *但是在执行代理方法的时候，其实是在执行下面的invoke方法
*/
public class MyInvokationHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        //第一个参数proxy：代表动态代理对象
        //第二个参数method：代表正在执行的方法
        //第三个参数args：代表调用目标方法时传入的实参
        System.out.println("------正在执行的方法-----："+method);
        if (args != null){
            System.out.println("下面执行该方法时传入的实参为：");
            for (Object o:args){
                System.out.println(o);
            }
        }else {
            System.out.println("调用该方法没有实参");
        }
        System.out.println("------------------------");
        return null;
    }
}

public class ProxyTest {

    public static void main(String[] args) {
		//创建一个InvocationHandler对象
        InvocationHandler invocationHandler=new MyInvokationHandler();
		//使用指定的InvocationHandler来生成一个动态代理对象
        Persion persion =(Persion) Proxy.newProxyInstance(Persion.class.getClassLoader(), new Class[]{Persion.class}, invocationHandler);
		调用动态代理对象里面的walk方法和sayHello方法
        persion.walk();
        persion.sayHello("hello java");
    }
}

/*

下面为MyInvokationHandler的打印：


------正在执行的方法-----：public abstract void com.ali.proxy.Persion.walk()
调用该方法没有实参
------------------------
------正在执行的方法-----：public abstract void com.ali.proxy.Persion.sayHello(java.lang.String)
下面执行该方法时传入的实参为：
hello java
------------------------

*/
```

从上面的打印可以看出，不管是执行代理对象的walk方法，还是执行代理对象的sayHello方法，实际上都是执行MyInvokationHandler对象的invoke方法。

## 3.5 动态代理和AOP
代码如下：

```java
//定义一个接口
public interface Dog {
    void info();
    void run();
}

//定义一个实现类，实现Dog接口，也就是本文中需要被代理的对象
public class BigDog implements Dog{
    @Override
    public void info() {
        System.out.println("我很大");
    }

    @Override
    public void run() {
        System.out.println("我跑的快");
    }
}

public class MyInvocationHandle implements InvocationHandler {
    //需要被代理的对象
    private Object target;
    public void setTarget(Object target){
        this.target=target;
    }

    //执行动态代理的所有方法，都会被替换成执行下面的invoke方法
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        System.out.println("-----我是在调用之前执行的-----");

        //这一行很关键，已target作为主调来执行menthod的invoke方法，target就是被代理对象，也就是执行被代理对象里面的方法
        Object result = method.invoke(target, args);

        System.out.println("-----我是在调用之后执行的-----");
        return result;
    }
}

public class Test {
    public static void main(String[] args) {
        //创建一个InvocationHandler对象
        MyInvocationHandle myInvocationHandle=new MyInvocationHandle();
        Dog dog=new BigDog();
        //为InvocationHandler对象设置target，也就是设置被代理对象
        myInvocationHandle.setTarget(dog);

        //生成一个代理对象
        Dog proDog=(Dog) Proxy.newProxyInstance(dog.getClass().getClassLoader(),dog.getClass().getInterfaces(),myInvocationHandle);
        proDog.info();
        proDog.run();

        /**
         * 很明显我们发现一个问题，JDK的动态代理只能为接口创建代理对象，换句话说
         * 也就是说，被代理的对象必须要作为一个接口的实现类，在这种情况下，我们才能使用JDk动态代理
         */
    }

/* 下面这些为代码中的打印

-----我是在调用之前执行的-----
我很大
-----我是在调用之后执行的-----
-----我是在调用之前执行的-----
我跑的快
-----我是在调用之后执行的-----

*/
```

