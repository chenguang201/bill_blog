**笔记来源：**[**黑马程序员深入学习Java并发编程，JUC并发编程全套教程**](https://www.bilibili.com/video/BV16J411h7Rd/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

------



## 1 创建和运行线程
### 1.1 使用Thread  
```java
// 创建线程对象
Thread t = new Thread() {
    public void run() {
    // 要执行的任务
    }
};
// 启动线程
t.start();
```

例如：  

```java
// 构造方法的参数是给线程指定名字，推荐
Thread t1 = new Thread("t1") {
    @Override
    // run 方法内实现了要执行的任务
    public void run() {
        log.debug("hello");
    }
};
t1.start();
```

输出  

```java
19:19:00 [t1] c.ThreadStarter - hello
```



### 1.2 使用Runnable
把【线程】和【任务】（要执行的代码）分开  

+ Thread 代表线程  
+ Runnable 可运行的任务（线程要执行的代码）  

```java
Runnable runnable = new Runnable() {
    public void run(){
    // 要执行的任务
    }
};
// 创建线程对象
Thread t = new Thread( runnable );
// 启动线程
t.start();
```

```java
// 创建任务对象
Runnable task2 = new Runnable() {
    @Override
    public void run() { 
        log.debug("hello");
    } 
};
// 参数1 是任务对象; 参数2 是线程名字，推荐 
Thread t2 = new Thread(task2, "t2"); t2.start();
```

输出  

```plain
19:19:00 [t2] c.ThreadStarter - hello
```

Java 8 以后可以使用 lambda 精简代码  

```java
// 创建任务对象
Runnable task2 = () -> log.debug("hello");
// 参数1 是任务对象; 参数2 是线程名字，推荐 
Thread t2 = new Thread(task2, "t2"); 
t2.start();
```

Thread 与 Runnable的关系：Runnable是一个接口，接口中只有一个 run 方法，Thread 类是Runnable 接口的一个实现类，Thread 类中有一个构造方法，构造方法就是的一个入参就是 Runnable 对象。


小结  

+ 第一种方式是把线程和任务合并在了一起，第二种方式是把线程和任务分开了
+ 用Runnable更容易与线程池等高级 API 配合
+ 用Runnable让任务类脱离了 Thread 继承体系，更灵活

### 1.3 FutureTask配合Thread  
FutureTask 能够接收 Callable 类型的参数，用来处理有返回结果的情况  

```java
// 创建任务对象
FutureTask<Integer> task3 = new FutureTask<>(() -> {
    log.debug("hello");
    return 100; 
});
// 参数1 是任务对象; 参数2 是线程名字，推荐 
new Thread(task3, "t3").start();
// 主线程阻塞，同步等待 task 执行完毕的结果 
Integer result = task3.get(); 
log.debug("结果是:{}", result);
```

输出  

```java
 19:22:27 [t3] c.ThreadStarter - hello 
 19:22:27 [main] c.ThreadStarter - 结果是:100
```

FutureTask 也是 Runnable 接口的一个实现类，在 重写的 run 方法里面调用了一个call 方法。所以需要我们提供一个 Callable 的接口，重新 call 方法  

```java
public void run() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();  //调用我们重写的call方法
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

## 2 观察多个线程同时运行
主要是理解

+ 交替执行
+ 谁先谁后，不由我们控制

## 3 查看进程线程的方法
**windows**

>```java
>//任务管理器可以查看进程和线程数，也可以用来杀死进程
>tasklist //查看进程
>taskkill //杀死进程
>```

**linux**

>```powershell
>ps -fe 						#查看所有进程
>ps -fT -p <PID> 			#查看某个进程（PID）的所有线程
>kill 						#杀死进程
>top 						#按大写 H 切换是否显示线程
>top -H -p <PID>  			#查看某个进程（PID）的所有线程
>```

**Java**

>```java
>jps 			//命令查看所有 Java 进程
>jstack <PID>  	//查看某个 Java 进程（PID）的所有线程状态
>jconsole  		//来查看某个 Java 进程中线程的运行情况（图形界面）
>```

## 4 原理之线程运行
**栈与栈帧**

Java Virtual Machine Stacks （Java 虚拟机栈）  
我们都知道 JVM 中由堆、栈、方法区所组成，其中栈内存是给谁用的呢？其实就是线程，每个线程启动后，虚拟机就会为其分配一块栈内存。

+ 每个栈由多个栈帧（Frame）组成，对应着每次方法调用时所占用的内存
+ 每个线程只能有一个活动栈帧，对应着当前正在执行的那个方法



**线程上下文切换（Thread Context Switch）**  
因为以下一些原因导致 cpu 不再执行当前的线程，转而执行另一个线程的代码

+ 线程的 cpu 时间片用完
+ 垃圾回收
+ 有更高优先级的线程需要运行
+ 线程自己调用了 sleep、yield、wait、join、park、synchronized、lock 等方法

当 Context Switch 发生时，需要由操作系统保存当前线程的状态，并恢复另一个线程的状态，Java 中对应的概念就是程序计数器（Program Counter Register），它的作用是记住下一条 jvm 指令的执行地址，是线程私有的

+ 状态包括程序计数器、虚拟机栈中每个栈帧的信息，如局部变量、操作数栈、返回地址等
+ Context Switch 频繁发生会影响性能



## 5 常见方法
| 方法名                | static | 功能说明                              | 注意                                       |
| ------------------ | ------ | --------------------------------- | ---------------------------------------- |
| `start()`          |        | 启动一个新线程，在新的线程运行 run 方法中的代码        | start 方法只是让线程进入就绪，里面代码不一定立刻运行（CPU 的时间片还没分给它）。每个线程对象的start方法只能调用一次，如果调用了多次会出现IllegalThreadStateException |
| `run()`            |        | 新线程启动后会调用的方法                      | 如果在构造 Thread 对象时传递了 Runnable 参数，则线程启动后会调用 Runnable 中的 run 方法，否则默认不执行任何操作。但可以创建 Thread 的子类对象，来覆盖默认行为 |
| `join()`           |        | 等待线程运行结束                          |                                          |
| `join(long n)`     |        | 等待线程运行结束,最多等待 n 毫秒                |                                          |
| `getId()`          |        | 获取线程长整型的id                        | id 唯一                                    |
| `getName()`        |        | 获取线程名                             |                                          |
| `setName(String)`  |        | 修改线程名                             |                                          |
| `getPriority()`    |        | 获取线程优先级                           |                                          |
| `setPriority(int)` |        | 修改线程优先级                           | java中规定线程优先级是1~10 的整数，较大的优先级能提高该线程被 CPU 调度的机率 |
| `getState()`       |        | 获取线程状态                            | Java中线程状态是用 6 个 enum 表示，分别为：`NEW` `RUNNABLE` `BLOCKED` `WAITING` `TIMED_WAITING` `TERMINATED` |
| `isInterrupted()`  |        | 判断是否被打断                           | 不会清除打断标记                                 |
| `isAlive()`        |        | 线程是否存活（还没有运行完毕）                   |                                          |
| `interrupt()`      |        | 打断线程                              | 如果被打断线程正在 sleep，wait，join 会导致被打断的线程抛出 InterruptedException，并清除打断标记；如果打断的正在运行的线程，则会设置打断标记；park 的线程被打断，也会设置打断标记 |
| `interrupted()`    | static | 判断当前线程是否被打断                       | 会清除打断标记                                  |
| `currentThread()`  | static | 获取当前正在执行的线程                       |                                          |
| `sleep(long n)`    | static | 让当前执行的线程休眠n毫秒，休眠时让出 cpu 的时间片给其它线程 |                                          |
| `yield()`          | static | 提示线程调度器让出当前线程对CPU的使用              | 主要是为了测试和调试                               |


## 6 start与run
**调用run**

```java
public static void main(String[] args) { 
    Thread t1 = new Thread("t1") {
        @Override
        public void run() { 
            log.debug(Thread.currentThread().getName()); 
            FileReader.read(Constants.MP4_FULL_PATH);
        } 
    };
    t1.run();
    log.debug("do other things ..."); 
}
```

输出

```plain
19:39:14 [main] c.TestStart - main
19:39:14 [main] c.FileReader - read [1.mp4] start ...
19:39:18 [main] c.FileReader - read [1.mp4] end ... cost: 4227 ms 
19:39:18 [main] c.TestStart - do other things ...
```

程序仍在 main 线程运行， `FileReader.read()`方法调用还是同步的

**调用start**  
将上述代码的 `t1.run()`改为

```java
t1.start();
```

输出

```plain
19:41:30 [main] c.TestStart - do other things ...
19:41:30 [t1] c.TestStart - t1
19:41:30 [t1] c.FileReader - read [1.mp4] start ...
19:41:35 [t1] c.FileReader - read [1.mp4] end ... cost: 4542 ms
```

程序在 t1 线程运行， `FileReader.read()`方法调用是异步的

**小结**：直接调用 run 是在主线程中执行了run，没有启动新的线程。使用start是启动新的线程，通过新的线程间接执行 run 中的代码。

## 7 sleep 与 yield
**sleep：**

>1. 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态（阻塞）
>2. 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 InterruptedException
>3. 睡眠结束后的线程未必会立刻得到执行
>4. 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性

**yield：**

>1. 调用 yield 会让当前线程从 Running 进入 Runnable 就绪状态，然后调度执行其它线程
>2. 具体的实现依赖于操作系统的任务调度器




**线程优先级：**

+ 线程优先级会提示（hint）调度器优先调度该线程，但它仅仅是一个提示，调度器可以忽略它

+ 如果 cpu 比较忙，那么优先级高的线程会获得更多的时间片，但 cpu 闲时，优先级几乎没作用

  ```java
  Runnable task1 = () -> {
      int count = 0;
      for (;;) {
          System.out.println("---->1 " + count++);
      }
  };

  Runnable task2 = () -> {
      int count = 0;
      for (;;) {
          Thread.yield();
          System.out.println("---->2 " + count++);
      }
  };
  Thread t1 = new Thread(task1, "t1");
  Thread t2 = new Thread(task2, "t2");
  t1.setPriority(Thread.MIN_PRIORITY); 
  t2.setPriority(Thread.MAX_PRIORITY); 
  t1.start();
  t2.start();
  ```

  ​

## 8 join 方法详解
join 的简单理解就是：把指定的线程加入到当前线程。

### 8.1 为什么需要join
下面的代码执行，打印`r`是什么？

```java
static int r = 0;

public static void main(String[] args) throws InterruptedException {
    test1(); 
}

private static void test1() throws InterruptedException { 
    log.debug("开始");
    Thread t1 = new Thread(() -> {
        log.debug("开始"); 
        sleep(1000); 
        log.debug("结束"); 
        r = 10;
    });
    t1.start(); 
    log.debug("结果为:{}", r); 
    log.debug("结束");
}
```

输出：

```java
17:09:13.094 [main] DEBUG com.myspring.MyService1 - 开始
17:09:13.097 [Thread-0] DEBUG com.myspring.MyService1 - 开始
17:09:13.096 [main] DEBUG com.myspring.MyService1 - 结果为:0
17:09:13.098 [main] DEBUG com.myspring.MyService1 - 结束
17:09:14.101 [Thread-0] DEBUG com.myspring.MyService1 - 结束
```

分析

+ 因为主线程和线程 t1 是并行执行的，t1 线程需要 1 秒之后才能算出 r=10
+ 而主线程一开始就要打印 r 的结果，所以只能打印出 r=0

解决方法：用 sleep 行不行？为什么？

```java
    private static void test1() throws InterruptedException {
        log.debug("开始");
        Thread t1 = new Thread(() -> {
            log.debug("开始");
            try {
                sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("结束");
            r = 10;
        });
        t1.start();
        Thread.sleep(2000);
        log.debug("结果为:{}", r);
        log.debug("结束");
    }
```

输出：

```java
17:13:13.978 [main] DEBUG com.myspring.MyService1 - 开始
17:13:13.982 [Thread-0] DEBUG com.myspring.MyService1 - 开始
17:13:14.985 [Thread-0] DEBUG com.myspring.MyService1 - 结束
17:13:15.986 [main] DEBUG com.myspring.MyService1 - 结果为:10
17:13:15.990 [main] DEBUG com.myspring.MyService1 - 结束
```

解决方法：用 join，加在`t1.start()` 之后即可

```java
    private static void test1() throws InterruptedException {
        log.debug("开始");
        Thread t1 = new Thread(() -> {
            log.debug("开始");
            try {
                sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("结束");
            r = 10;
        });
        t1.start();
        t1.join();
        log.debug("结果为:{}", r);
        log.debug("结束");
    }
```

在这里，在主线程中执行了 t1 线程的 join 方法，简单理解就是将 t1 线程设置为当前线程，直到 t1 线程执行结束，才会运行主线程，所以下面的打印可以看到，join 后面的输入日志都是在 t1 线程执行完成之后打印的。

输出：

```java
17:11:05.344 [main] DEBUG com.myspring.MyService1 - 开始
17:11:05.348 [Thread-0] DEBUG com.myspring.MyService1 - 开始
17:11:06.352 [Thread-0] DEBUG com.myspring.MyService1 - 结束
17:11:06.353 [main] DEBUG com.myspring.MyService1 - 结果为:10
17:11:06.355 [main] DEBUG com.myspring.MyService1 - 结束
```

以调用方角度来讲，如果需要等待结果返回，才能继续运行就是同步；如果不需要等待结果返回，就能继续运行就是异步

![](https://cdn.nlark.com/yuque/__mermaid_v3/d43213e45ed11adadfcd1056882a84c6.svg)

### 8.2 等待多个结果
问，下面代码 cost 大约多少秒？

```java
static int r1 = 0;
static int r2 = 0;
public static void main(String[] args) throws InterruptedException {
    test2();
}
private static void test2() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        sleep(1);
        r1 = 10;
    });
    Thread t2 = new Thread(() -> {
        sleep(2);
        r2 = 20;
    });
    long start = System.currentTimeMillis();
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    long end = System.currentTimeMillis();
    log.debug("r1: {} r2: {} cost: {}", r1, r2, end - start);
}
```

输出：

```java
17:25:00.132 [main] DEBUG com.myspring.MyService1 - r1: 10 r2: 20 cost: 2001
```

分析如下

+ 第一个 join：等待 t1 时, t2 并没有停止, 而在运行
+ 第二个 join：1s 后，执行到此，t2 也运行了 1s，因此也只需再等待 1s

```mermaid
graph TB
    a("main") --> b("t1.join")
    b -->c("t2.join-仅需等待一秒")
    d("r=10")
    e("r=20")

    a --> t1.start
    t1.start --> |1s后| d
    d -->|t1终止| b

    a --> t2.start
    t2.start -->|2s后| e
    e-->|t2终止| c
```



如果颠倒两个 join 呢？

![](https://cdn.nlark.com/yuque/__mermaid_v3/6d6bf9be823e108fc3e8f8fb2b789e82.svg)

最终都是输出

```plain
20:45:43.239 [main] c.TestJoin - r1: 10 r2: 20 cost: 2005
```

### 8.3 有时效的 join
**等够时间**

```java
static int r1 = 0;
static int r2 = 0;
public static void main(String[] args) throws InterruptedException {
    test3();
}
public static void test3() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        sleep(1);
        r1 = 10;
    });
    long start = System.currentTimeMillis();
    t1.start();
    // 线程执行结束会导致 join 结束
    t1.join(1500);
    long end = System.currentTimeMillis();
    log.debug("r1: {} r2: {} cost: {}", r1, r2, end - start);
}
```

输出

```plain
20:48:01.320 [main] c.TestJoin - r1: 10 r2: 0 cost: 1010
```



**没等够时间**

```java
static int r1 = 0;
static int r2 = 0;
public static void main(String[] args) throws InterruptedException {
    test3();
}
public static void test3() throws InterruptedException {
    Thread t1 = new Thread(() -> {
        sleep(2);
        r1 = 10;
    });
    long start = System.currentTimeMillis();
    t1.start();
    // 线程执行结束会导致 join 结束
    t1.join(1500);
    long end = System.currentTimeMillis();
    log.debug("r1: {} r2: {} cost: {}", r1, r2, end - start);
}
```

输出

```plain
 20:52:15.623 [main] c.TestJoin - r1: 0 r2: 0 cost: 1502
```

## 9 interrupt 方法详解
### 9.1 打断正常运行的线程
**打断 sleep，wait，join的线程**：这几个方法都会让线程进入阻塞状态

第一种情况：打断 sleep 的线程, 会清空打断状态，以 sleep 为例

```java
private static void test1() throws InterruptedException { 
    Thread t1 = new Thread(()->{
        sleep(1); 
    }, "t1");
    t1.start();
    sleep(0.5);
    t1.interrupt();
    log.debug(" 打断状态: {}", t1.isInterrupted());
}
```

输出

```java
java.lang.InterruptedException: sleep interrupted
at java.lang.Thread.sleep(Native Method)
at java.lang.Thread.sleep(Thread.java:340)
at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386) at cn.itcast.n2.util.Sleeper.sleep(Sleeper.java:8)
at cn.itcast.n4.TestInterrupt.lambda$test1$3(TestInterrupt.java:59)
at java.lang.Thread.run(Thread.java:745) 21:18:10.374 [main] c.TestInterrupt - 打断状态: false
```



第二种情况：打断正常运行的线程, 不会清空打断状态

```java
private static void test2() throws InterruptedException {
    Thread t2 = new Thread(()->{
        while(true) {
            Thread current = Thread.currentThread();
            boolean interrupted = current.isInterrupted();
            if(interrupted) {
                log.debug(" 打断状态: {}", interrupted);
                break;
            }
        }
    }, "t2");
    t2.start();
    sleep(0.5);
    t2.interrupt();
}
```

输出

```java
20:57:37.964 [t2] c.TestInterrupt - 打断状态: true
```

### 9.2 模式之两阶段终止
Two Phase Termination：在一个线程 T1 中如何【**优雅** 】终止线程 T2？这里的【优雅】指的是给 T2 一个料理后事的机会。

错误思路

+ 使用线程对象的 stop() 方法停止线程 ：stop 方法会真正杀死线程，如果这时线程锁住了共享资源，那么当它被杀死后就再也没有机会释放锁， 其它线程将永远无法获取锁
+ 使用 System.exit(int) 方法停止线程：目的仅是停止一个线程，但这种做法会让整个程序都停止



两阶段终止模式

![](https://cdn.nlark.com/yuque/__mermaid_v3/eae51f2175ca81c8043ecd0aa80e7ac1.svg)

利用 isInterrupted：interrupt 可以打断正在执行的线程，无论这个线程是在 sleep，wait，还是正常运行

```java
package com.myspring;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class TPTInterrupt {
    private Thread thread;

    public void start() {
        thread = new Thread(() ->
        {
            while (true) {
                Thread current = Thread.currentThread();
                if (current.isInterrupted()) {
                    log.debug("料理后事");
                    break;
                }
                try {
                    Thread.sleep(1000);
                    log.debug("将结果保存");
                } catch (InterruptedException e) {
                    current.interrupt();
                }
                // 执行监控操作 }
            }
        }, "监控线程");
        thread.start();
    }

    public void stop() {
        thread.interrupt();
    }

    //调用
    public static void main(String[] args) throws InterruptedException {
        TPTInterrupt t = new TPTInterrupt();
        t.start();
        Thread.sleep(3500);
        log.debug("stop");
        t.stop();
    }
}

```

输出：

```java
17:36:03.722 [监控线程] DEBUG com.myspring.TPTInterrupt - 将结果保存
17:36:04.730 [监控线程] DEBUG com.myspring.TPTInterrupt - 将结果保存
17:36:05.734 [监控线程] DEBUG com.myspring.TPTInterrupt - 将结果保存
17:36:06.218 [main] DEBUG com.myspring.TPTInterrupt - stop
17:36:06.219 [监控线程] DEBUG com.myspring.TPTInterrupt - 料理后事
```

### 9.3 打断park线程
打断 park 线程, 不会清空打断状态

```java
private static void test3() throws InterruptedException { 
    Thread t1 = new Thread(() -> {
        log.debug("park...");
        LockSupport.park();
        log.debug("unpark...");
        log.debug("打断状态:{}", Thread.currentThread().isInterrupted());
    }, "t1"); t1.start();
    sleep(0.5);
    t1.interrupt(); 
}
```

输出

```java
21:11:52.795 [t1] c.TestInterrupt - park... 
21:11:53.295 [t1] c.TestInterrupt - unpark... 
21:11:53.295 [t1] c.TestInterrupt - 打断状态：true
```



如果打断标记已经是 true, 则 park 会失效

```java
private static void test4() { 
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 5; i++) {
            log.debug("park...");
            LockSupport.park();
            log.debug("打断状态:{}", Thread.currentThread().isInterrupted());
        } 
    });
    t1.start();
    sleep(1);
    t1.interrupt(); 
}
```

输出

```java
21:13:48.783 [Thread-0] c.TestInterrupt - park... 
21:13:49.809 [Thread-0] c.TestInterrupt - 打断状态：true 
21:13:49.812 [Thread-0] c.TestInterrupt - park... 
21:13:49.813 [Thread-0] c.TestInterrupt - 打断状态：true 
21:13:49.813 [Thread-0] c.TestInterrupt - park... 
21:13:49.813 [Thread-0] c.TestInterrupt - 打断状态：true 
21:13:49.813 [Thread-0] c.TestInterrupt - park... 
21:13:49.813 [Thread-0] c.TestInterrupt - 打断状态：true 
21:13:49.813 [Thread-0] c.TestInterrupt - park... 
21:13:49.813 [Thread-0] c.TestInterrupt - 打断状态：true
```

> 提示：可以使用 Thread.interrupted() 清除打断状态.
>

## 10 不推荐的方法
还有一些不推荐使用的方法，这些方法已过时，容易破坏同步代码块，造成线程死锁

| 方法名       | static | 功能说明       |
| --------- | ------ | ---------- |
| stop()    |        | 停止线程运行     |
| suspend() |        | 挂起(暂停)线程运行 |
| resume()  |        | 恢复线程运行     |


## 11 主线程与守护线程
默认情况下，Java 进程需要等待所有线程都运行结束，才会结束。有一种特殊的线程叫做守护线程，只要其它非守护线程运行结束了，即使守护线程的代码没有执行完，也会强制结束。

```java
log.debug("开始运行..."); 

Thread t1 = new Thread(() -> {
    log.debug("开始运行..."); 
    sleep(2); 
    log.debug("运行结束...");
}, "daemon");

// 设置该线程为守护线程 
t1.setDaemon(true); 
t1.start();

sleep(1); 
log.debug("运行结束...");
```

输出

```java
08:26:38.123 [main] c.TestDaemon - 开始运行... 
08:26:38.213 [daemon] c.TestDaemon - 开始运行... 
08:26:39.215 [main] c.TestDaemon - 运行结束...
```

注意：垃圾回收器线程就是一种守护线程。Tomcat 中的 Acceptor 和 Poller 线程都是守护线程，所以 Tomcat 接收到 shutdown 命令后，不会等待它们处理完当前请求

## 12 五种状态 vs 六种状态
[03-4 线程状态转换](https://www.yuque.com/chenguang201/java/nnzp2whdbla71gnp)

