---
title: Java Thread类里的方法——start()\interrupt()\join()等
layout: post
categories: [Java, concurrent]
image: /assets/img/notes/Multithread_ThreadRunnable.png
description: "Welcome"
---

## Java线程

### 启动线程

方法1， 直接使用Thread

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

方法2，使用Runnable配合Thread

把【线程】和【任务】（要执行的代码）分开 

- Thread 代表线程 
- Runnable 可运行的任务（线程要执行的代码）

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

- 方法1 是把线程和任务合并在了一起，方法2 是把线程和任务分开了 
- 用 Runnable 更容易与线程池等高级 API 配合 
- 用 Runnable 让任务类脱离了 Thread 继承体系，更灵活

方法3，FutureTask配合Thread

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

### 栈与栈帧 

Java Virtual Machine Stacks （Java 虚拟机栈） 我们都知道 JVM 中由堆、栈、方法区所组成，其中栈内存是给谁用的呢？其实就是线程，每个线程启动后，虚拟 机就会为其分配一块栈内存。 

- 每个栈由多个栈帧（Frame）组成，对应着每次方法调用时所占用的内存 
- 每个线程**只能有一个活动栈帧**，对应着当前正在执行的那个方法

### 线程上下文切换（Thread Context Switch）

因为以下一些原因导致 cpu 不再执行当前的线程，转而执行另一个线程的代码

- 线程的 cpu 时间片用完 
- 垃圾回收 
- 有更高优先级的线程需要运行
-  线程自己调用了 sleep、yield、wait、join、park、synchronized、lock 等方法

当 Context Switch 发生时，需要由操作系统保存当前线程的状态，并恢复另一个线程的状态，Java 中对应的概念 就是程序计数器（Program Counter Register），它的作用是记住下一条 jvm 指令的执行地址，是线程私有的

- 状态包括程序计数器、虚拟机栈中每个栈帧的信息，如局部变量、操作数栈、返回地址等 
- Context Switch 频繁发生会影响性能

### 常见方法

| 方法名              | 功能                                                         | 注意                                                         |
| ------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| start()             | **启动一个新线程**，在新的线程 运行 run 方法 中的代码        | start 方法只是让线程进入就绪，里面代码不一定立刻 运行（CPU 的时间片还没分给它）。每个线程对象的 start方法只能调用一次，如果调用了多次会出现 IllegalThreadStateException |
| run()               | 新线程启动后会 调用的方法                                    | 如果在构造 Thread 对象时传递了 Runnable 参数，则 线程启动后会调用 Runnable 中的 run 方法，否则默 认不执行任何操作。但可以创建 Thread 的子类对象， 来覆盖默认行为 |
| join()/join(long n) | 等待线程运行结 束                                            |                                                              |
| setPriority(int)    | 修改线程优先级                                               | java中规定线程优先级是1~10 的整数，较大的优先级 能提高该线程被 CPU 调度的机率 |
| getState()          | 获取线程状态                                                 | Java 中线程状态是用 6 个 enum 表示，分别为： NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED |
| isInterrupted()     | 判断是否被打 断                                              | 不会清除 打断标记                                            |
| interrupt()         | 打断线程                                                     | 如果被打断线程正在 sleep，wait，join 会导致被打断 的线程抛出 InterruptedException，并清除 打断标 记 ；如果打断的正在运行的线程，则会设置 打断标 记 ；park 的线程被打断，也会设置 打断标记 |
| interrupted()       | 判断当前线程是 否被打断                                      | 会清除 打断标记                                              |
| sleep(long n)       | 让当前执行的线 程休眠n毫秒， 休眠时让出 cpu 的时间片给其它 线程 |                                                              |
| yield()             | 提示线程调度器 让出当前线程对 CPU的使用                      |                                                              |

#### start和run

启动一个新的线程要使用start而不是run，做一个小实验，分别调用线程的run和start来观察输出结果。

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

输出：

```
19:39:14 [main] c.TestStart - main
19:39:14 [main] c.FileReader - read [1.mp4] start ...
19:39:18 [main] c.FileReader - read [1.mp4] end ... cost: 4227 ms
19:39:18 [main] c.TestStart - do other things ...
```

程序仍在main，FileReader.read()方法调用还是同步的，然后我们把t1.run()改成：

```java
t1.start();
```

输出结果：

```
19:41:30 [main] c.TestStart - do other things ...
19:41:30 [t1] c.TestStart - t1
19:41:30 [t1] c.FileReader - read [1.mp4] start ...
19:41:35 [t1] c.FileReader - read [1.mp4] end ... cost: 4542 ms
```

- 直接调用 run 是在主线程中执行了 run，没有启动新的线程 
- 使用 start 是启动新的线程，通过新的线程间接执行 run 中的代码

#### sleep与yield

**sleep**：

1. 调用 sleep 会让当前线程从 Running 进入 Timed Waiting 状态（阻塞） 
2. 其它线程可以使用 interrupt 方法打断正在睡眠的线程，这时 sleep 方法会抛出 InterruptedException 
3. 睡眠结束后的线程未必会立刻得到执行 
4. 建议用 TimeUnit 的 sleep 代替 Thread 的 sleep 来获得更好的可读性

**yield**

1. 调用 yield 会让当前线程从 Running 进入 Runnable 就绪状态，然后调度执行其它线程 
2. 具体的实现依赖于操作系统的任务调度器

#### 线程优先级

- 线程优先级会提示（hint）调度器优先调度该线程，但它仅仅是一个提示，调度器可以忽略它 
- 如果 cpu 比较忙，那么优先级高的线程会获得更多的时间片，但 cpu 闲时，优先级几乎没作用

#### join()

在下面这个程序中如果要结果打印出来会是多少，因为t1休眠了1秒，而主线程是直接运行的，所以在打印的时候r的值还不会被改成10.

```java
static int r = 0;
public static void main(String[] args) throws InterruptedException {
 test1();
}
private static void test1() throws InterruptedException {
 Thread t1 = new Thread(() -> {
 sleep(1);
 r = 10;
 });
 t1.start();
 log.debug("结果为:{}", r);
}

```

在t1.start()后面使用t1.join()，r的值就会输入10了。

#### interrupt()

**打断 sleep，wait，join 的线程**，这几个方法都会让线程进入阻塞状态。

##### 打断sleep的线程，会清空打断状态，

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

输出：

```
java.lang.InterruptedException: sleep interrupted
 ...
21:18:10.374 [main] c.TestInterrupt - 打断状态: false
```

##### 打断正常运行的线程，不会清空打断状态

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

输出:

```java
20:57:37.964 [t2] c.TestInterrupt - 打断状态: true 
```

##### 打断park线程，不会清空状态

```java
private static void test3() throws InterruptedException {
 	Thread t1 = new Thread(() -> {
 		log.debug("park...");
 		LockSupport.park();
 		log.debug("unpark...");
 		log.debug("打断状态：{}", Thread.currentThread().isInterrupted());
 	}, "t1");
 	t1.start();
 	sleep(0.5);
 	t1.interrupt();
}
```

输出：

```java
21:11:52.795 [t1] c.TestInterrupt - park...
21:11:53.295 [t1] c.TestInterrupt - unpark...
21:11:53.295 [t1] c.TestInterrupt - 打断状态：true 
```

如果打断标记已经是 true, 则 park 会失效.

```java
private static void test4() {
 	Thread t1 = new Thread(() -> {
 		for (int i = 0; i < 5; i++) {
 			log.debug("park...");
        	LockSupport.park();
 			log.debug("打断状态：{}", Thread.currentThread().isInterrupted());
 		}
 	});
 	t1.start();
	 sleep(1);
 	t1.interrupt();
}
```

输出：只有第一次进行了一秒钟

```
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

**可以使用Thread.interrupted()来清除打断状态**



### 两阶段终止模式Two Phase Termination

在一个线程 T1 中如何“优雅”终止线程 T2？这里的【优雅】指的是给 T2 一个料理后事的机会。

不优雅的方式：

- 使用线程对象的 stop() 方法停止线程
  - stop 方法会真正杀死线程，如果这时线程锁住了共享资源，那么当它被杀死后就再也没有机会释放锁， 其它线程将永远无法获取锁
- 使用System.exit(int) 方法停止线程
  - 目的仅是停止一个线程，但这种做法会让整个程序都停止

例子，业务需要一个监控线程来监控程序，如果被打断需要料理后事。

![image-20210121145326004](/assets/img/notes/image-20210121145326004.png)

#### 利用isInterrupted()打断sleep线程

interrupt 可以打断正在执行的线程，无论这个线程是在 sleep，wait，还是正常运行

```java
class TPTInterrupt {
 	private Thread thread;
	public void start(){
 	thread = new Thread(() -> {
		while(true) {
 			Thread current = Thread.currentThread();
 			if(current.isInterrupted()) {
 				log.debug("料理后事");
 				break;
 			}
 			try {
 				Thread.sleep(1000);
 				log.debug("将结果保存");
 			} catch (InterruptedException e) {
 				current.interrupt();
    			}
            // 执行监控操作
        	}
    	}, "监控线程");
        thread.start();
    }
    public void stop() {
        thread.interrupt();
    }
}
```

调用

```java
TPTInterrupt t = new TPTInterrupt();
t.start();
Thread.sleep(3500);
log.debug("stop");
t.stop();

```

输出

```
11:49:42.915 c.TwoPhaseTermination [监控线程] - 将结果保存
11:49:43.919 c.TwoPhaseTermination [监控线程] - 将结果保存
11:49:44.919 c.TwoPhaseTermination [监控线程] - 将结果保存
11:49:45.413 c.TestTwoPhaseTermination [main] - stop
11:49:45.413 c.TwoPhaseTermination [监控线程] - 料理后事
```

#### 利用停止标记

```java
class TPTInterrupt {
 	private Thread thread;
    private volatile boolean stop = false;
	public void start(){
 	thread = new Thread(() -> {
		while(true) {
 			Thread current = Thread.currentThread();
 			if(stop) {
 				log.debug("料理后事");
 				break;
 			}
 			try {
 				Thread.sleep(1000);
 				log.debug("将结果保存");
 			} catch (InterruptedException e) {
                stop = true;
    			}
            // 执行监控操作
        	}
    	}, "监控线程");
        thread.start();
    }
    public void stop() {
        stop = true;
        thread.interrupt();
    }
}
```

调用

```java
TPTVolatile t = new TPTVolatile();
t.start();
Thread.sleep(3500);
log.debug("stop");
t.stop();
```

输出

```java
11:54:52.003 c.TPTVolatile [监控线程] - 将结果保存
11:54:53.006 c.TPTVolatile [监控线程] - 将结果保存
11:54:54.007 c.TPTVolatile [监控线程] - 将结果保存
11:54:54.502 c.TestTwoPhaseTermination [main] - stop
11:54:54.502 c.TPTVolatile [监控线程] - 料理后事
```



