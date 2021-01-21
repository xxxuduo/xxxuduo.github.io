---
title: Monitor原理、对象头和Java中锁的优化
layout: post
categories: [Java, Concurrent]
image: /assets/img/notes/writebarrier.png
description: "Welcome"
---

## Monitor原理

### final变量的原理

```java
public class TestFinal {
 	final int a = 20;
}

```

字节码文件

```java
0: aload_0
1: invokespecial #1 // Method java/lang/Object."<init>":()V
4: aload_0
5: bipush 20
7: putfield #2 // Field a:I
 <-- 写屏障
10: return
```

发现 final 变量的赋值也会通过 putfield 指令来完成，同样在这条指令之后也会加入写屏障，保证在其它线程读到 它的值时不会出现为 0 的情况

### Monitor概念

#### Java对象头

Monitor 被翻译为监视器或管程 

以32位虚拟机位例：

普通对象：对象头由mark word和klass word（从klass word知道类型）

```
|---------------------------------------------------------|
|				 Object Header (64 bits) 		    	  |
|--------------------------------|------------------------|
|       Mark Word (32 bits)		 | Klass Word (32 bits)   |
|--------------------------------|------------------------|
```

数组对象

```
|---------------------------------------------------------------------------------|
| 							Object Header (96 bits)                               |
|--------------------------------|-----------------------|------------------------|
|    	Mark Word(32bits)    	 |   Klass Word(32bits)  |  array length(32bits)  |
|--------------------------------|-----------------------|------------------------|

```

Mark Word 结构是

```
|-------------------------------------------------------|--------------------|
| 					Mark Word (32 bits)     		    | 		State 		 |
|-------------------------------------------------------|--------------------|
| 	hashcode:25 	| 	age:4 	|	biased_lock:0  | 01 | 		Normal 		 |
|-------------------------------------------------------|--------------------|
| 	thread:23    | epoch:2 | age:4 | biased_lock:1 | 01 | 		Biased 		 |
|-------------------------------------------------------|--------------------|
|			 ptr_to_lock_record:30				   | 00 | Lightweight Locked |
|-------------------------------------------------------|--------------------|
|			 ptr_to_heavyweight_monitor:30		   | 10 | Heavyweight Locked |
|-------------------------------------------------------|--------------------|
| 												   | 11 |   Marked for GC    |
|-------------------------------------------------------|--------------------|

```

64 位虚拟机 Mark Word

```
|---------------------------------------------------------------|--------------------|
| 						Mark Word (64 bits)		 			    | 		State 		 |
|---------------------------------------------------------------|--------------------|
| unused:25 |hashcode:31| unused:1 | age:4 | biased_lock:0 | 01 | 		Normal	     |
|---------------------------------------------------------------|--------------------|
| 	thread:54 | epoch:2 | unused:1 | age:4 | biased_lock:1 | 01 | 		Biased 		 |
|---------------------------------------------------------------|--------------------|
| 				 	ptr_to_lock_record:62				   | 00 | Lightweight Locked |
|---------------------------------------------------------------|--------------------|
| 				ptr_to_heavyweight_monitor:62 	  		   | 10 | Heavyweight Locked |
|---------------------------------------------------------------|--------------------|
| 														   | 11 |    Marked for GC   |
|---------------------------------------------------------------|--------------------|
```



每个 Java 对象都可以关联一个 Monitor 对象(Monitor对象是由操作系统提供，在java中看不到他的表示)，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的 Mark Word 中就被设置指向 Monitor 对象的指针，即记住Monitor的地址（ptr_to_heavyweight_monitor）。

- 刚开始 Monitor 中 Owner 为 null 
- 当 Thread-2 执行 synchronized(obj) 就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一 个 Owner 
- 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行 synchronized(obj)，就会进入 EntryList，BLOCKED状态 
- Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的 
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲 wait-notify 时会分析

> - synchronized必须是进入同一个对象的monitor才有上述的效果
> - 不加synchronized的对象不会关联监视器，不遵从以上规则

#### 字节码角度

```java
static final Object lock = new Object();
static int counter = 0;
public static void main(String[] args) {
 	synchronized (lock) {
 	counter++;
 	}
}
```

对应的字节码文件为

```java
public static void main(java.lang.String[]);
 descriptor: ([Ljava/lang/String;)V
 flags: ACC_PUBLIC, ACC_STATIC
 Code:
 stack=2, locals=3, args_size=1
 0: getstatic #2 // <- 拿到lock引用 （synchronized开始）
 3: dup
 4: astore_1 // lock引用 -> slot 1
 5: monitorenter // 将 lock对象 MarkWord 置为 Monitor 指针
 6: getstatic #3 // <- i
 9: iconst_1 // 准备常数 1
 10: iadd // +1
 11: putstatic #3 // -> i
 14: aload_1 // <- lock引用
 15: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 16: goto 24
 // 如果发生异常的话 对应下面的Exception table
 19: astore_2 // e -> slot 2
 20: aload_1 // <- lock引用 之前store1了
 21: monitorexit // 将 lock对象 MarkWord 重置, 唤醒 EntryList
 22: aload_2 // <- slot 2 (e)
 23: athrow // throw e
 24: return
 Exception table:
 from to target type
 6 16 19 any
 19 22 19 any
 LineNumberTable:
 line 8: 0
 line 9: 6
 line 10: 14
 line 11: 24
 LocalVariableTable:
 Start Length Slot Name Signature
 0 25 0 args [Ljava/lang/String;
 StackMapTable: number_of_entries = 2
 frame_type = 255 /* full_frame */
 offset_delta = 19
 locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
 stack = [ class java/lang/Throwable ]
 frame_type = 250 /* chop */
 offset_delta = 4
```

> 这说明了加上synchronized之后，字节码文件都会保证锁正确释放

### 锁优化

#### 轻量级锁

轻量级锁的使用场景：如果一个对象虽然有多线程要加锁，但加锁的时间是错开的（也就是没有竞争），那么可以 使用轻量级锁来优化。 

轻量级锁对使用者是透明的，即语法仍然是synchronized。

假设有两个方法同步块，利用同一个对象加锁

```java
static final Object obj = new Object();
public static void method1() {
 	synchronized( obj ) {
 		// 同步块 A
 		method2();
 	}
}
public static void method2() {
 	synchronized( obj ) {
 		// 同步块 B
 	}
}
```

- 创建锁记录（Lock Record）对象，每个线程都的栈帧都会包含一个锁记录的结构，内部可以存储锁定对象的 Mark Word

![image-20210121232415618](/assets/img/notes/image-20210121232415618.png)

- 让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object 的 Mark Word，将 Mark Word 的值存 入锁记录

![image-20210121232854690](/assets/img/notes/image-20210121232854690.png)

- 如果 cas 替换成功，对象头中存储了 锁记录地址和状态 00 ，表示由该线程给对象加锁，这时图示如下

![image-20210121233254536](/assets/img/notes/image-20210121233254536.png)

- 如果 cas 失败，有两种情况 
  - 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程 
  - 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数

![image-20210121233630933](/assets/img/notes/image-20210121233630933.png)

- 当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重 入计数减一

![image-20210121233722978](/assets/img/notes/image-20210121233722978.png)

- 当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象 头 
  - 成功，则解锁成功 
  - 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程

#### 锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有 竞争），这时需要进行锁膨胀，将轻量级锁变为重量级锁。

```java
static Object obj = new Object();
public static void method1() {
 	synchronized( obj ) {
 		// 同步块
 	}
}
```

- 当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁

![image-20210121234508030](/assets/img/notes/image-20210121234508030.png)

- 这时 Thread-1 加轻量级锁失败，进入锁膨胀流程 
  - 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址 
  - 然后自己进入 Monitor 的 EntryList BLOCKED

![image-20210121235039647](/assets/img/notes/image-20210121235039647.png)

- 当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁 流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程

#### 自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步 块，释放了锁），这时当前线程就可以避免阻塞。

自旋成功的情况：

![image-20210121235335564](/assets/img/notes/image-20210121235335564.png)

自旋失败的情况：

![image-20210121235417609](/assets/img/notes/image-20210121235417609.png)

- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势。
- 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会 高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能。 
- Java 7 之后不能控制是否开启自旋功能

#### 偏向锁

轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。 

Java 6 中引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现 这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有，例如

```java
static final Object obj = new Object();
public static void m1() {
 	synchronized( obj ) {
 		// 同步块 A
 		m2();
 	}
}
public static void m2() {
 	synchronized( obj ) {
 		// 同步块 B
 		m3();
 	}
}
public static void m3() {
 	synchronized( obj ) {
      // 同步块 C
 	}
}
```

![image-20210122000132696](/assets/img/notes/image-20210122000132696.png)

![image-20210122000150220](/assets/img/notes/image-20210122000150220.png)

##### 偏向状态

回忆一下对象头格式

![image-20210122000304972](/assets/img/notes/image-20210122000304972.png)

biased_lock为1时表示偏向锁开启

一个对象创建时：

- 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的 thread、epoch、age 都为 0 
- 偏向锁是默认是**延迟**的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数 - XX:BiasedLockingStartupDelay=0 来禁用延迟 
- 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、 age 都为 0，**第一次用到 hashcode 时才会赋值**

偏向锁获取过程：

　　（1）访问Mark Word中偏向锁的标识是否设置成1，锁标志位是否为01——确认为可偏向状态。

　　（2）如果为可偏向状态，则测试线程ID是否指向当前线程，如果是，进入步骤（5），否则进入步骤（3）。

　　（3）如果线程ID并未指向当前线程，则通过CAS操作竞争锁。如果竞争成功，则将Mark Word中线程ID设置为当前线程ID，然后执行（5）；如果竞争失败，执行（4）。

　　（4）如果CAS获取偏向锁失败，则表示有竞争。当到达全局安全点（safepoint）时获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码。

　　（5）执行同步代码。

##### 测试偏向锁

```java
class Dog {}
```

利用 jol 第三方工具来查看对象头信息

```java
public static void main(String[] args) throws IOException {
 	Dog d = new Dog();
 	ClassLayout classLayout = ClassLayout.parseInstance(d);
 	new Thread(() -> {
 		log.debug("synchronized 前");
 		System.out.println(classLayout.toPrintableSimple(true));
 		synchronized (d) {
 			log.debug("synchronized 中");
 			System.out.println(classLayout.toPrintableSimple(true));
 		}
 	log.debug("synchronized 后");
 	System.out.println(classLayout.toPrintableSimple(true));
 	}, "t1").start();
}
```

输出

```
11:08:58.117 c.TestBiased [t1] - synchronized 前
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000101
11:08:58.121 c.TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101
11:08:58.121 c.TestBiased [t1] - synchronized 后
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101 
```

>添加虚拟机参数 -XX:BiasedLockingStartupDelay=0
>
>处于偏向锁的对象解锁后，线程 id 仍存储于对象头中

##### 测试禁用偏向锁

在上面测试代码运行时在添加 VM 参数 -XX:-UseBiasedLocking 禁用偏向锁

```
11:13:10.018 c.TestBiased [t1] - synchronized 前
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
11:13:10.021 c.TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00100000 00010100 11110011 10001000
11:13:10.021 c.TestBiased [t1] - synchronized 后
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 
```

测试 hashCode 正常状态对象一开始是没有 hashCode 的，第一次调用才生成

假如此时有另外一个线程线程 B 尝试获取该锁，线程 B - thread ID 为 101，同样的去检查锁标志位和是否可以偏向的状态发现可以后，然后 CAS 将 Mark Word 的 thread ID 指向自己，发现失败了，因为 thread ID 已经指向了线程 A ，那么此时就会去执行撤销偏向锁的操作了，会在一个全局安全点（没有字节码在执行）去暂停拥有偏向锁的线程（线程 A），然后检查线程 A 的状态，那么此时线程 A 就有 2 种情况了。

- 第一种情况，线程 A 已经终止状态，那么将 Mark Word 的线程 ID 置位空后，CAS 将线程 ID 偏向线程 B 然后就又回到上述又是偏向锁线程的运行状态了

- | thread ID | -     | 是否是偏向锁 | 锁标志位       |
  | --------- | ----- | ------------ | -------------- |
  | 101       | epoch | 1            | 01（未被锁定） |

- 第二种情况，线程 A 处于活动状态，那么就会将偏向锁升级为轻量级锁，然后唤醒线程 A 执行完后续操作，线程 B 自旋获取轻量级锁。

- | thread ID | 是否是偏向锁 | 锁标志位         |
  | --------- | ------------ | ---------------- |
  | 空        | 0            | 00（轻量级锁定） |

可以发现偏向锁适用于从始至终都只有一个线程在运行的情况，省略掉了自旋获取锁，以及重量级锁互斥的开销，这种锁的开销最低，性能最好接近于无锁状态，但是如果线程之间存在竞争的话，就需要频繁的去暂停拥有偏向锁的线程然后检查状态，决定是否重新偏向还是升级为轻量级别锁，性能就会大打折扣了，如果事先能够知道可能会存在竞争那么可以选择关闭掉偏向锁



##### 撤销 - 调用对象 hashCode

调用了对象的 hashCode，但偏向锁的对象 MarkWord 中存储的是线程 id，如果调用 hashCode 会导致偏向锁被 撤销 

- 轻量级锁会在锁记录中记录 hashCode 
- 重量级锁会在 Monitor 中记录 hashCode 在调用 hashCode 后使用偏向锁，记得去掉 -XX:-UseBiasedLocking

```java
11:22:10.386 c.TestBiased [main] - 调用 hashCode:1778535015
11:22:10.391 c.TestBiased [t1] - synchronized 前
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001
11:22:10.393 c.TestBiased [t1] - synchronized 中
00000000 00000000 00000000 00000000 00100000 11000011 11110011 01101000
11:22:10.393 c.TestBiased [t1] - synchronized 后
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001 
```

这个时候就已经没有偏向锁了

##### 撤销 - 其它线程使用对象

当有其它线程使用偏向锁对象时，会将偏向锁升级为轻量级锁

```java
private static void test2() throws InterruptedException {
 	Dog d = new Dog();
 	Thread t1 = new Thread(() -> {
 		synchronized (d) {
 		log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
 		}
 		synchronized (TestBiased.class) {
 		TestBiased.class.notify();
 	}
 	// 如果不用 wait/notify 使用 join 必须打开下面的注释
 	// 因为：t1 线程不能结束，否则底层线程可能被 jvm 重用作为 t2 线程，底层线程 id 是一样的
 	/*try {
		 System.in.read();
 	 } catch (IOException e) {
 		e.printStackTrace();
	 }*/
 	}, "t1");
 	t1.start();
 	Thread t2 = new Thread(() -> {
 		synchronized (TestBiased.class) {
			try {
 				TestBiased.class.wait();
 			} catch (InterruptedException e) {
	 			e.printStackTrace();
 			}
 		}
 	log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
 	synchronized (d) {
 		log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
 	}
	log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
 	}, "t2");
 	t2.start();
}

```

```
[t1] - 00000000 00000000 00000000 00000000 00011111 01000001 00010000 00000101
[t2] - 00000000 00000000 00000000 00000000 00011111 01000001 00010000 00000101
[t2] - 00000000 00000000 00000000 00000000 00011111 10110101 11110000 01000000
[t2] - 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 
```

这里模拟了两个线程竞争的状态，注意它有两个锁，wait notify是对于class的锁，是重型锁。

##### 批量重偏向

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象 的 Thread ID

当撤销偏向锁阈值超过 20 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给这些对象加锁时重新偏向至 加锁线程

##### 批量撤销 

当撤销偏向锁阈值超过 40 次后，jvm 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象 都会变为不可偏向的，新建的对象也是不可偏向的

#### 锁消除

锁消除主要是 JIT 编译器的优化操作，首先对于热点代码 JIT 编译器会将其编译为机器码，后续执行的时候就不需要在对每一条 class 字节码解释为机器码然后再执行了从而提升效率，它会根据逃逸分析来对代码做一定程度的优化比如锁消除，栈上分配等等

```java
public void f() {
    Object obj = new Object();
    synchronized(obj) {
         System.out.println(obj);
    }
}
```

JIT 编译器发现 f() 中的对象只会被一个线程访问，那么就会取消同步

```java
public void f() {
    Object obj = new Object();
    System.out.println(obj);
}
```

