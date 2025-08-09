**笔记来源：**[**尚硅谷JUC并发编程（对标阿里P6-P7）**](https://www.bilibili.com/video/BV1ar4y1x727?p=1&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

# 1 Future接口介绍
Future接口（FutureTask实现类）定义了操作异步任务执行一些方法，如获取异步任务的执行结果、取消任务的执行、判断任务是否被取消、判断任务执行是否完毕等。（异步：可以被叫停，可以被取消）

![](images/1.png)

一句话：Future接口可以为主线程开一个分支任务，专门为主线程处理耗时和费力的复杂业务。

比如主线程让一个子线程去执行任务，子线程可能比较耗时，启动子线程开始执行任务后，主线程就去做其他事情了，过了一会才去获取子任务的执行结果。老师在上课，但是口渴，于是让班长这个线程去买水，自己可以继续上课，实现了异步任务。

Future是Java5新加的一个接口，他提供了一种异步并行计算的功能。

如果主线程需要执行一个很耗时的计算任务，我们就可以通过Future把这个任务放到异步线程中执行。主线程继续处理其他任务或者先行结束，再通过Future获取计算结果。

目的：异步多线程任务执行且有返回结果。

三个特点：多线程/有返回/异步任务（班长作为老师去买水作为新启动的异步多线程任务且买到水有结果返回）

# 2 FutureTask
## 2.1 FutureTask的架构
FutureTask的架构图如下，不仅是实现了Future接口，还实现了FutureTask的接口。

所以说上面提到的三个特点：多线程/有返回/异步任务，此时FutureTask已经满足了两个：多线程，异步任务。那么如何实现有返回呢？

![](images/2.png)

我们来看看FutureTask里面的方法吧

![](images/3.png)

这里面的两个构造方法，其中有一个是Callable接口，我们来看看Callable接口，它是满足我们有返回这个需求的。

![](images/4.png)

所以说，FutureTask这个实现类可以满足上面说的三个特点：多线程/有返回/异步任务

我们再来看看，既然FutureTask作为Runable接口的实现类，那么将来异步任务必然是在其run方法里面执行的，我们来看看其run方法

![](images/5.png)

我们发现，在其run方法里面，是调用Callable接口的call方法的，所以我们的异步任务其实是写在call方法里面的

FutureTask的初步使用

```java
package com.bilibili.juc.cf;

import java.util.concurrent.*;

public class CompletableFutureDemo
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        FutureTask<String> futureTask = new FutureTask<>(new MyThread());

        Thread t1 = new Thread(futureTask,"t1");
        t1.start();

        System.out.println(futureTask.get());
    }
}

class MyThread implements Callable<String>
{
    @Override
    public String call() throws Exception
    {
        System.out.println("-----come in call() " );
        return "hello Callable";
    }
}
```

打印出

```java
-----come in call() 
hello Callable
```

针对上面的代码，我们来分析FutureTask的优缺点

优点：Future结合线程池异步多线程任务配合，能够显著的提高程序的执行效率

```java
package com.bilibili.juc.cf;

import java.util.concurrent.*;

public class FutureThreadPoolDemo
{
    public static void main(String[] args) throws ExecutionException, InterruptedException
    {
        //3个任务，目前开启多个异步任务线程来处理，请问耗时多少？

        ExecutorService threadPool = Executors.newFixedThreadPool(3);

        long startTime = System.currentTimeMillis();

        FutureTask<String> futureTask1 = new FutureTask<String>(() -> {
            try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
            return "task1 over";
        });
        threadPool.submit(futureTask1);

        FutureTask<String> futureTask2 = new FutureTask<String>(() -> {
            try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }
            return "task2 over";
        });
        threadPool.submit(futureTask2);

        System.out.println(futureTask1.get());
        System.out.println(futureTask2.get());

        try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }

        long endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒");


        System.out.println(Thread.currentThread().getName()+"\t -----end");
        threadPool.shutdown();


    }

    private static void m1()
    {
        //3个任务，目前只有一个线程main来处理，请问耗时多少？

        long startTime = System.currentTimeMillis();
        //暂停毫秒
        try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
        try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }
        try { TimeUnit.MILLISECONDS.sleep(300); } catch (InterruptedException e) { e.printStackTrace(); }

        long endTime = System.currentTimeMillis();
        System.out.println("----costTime: "+(endTime - startTime) +" 毫秒");

        System.out.println(Thread.currentThread().getName()+"\t -----end");
    }
}
```

缺点

+ get()方法引起阻塞：一旦调用get()方法求结果，如果计算没有完成容易导致程序阻塞。
+ idDone()轮训：轮训的方式会耗费无谓的CPU资源，而且也不见得能及时的得到计算结果，如果想要异步获取结果，通常都会以轮训的方式去获取结果，尽量不要阻塞。

```java
package com.bilibili.juc.cf;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.TimeoutException;

public class FutureAPIDemo
{
    public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException
    {
        FutureTask<String> futureTask = new FutureTask<String>( () -> {
            System.out.println(Thread.currentThread().getName()+"\t -----come in");
            try { TimeUnit.SECONDS.sleep(5); } catch (InterruptedException e) { e.printStackTrace(); }
            return "task over";
        });
        Thread t1 = new Thread(futureTask, "t1");
        t1.start();

        System.out.println(Thread.currentThread().getName()+"\t ----忙其它任务了");

        //System.out.println(futureTask.get());
        //System.out.println(futureTask.get(3,TimeUnit.SECONDS));

        while(true)
        {
            if(futureTask.isDone())
            {
                System.out.println(futureTask.get());
                break;
            }else{
                //暂停毫秒
                try { TimeUnit.MILLISECONDS.sleep(500); } catch (InterruptedException e) { e.printStackTrace(); }
                System.out.println("正在处理中，不要再催了，越催越慢 ，再催熄火");
            }
        }
    }
}

/**
 *	1 get容易导致阻塞，一般建议放在程序后面，一旦调用不见不散，非要等到结果才会离开，不管你是否计算完成，容易程序堵塞。
 *	2 假如我不愿意等待很长时间，我希望过时不候，可以自动离开.
 */
```

结论：Future对于结果的获取不是很友好，只能通过阻塞或者轮训的方式得到任务的结果。

想完成一些复杂的任务，对于简单的业务场景使用Future完全可以，但是一些复杂的需求，比如：

+  回调通知：应对Future的完成时间，完成了可以告诉我，也就是我们的回调通知，通过轮训的方式去判断任务是否完成，这样非常占CPU，而且代码也不优雅。
+  创建异步任务：这个需求，Future配合线程池可以完成
+  多个任务前后依赖可以组合处理（水煮鱼）
    - 想将多个异步任务的计算结果组合起来，后一个异步任务的计算结果需要前一个异步任务的值，将两个或多个异步任务计算合成一个异步计算，这几个异步计算相互独立，同时后面这个又依赖前一个处理的结果
+  对计算速度选最快的：当Future集合中某一个任务最快结束时，返回结果，返回第一名处理结果。

针对于上述的特殊需求，使用Future之前提供的那点API就囊中羞涩，处理起来不够优雅，这时候还是让CompletableFuture以声明式的方式优雅的处理这些需求，Future能干的，CompletableFuture都能干。

# 3 CompletableFuture
## 3.1 CompletableFuture结构
CompletableFuture为什么出现呢？  

Future的get方法在计算完成之前会一直处在阻塞状态下，isDone方法容易耗费CPU资源，对于真正的异步处理我们是希望可以通过传入回调函数，在Future结束时候自动调用该回调函数，这样我们就不用等待结果。  

阻塞的方式和异步编程的设计理念想违背，而轮询的方式会消耗无谓的CPU资源，因此JDK8设计出CompletableFuture。

CompletableFuture提供了一种观察者模式类似的机制，可以让任务执行完成后通知监听的一方。

CompletableFuture类架构图    

![](images/6.png)

先来看看这个CompletionStage  

![](images/7.png)

有这么多的方法，要远远的比Future复杂的多。那么CompletionStage是什么呢？  ![](images/8.png)

代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另外一个阶段，有些类似Linux系统的管道分隔符传参数。  

CompletableFuture

![](images/9.png)

