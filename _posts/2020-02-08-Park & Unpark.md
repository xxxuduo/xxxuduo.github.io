---
title: Park/Unpark原理
layout: post
categories: [Java, Concurrent]
image: /assets/img/notes/image-20210124221357846.png
description: "Welcome"
---

## Park & Unpark

### 基本使用

```java
// 暂停当前线程
LockSupport.park();
// 恢复某个线程的运行
LockSupport.unpark(暂停线程对象)
```

先 park 再 unpark

```java
Thread t1 = new Thread(() -> {
 	log.debug("start...");
 	sleep(1);
 	log.debug("park...");
 	LockSupport.park();
 	log.debug("resume...");
},"t1");
t1.start();
sleep(2);
log.debug("unpark...");
LockSupport.unpark(t1);

// output
18:42:52.585 c.TestParkUnpark [t1] - start...
18:42:53.589 c.TestParkUnpark [t1] - park...
18:42:54.583 c.TestParkUnpark [main] - unpark...
18:42:54.583 c.TestParkUnpark [t1] - resume...
```

先unpark再park

```java
Thread t1 = new Thread(() -> {
 	log.debug("start...");
 	sleep(2);
 	log.debug("park...");
 	LockSupport.park();
 	log.debug("resume...");
}, "t1");
t1.start();
sleep(1);
log.debug("unpark...");
LockSupport.unpark(t1);

// output
18:43:50.765 c.TestParkUnpark [t1] - start...
18:43:51.764 c.TestParkUnpark [main] - unpark...
18:43:52.769 c.TestParkUnpark [t1] - park...
18:43:52.769 c.TestParkUnpark [t1] - resume...
```

与 Object 的 wait & notify 相比

- wait，notify 和 notifyAll 必须配合 Object Monitor 一起使用，而 park，unpark 不必
- park & unpark 是以线程为单位来【阻塞】和【唤醒】线程，而 notify 只能随机唤醒一个等待线程，notifyAll 是唤醒所有等待线程，就不那么【精确】
- park & unpark 可以先 unpark，而 wait & notify 不能先 notify

### 原理

Unsafe类的park/unpark是native级别的实现。使用native关键字说明这个方法是原生函数，也就是这个方法是用C/C++语言实现的，并且被编译成了DLL，由java去调用。

每个线程都有自己的一个 Parker 对象，由三部分组成 _counter ， _cond 和 _mutex 

打个比喻：

- 线程就像一个旅人，Parker 就像他随身携带的背包，条件变量就好比背包中的帐篷。_counter 就好比背包中 的备用干粮（0 为耗尽，1 为充足）
- 调用 park 就是要看需不需要停下来歇息 
  - 如果备用干粮耗尽，那么钻进帐篷歇息 
  - 如果备用干粮充足，那么不需停留，继续前进
- 调用 unpark，就好比令干粮充足 
  - 如果这时线程还在帐篷，就唤醒让他继续前进 
  - 如果这时线程还在运行，那么下次他调用 park 时，仅是消耗掉备用干粮，不需停留继续前进
    - 因为背包空间有限，多次调用 unpark 仅会补充一份备用干粮

```java
public static void park() {
    UNSAFE.park(false, 0L);
}
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
```

Unsafe类实现：

```java
//park
public native void park(boolean isAbsolute, long time);
 //unpack
public native void unpark(Object var1);
```

在Linux系统下，park和unpark是用的Posix线程库pthread中的mutex（互斥量），condition（条件变量）来实现的。简单来说，mutex和condition保护了一个叫_counter的信号量。当park时，这个变量被设置为0，当unpark时，这个变量被设置为1。当_counter=0 时线程阻塞，当_counter>0直接设为0并返回。

每个Java线程都有一个Parker实例，Parker类部分源码如下：

```java
class Parker : public os::PlatformParker {  
private:  
  volatile int _counter ;  
  ...  
public:  
  void park(bool isAbsolute, jlong time);  
  void unpark();  
  ...  
}  
class PlatformParker : public CHeapObj<mtInternal> {  
  protected:  
    pthread_mutex_t _mutex [1] ;  
    pthread_cond_t  _cond  [1] ;  
    ...  
}
```

 由源码可知Parker类继承于PlatformParker，实际上时用Posix的mutex，condition来实现的。Parker类里的_counter字段，就是用来记录park和unpark是否需要阻塞的标识。 

### 执行过程

具体的执行逻辑已经用注释标记在代码中，简要来说，就是检查_counter是不是大于0，如果是，则把_counter设置为0，返回。如果等于零，继续执行，阻塞等待。

```C
void Parker::park(bool isAbsolute, jlong time) {
  //判断信号量counter是否大于0，如果大于设为0返回
  if (Atomic::xchg(0, &_counter) > 0) return;
 
  //获取当前线程
  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");
  JavaThread *jt = (JavaThread *)thread;
 
  //如果中途已经是interrupt了，那么立刻返回，不阻塞
  // Check interrupt before trying to wait
  if (Thread::is_interrupted(thread, false)) {
    return;
  }
 
  //记录当前绝对时间戳
  // Next, demultiplex/decode time arguments
  timespec absTime;
  //如果park的超时时间已到，则返回
  if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
    return;
  }
  //更换时间戳
  if (time > 0) {
    unpackTime(&absTime, isAbsolute, time);
  }
 
  // Enter safepoint region
  // Beware of deadlocks such as 6317397.
  // The per-thread Parker:: mutex is a classic leaf-lock.
  // In particular a thread must never block on the Threads_lock while
  // holding the Parker:: mutex.  If safepoints are pending both the
  // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
  //进入安全点，利用该thread构造一个ThreadBlockInVM
  ThreadBlockInVM tbivm(jt);
 
  // Don't wait if cannot get lock since interference arises from
  // unblocking.  Also. check interrupt before trying wait
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }
 
  //记录等待状态
  int status ;
  //中途再次检查许可，有则直接返回不等带。
  if (_counter > 0)  { // no wait needed
    _counter = 0;
    status = pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
    // Paranoia to ensure our locked and lock-free paths interact
    // correctly with each other and Java-level accesses.
    OrderAccess::fence();
    return;
  }
 
  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()
  
  assert(_cur_index == -1, "invariant");
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    //线程条件等待 线程等待信号触发，如果没有信号触发，无限期等待下去。
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
    //线程等待一定的时间，如果超时或有信号触发，线程唤醒
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (&_cond[_cur_index]) ;
      pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
    }
  }
  _cur_index = -1;
  assert_status(status == 0 || status == EINTR ||
                status == ETIME || status == ETIMEDOUT,
                status, "cond_timedwait");
 
 
  _counter = 0 ;
  status = pthread_mutex_unlock(_mutex) ;
  assert_status(status == 0, status, "invariant") ;
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence();
 
 
  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}
```

unpark直接设置_counter为1，再unlock mutex返回。如果_counter之前的值是0，则还要调用pthread_cond_signal唤醒在park中等待的线程。

![image-20210124221357846](/assets/img/notes/image-20210124221357846.png)

1. 当前线程调用 Unsafe.park() 方法 
2. 检查 _counter ，本情况为 0，这时，获得 _mutex 互斥锁 
3. 线程进入 _cond 条件变量阻塞 
4. 设置 _counter = 0

![image-20210124222010208](/assets/img/notes/image-20210124222010208.png)

1. 调用 Unsafe.unpark(Thread_0) 方法，设置 _counter 为 1 
2. 唤醒 _cond 条件变量中的 Thread_0 
3. Thread_0 恢复运行 
4. 设置 _counter 为 0