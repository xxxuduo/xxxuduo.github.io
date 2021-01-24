---
title: ReentrantLock原理
layout: post
categories: [Java, Concurrent]
image: /assets/img/notes/reentrantLock.png
description: "Welcome"
---

## Reentrant Lock

### Synchronized和ReentrantLock：

#### 相同点

1. ReentrantLock和synchronized都是独占锁,只允许线程互斥的访问临界区。但是实现上两者不同:synchronized加锁解锁的过程是隐式的,用户不用手动操作,优点是操作简单，但显得不够灵活。一般并发场景使用synchronized的就够了；ReentrantLock需要手动加锁和解锁,且解锁的操作尽量要放在finally代码块中,保证线程正确释放锁。ReentrantLock操作较为复杂，但是因为可以手动控制加锁和解锁过程,在复杂的并发场景中能派上用场。
2. ReentrantLock和synchronized都是可重入的。synchronized因为可重入因此可以放在被递归执行的方法上,且不用担心线程最后能否正确释放锁；而ReentrantLock在重入时要却确保重复获取锁的次数必须和重复释放锁的次数一样，否则可能导致其他线程无法获得该锁。

#### 不同点

1. ReentrantLock是Java层面的实现，synchronized是JVM层面的实现。
2. ReentrantLock可以实现公平和非公平锁。
3. ReentantLock获取锁时，限时等待，配合重试机制更好的解决死锁
4. ReentrantLock可响应中断
5. 使用synchronized结合Object上的wait和notify方法可以实现线程间的等待通知机制，线程可以进入waitset里等待。ReentrantLock结合Condition接口同样可以实现这个功能。可以有**多个waitSet**来配合不同的条件进行等待（支持多个条件变量）

### 基本语法

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

Synchronized是在关键词级别去保护临界区，ReentrantLock是在对象的级别保护临界区。

### 可重入

可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁 如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住

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

输出：

```
17:59:11.862 [main] c.TestReentrant - execute method1
17:59:11.865 [main] c.TestReentrant - execute method2
17:59:11.865 [main] c.TestReentrant - execute method3 
```

### 可打断

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
 	try {
        // 注意如果是不可中断模式，那么即使使用了 interrupt 也不会让等待中断
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
 at
java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchr
onizer.java:898)
 at
java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchron
izer.java:1222)
 at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
 at cn.itcast.n4.reentrant.TestInterrupt.lambda$main$0(TestInterrupt.java:17)
 at java.lang.Thread.run(Thread.java:748)
18:02:41.532 [t1] c.TestInterrupt - 等锁的过程中被打断
```

### 锁超时

立刻失败

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

output

```
18:15:02.918 [main] c.TestTimeout - 获得了锁
18:15:02.921 [t1] c.TestTimeout - 启动...
18:15:02.921 [t1] c.TestTimeout - 获取立刻失败，返回
```

如果是定时等待就可以把上面代码里的：

>lock.tryLock()

改成

> lock.tryLock(1, TimeUnit.SECONDS)

#### 通过trylock解决哲学家就餐

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

### 公平锁

ReentrantLock 默认是不公平的

> ReentrantLock lock = new ReentrantLock(true);

### 条件变量

synchronized 中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待

ReentrantLock 的条件变量比 synchronized 强大之处在于，它是支持多个条件变量的，这就好比

- synchronized 是那些不满足条件的线程都在一间休息室等消息 
- 而 ReentrantLock 支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤 醒

使用要点：

- await 前需要获得锁 
- await 执行后，会释放锁，进入 conditionObject 
- 等待 await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁 
- 竞争 lock 锁成功后，从 await 后继续执行

```java
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

```
18:52:27.680 [main] c.TestCondition - 送早餐来了
18:52:27.682 [Thread-1] c.TestCondition - 等到了它的早餐
18:52:28.683 [main] c.TestCondition - 送烟来了
18:52:28.683 [Thread-0] c.TestCondition - 等到了它的烟
```



