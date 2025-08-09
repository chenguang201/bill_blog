

**笔记来源：**[**黑马程序员深入学习Java并发编程，JUC并发编程全套教程**](https://www.bilibili.com/video/BV16J411h7Rd/?spm_id_from=333.337.search-card.all.click&vd_source=e8046ccbdc793e09a75eb61fe8e84a30)

------

**前言：** 相对于 synchronized 它具备如下特点

+ 可中断
+ 可以设置超时时间
+ 可以设置为公平锁
+ 支持多个条件变量

与 synchronized 一样，都支持可重入


**基本语法**

```java
// 获取锁
reentrantLock.lock();
try {
    // 临界区
} finally {
    // 释放锁
    reentrantLock.unlock();
}
```

## 1 可重入
可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住

```java
static ReentrantLock lock = new ReentrantLock();

public static void main(String[] args) {
    method1();
}

public static void method1() {
    lock.lock();
    try {
        log.debug("execute method1");
        method2();
    } finally {
        lock.unlock();
    }
}

public static void method2() {
    lock.lock();
    try {
        log.debug("execute method2");
        method3();
    } finally {
        lock.unlock();
    }
}

public static void method3() {
    lock.lock();
    try {
        log.debug("execute method3");
    } finally {
        lock.unlock();
    }
}
```

输出

```java
17:59:11.862 [main] c.TestReentrant - execute method1 
17:59:11.865 [main] c.TestReentrant - execute method2 
17:59:11.865 [main] c.TestReentrant - execute method3
```



## 2 可打断
示例

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    try {
        lock.lockInterruptibly();
    } catch (InterruptedException e) {
        e.printStackTrace();
        log.debug("等锁的过程中被打断");
        return;
    }

    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();

try {
    sleep(1);
    t1.interrupt();
    log.debug("执行打断");
} finally {
    lock.unlock();
}
```

输出

```java
18:02:40.520 [main] c.TestInterrupt - 获得了锁
18:02:40.524 [t1] c.TestInterrupt - 启动... 
18:02:41.530 [main] c.TestInterrupt - 执行打断
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.ali.bingfa.TestLockInterrupt.lambda$main$0(TestLockInterrupt.java:12)
	at com.ali.bingfa.TestLockInterrupt$$Lambda$1/1534030866.run(Unknown Source)
	at java.lang.Thread.run(Thread.java:745)
18:02:41.532 [t1] c.TestInterrupt - 等锁的过程中被打断
```



**注意：** 如果是不可中断模式，那么即使使用了 interrupt 也不会让等待中断

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    lock.lock();
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();

try {
    sleep(1);
    t1.interrupt();
    log.debug("执行打断");
    sleep(1);
} finally {
    log.debug("释放了锁");
    lock.unlock();
}
```

输出

```java
18:06:56.261 [main] c.TestInterrupt - 获得了锁
18:06:56.265 [t1] c.TestInterrupt - 启动... 
18:06:57.266 [main] c.TestInterrupt - 执行打断 // 这时 t1 并没有被真正打断, 而是仍继续等待锁
18:06:58.267 [main] c.TestInterrupt - 释放了锁
18:06:58.267 [t1] c.TestInterrupt - 获得了锁
```



## 3 锁超时
**立刻失败**

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    if (!lock.tryLock()) {
        log.debug("获取立刻失败，返回");
        return;
    }

    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```java
18:15:02.918 [main] c.TestTimeout - 获得了锁
18:15:02.921 [t1] c.TestTimeout - 启动... 
18:15:02.921 [t1] c.TestTimeout - 获取立刻失败，返回
```



**超时失败**

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    try {
        if (!lock.tryLock(1, TimeUnit.SECONDS)) {
            log.debug("获取等待 1s 后失败，返回");
            return;
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```java
18:19:40.537 [main] c.TestTimeout - 获得了锁
18:19:40.544 [t1] c.TestTimeout - 启动... 
18:19:41.547 [t1] c.TestTimeout - 获取等待 1s 后失败，返回
```



使用 tryLock 解决哲学家就餐问题

```java
class Chopstick extends ReentrantLock {
    String name;

    public Chopstick(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```



```java
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;
    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }

    @Override
    public void run() {
        while (true) {
            // 尝试获得左手筷子
            if (left.tryLock()) {
                try {
                    // 尝试获得右手筷子
                    if (right.tryLock()) {
                        try {
                            eat();
                        } finally {
                            right.unlock();
                        }
                    }
                } finally {
                    left.unlock();
                }
            }
        }
    }

    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }
}
```

## 4 公平锁
ReentrantLock 默认是不公平的

```java
ReentrantLock lock = new ReentrantLock(false);

lock.lock();
for (int i = 0; i < 500; i++) {
    new Thread(() -> {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " running...");
        } finally {
            lock.unlock();
        }
    }, "t" + i).start();
}

// 1s 之后去争抢锁
Thread.sleep(1000);
new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " start...");
    lock.lock();
    try {
        System.out.println(Thread.currentThread().getName() + " running...");
    } finally {
        lock.unlock();
    }
}, "强行插入").start();
lock.unlock();
```

强行插入，有机会在中间输出  
注意：该实验不一定总能复现

```bash
t39 running... 
t40 running... 
t41 running... 
t42 running... 
t43 running... 
强行插入 start... 
强行插入 running... 
t44 running... 
t45 running... 
t46 running... 
t47 running... 
t49 running...
```



改为公平锁后

```java
ReentrantLock lock = new ReentrantLock(true);
```

强行插入，总是在最后输出

```bash
t465 running... 
t464 running... 
t477 running... 
t442 running... 
t468 running... 
t493 running... 
t482 running... 
t485 running... 
t481 running... 
强行插入 running...
```

公平锁一般没有必要，会降低并发度，后面分析原理时会讲解

## 5 条件变量
synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待。  
ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比

+ synchronized 是那些不满足条件的线程都在一间休息室等消息
+ 而ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤醒



使用要点：

+ await 前需要获得锁
+ await 执行后，会释放锁，进入 conditionObject 等待
+ await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁
+ 竞争 lock 锁成功后，从 await 后继续执行



例子：

```java
ReentrantLock lock = new ReentrantLock(true);
static ReentrantLock lock = new ReentrantLock();
static Condition waitCigaretteQueue = lock.newCondition();
static Condition waitbreakfastQueue = lock.newCondition();
static volatile boolean hasCigrette = false;
static volatile boolean hasBreakfast = false;

public static void main(String[] args) {
    new Thread(() -> {
        try {
            lock.lock();
            while (!hasCigrette) {
                try {
                    waitCigaretteQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的烟");
        } finally {
            lock.unlock();
        }
    }).start();

    new Thread(() -> {
        try {
            lock.lock();
            while (!hasBreakfast) {
                try {
                    waitbreakfastQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的早餐");
        } finally {
            lock.unlock();
        }
    }).start();

    sleep(1);
    sendBreakfast();
    sleep(1);
    sendCigarette();
}

private static void sendCigarette() {
    lock.lock();
    try {
        log.debug("送烟来了");
        hasCigrette = true;
        waitCigaretteQueue.signal();
    } finally {
        lock.unlock();
    }
}

private static void sendBreakfast() {
    lock.lock();
    try {
        log.debug("送早餐来了");
        hasBreakfast = true;
        waitbreakfastQueue.signal();
    } finally {
        lock.unlock();
    }
}
```

输出

```java
18:52:27.680 [main] c.TestCondition - 送早餐来了
18:52:27.682 [Thread-1] c.TestCondition - 等到了它的早餐
18:52:28.683 [main] c.TestCondition - 送烟来了
18:52:28.683 [Thread-0] c.TestCondition - 等到了它的烟
```

## 6 同步模式之顺序控制
### 6.1 固定顺序运行
目标：实现必须先 2 后 1 打印

**方式一：wait notify版**   

```java
// 用来同步的对象 
static Object obj = new Object(); 
// t2 运行标记， 代表 t2 是否执行过 
static boolean t2runed = false; 

public static void main(String[] args) { 
	Thread t1 = new Thread(() -> { 
		synchronized (obj) { 
			// 如果 t2 没有执行过 
			while (!t2runed) { 
				try { 
					// t1 先等一会 
					obj.wait(); 
 				} catch (InterruptedException e) { 
					e.printStackTrace(); 
            	} 
 			} 
 		} 
		System.out.println(1); 
 	}); 
    
	Thread t2 = new Thread(() -> { 
		System.out.println(2); 
		synchronized (obj) { 
			// 修改运行标记 
			t2runed = true; 
			// 通知 obj 上等待的线程（可能有多个，因此需要用 notifyAll） 
			obj.notifyAll(); 
 		} 
 	}); 
	t1.start(); 
	t2.start(); 
} 
```

**方式二： Park Unpark版**    

可以看到，实现上很麻烦： 

1. 首先，需要保证先 wait 再 notify，否则 wait 线程永远得不到唤醒。因此使用了『运行标记』来判断该不该 wait
2. 第二，如果有些干扰线程错误地 notify 了 wait 线程，条件不满足时还要重新等待，使用了 while 循环来解决 此问题
3. 最后，唤醒对象上的 wait 线程需要使用 notifyAll，因为『同步对象』上的等待线程可能不止一个 

可以使用LockSupport 类的park 和 unpark来简化上面的题目：

```java
Thread t1 = new Thread(() -> { 
	try { 
        Thread.sleep(1000); 
    } catch (InterruptedException e) {
        
    } 
	// 当没有『许可』时，当前线程暂停运行；有『许可』时，用掉这个『许可』，当前线程恢复运行 
    LockSupport.park(); 
	System.out.println("1"); 
}); 

Thread t2 = new Thread(() -> { 
	System.out.println("2"); 
	// 给线程 t1 发放『许可』（多次连续调用 unpark 只会发放一个『许可』） 
	LockSupport.unpark(t1); 
}); 

t1.start(); 
t2.start(); 
```

park 和 unpark 方法比较灵活，他俩谁先调用，谁后调用无所谓。并且是以线程为单位进行『暂停』和『恢复』，不需要『同步对象』和『运行标记』 

### 6.2 交替输出 
线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。现在要求输出 abcabcabcabcabc 怎么实现

**方式一：wait notify 版**   

```java
class SyncWaitNotify { 
	private int flag; 
	private int loopNumber; 
	public SyncWaitNotify(int flag, int loopNumber) { 
		this.flag = flag; 
		this.loopNumber = loopNumber; 
 	} 

    public void print(int waitFlag, int nextFlag, String str) { 
		for (int i = 0; i < loopNumber; i++) { 
			synchronized (this) { 
				while (this.flag != waitFlag) { 
					try { 
						this.wait(); 
 					} catch (InterruptedException e) { 
						e.printStackTrace(); 
 					} 
 				} 
				System.out.print(str); 
				flag = nextFlag; 
				this.notifyAll(); 
 			} 
 		} 
 	} 
} 
```

```java
SyncWaitNotify syncWaitNotify = new SyncWaitNotify(1, 5); 
new Thread(() -> { 
	syncWaitNotify.print(1, 2, "a"); 
}).start(); 
new Thread(() -> { 
	syncWaitNotify.print(2, 3, "b"); 
}).start(); 
new Thread(() -> { 
	syncWaitNotify.print(3, 1, "c"); 
}).start(); 
```

**方式二：Lock 条件变量版**    

```java
class AwaitSignal extends ReentrantLock { 
	public void start(Condition first) { 
		this.lock(); 
		try { 
			log.debug("start"); 
			first.signal(); 
 		} finally { 
			this.unlock(); 
 		} 
 	} 

    public void print(String str, Condition current, Condition next) { 
		for (int i = 0; i < loopNumber; i++) { 
			this.lock(); 
			try { 
				current.await(); 
				log.debug(str); 
				next.signal(); 
 			} catch (InterruptedException e) { 
				e.printStackTrace(); 
 			} finally { 
				this.unlock(); 
 			} 
 		} 
 	} 

    // 循环次数 
	private int loopNumber; 
	public AwaitSignal(int loopNumber) { 
		this.loopNumber = loopNumber; 
 	} 
} 
```

```java
AwaitSignal as = new AwaitSignal(5); 
Condition aWaitSet = as.newCondition(); 
Condition bWaitSet = as.newCondition(); 
Condition cWaitSet = as.newCondition(); 

new Thread(() -> { 
	as.print("a", aWaitSet, bWaitSet); 
}).start(); 

new Thread(() -> { 
	as.print("b", bWaitSet, cWaitSet); 
}).start(); 

new Thread(() -> { 
	as.print("c", cWaitSet, aWaitSet); 
}).start(); 

as.start(aWaitSet); 
```

>  **注意** ：该实现没有考虑 a，b，c 线程都就绪再开始

**方式三：Park Unpark 版**   

```java
class SyncPark { 
	private int loopNumber; 
	private Thread[] threads; 
	
    public SyncPark(int loopNumber) { 
		this.loopNumber = loopNumber; 
 	} 

    public void setThreads(Thread... threads) { 
		this.threads = threads; 
 	} 

    public void print(String str) { 
		for (int i = 0; i < loopNumber; i++) { 
			LockSupport.park(); 
			System.out.print(str); 
			LockSupport.unpark(nextThread()); 
 		} 
 	} 

    private Thread nextThread() { 
		Thread current = Thread.currentThread(); 
		int index = 0; 
		for (int i = 0; i < threads.length; i++) { 
			if(threads[i] == current) { 
				index = i; 
				break; 
 			} 
 		} 
		if(index < threads.length - 1) { 
			return threads[index+1]; 
 		} else { 
			return threads[0]; 
 		} 
 	} 
    
	public void start() { 
		for (Thread thread : threads) { 
			thread.start(); 
 		} 
		LockSupport.unpark(threads[0]); 
 	} 
} 
```

```java
SyncPark syncPark = new SyncPark(5);
Thread t1 = new Thread(() -> {
 	syncPark.print("a");
});
Thread t2 = new Thread(() -> {
 	syncPark.print("b");
});
Thread t3 = new Thread(() -> {
 	syncPark.print("c
");
});
syncPark.setThreads(t1, t2, t3);
syncPark.start();
```

