---
title: FutureTask 源码学习
layout: post
categories: [Java, concurrent]
image: /assets/img/notes/FutureTask.jpg
description: "Welcome"
---

## 1. 状态机

```java
    /**
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
	// task状态
    private volatile int state;
	// 尚未执行
    private static final int NEW          = 0;
	// 正在结束，尚未完全结束，临界状态
    private static final int COMPLETING   = 1;
	// 任务正常结束
    private static final int NORMAL       = 2;
	// 任务在执行过程中发生了异常，内部封装的callable.run()向上跑出异常了
    private static final int EXCEPTIONAL  = 3;
	// 当前任务被取消
    private static final int CANCELLED    = 4;
	// 当前任务中断中
    private static final int INTERRUPTING = 5;
	// 当前任务已中断
    private static final int INTERRUPTED  = 6;
```

## 2. 成员变量

```java
    /** The underlying callable; nulled out after running runnable使用装饰着模式 伪装成Callable接口*/
	// submit(runnable/callable)都会复制到这里来
    private Callable<V> callable;
    /** The result to return or exception to throw from get() 正常情况下：任务正常执行结束，outcome保存执行结果。callable返回值
    * 非正常情况下：callable向上跑出异常，outcome保存异常*/
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
	// 当前任务被线程执行期间，保存当前执行任务的线程对象引用
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
	// 因为会有很多线程去get当前任务的结果，所以这里使用了一种数据结构 stack 头插 头取的队列
    private volatile java.util.concurrent.FutureTask.WaitNode waiters;
```

## 3. 构造方法

```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        // callable就是我们自己实现的业务类
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    public FutureTask(Runnable runnable, V result) {
        // 使用装饰者模式，将runnable转换成了callable接口，外部线程通过get获取当前任务执行结果时，结果可能为null，也可能为传进来的值
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }

    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }

    /**
     * A callable that runs given task and returns given result
     */
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            // run完之后返回之前给的result
            task.run();
            return result;
        }
    }
```

一般我们不会去new 一个FutureTask，一般是通过executor.submit一个任务，这个任务在真正exe之前，任务会被转变为task。在AbstractExecutorService里，是这样操作的：

```java
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 把任务真正地提交到线程池里面
        execute(ftask);
        return ftask;
    }
```

## 4. run方法

入口方法调用过程是：

submit(callable/runnable)  -->  newTaskFor(runnable)  -->  exeute(task) --> pool

```java
	// 任务执行入口
    public void run() {
        // 条件1： state != new, 条件成立，说明当前task已经被执行过了或者被cancelled了，线程就不处理了
        // 条件2：如果cas失败，当前任务被其他任务抢占了
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        // 执行到这里，当前task一定是NEW状态，而且当前线程也抢占task成功
        try {
            // 自己封装逻辑的callable或者装饰后的runnable
            Callable<V> c = callable;
            // 条件1：防止空指针
            // 条件2：防止外部线程cancel掉当前任务
            if (c != null && state == NEW) {
                // 保留一个结果的引用
                V result;
                // true表示callable.run 代码块执行成功
                // false表示callble.run 代码块跑出异常
                boolean ran;
                try {
                   	// 我们自己写的业务逻辑
                    result = c.call();
                    // c.call未抛出任何异常，ran会设置为true 代码块执行成功
                    ran = true;
                } catch (Throwable ex) {
                    // 有bug
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    // 说明当前c.call正常执行结束了
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


    protected void set(V v) {
        // 使用CAS方式设置当前任务为 完成中
        // 有可能失败吗？外部线程等不及了，在cas执行之前cancel了
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            // 等结果赋值给outcoome之后，马上会将当前任务状态修改为NORMAL 正常结束状态
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }

    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }


    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        // q指向waiters链表的头接地那
        for (WaitNode q; (q = waiters) != null;) {
            // 使用cas设置waiters为null 因为怕外部线程使用cancel取消当前任务 也会触发finishCompletion
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        // 唤醒线程
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```

## get方法

```java
    public V get() throws InterruptedException, ExecutionException {
        // 获取状态
        int s = state;
        // 条件：未执行、正在执行、正在完成，调用get的外部线程会被阻塞
        if (s <= COMPLETING)
            // 返回当前状态，可能当前线程已经在里面睡过一会了
            s = awaitDone(false, 0L);
        return report(s);
    }



    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        // 引用当前线程封装成WaitNode对象
        WaitNode q = null;
        // 表示当前线程waitNode对象 有没有入队、压栈
        boolean queued = false;
        for (;;) {
            // 条件成立说明当前线程唤醒，是被其他线程使用中断这种方式唤醒的，interrupted()返回true后，会将Thread的中断
            if (Thread.interrupted()) {
                // 当前线程出队
                // 方法里
                removeWaiter(q);
                // 抛出中断异常
                throw new InterruptedException();
            }
			// 如果是被unpark(thread)唤醒，会正常自旋，走下面逻辑
            // 获取当前任务最新状态
            int s = state;
            // 条件成立，说明当前任务已经有结果了
            if (s > COMPLETING) {
                // 条件成立说明已经为当前线程创建过node了，此时需要将node.thread = null，help GC，直接返回
                if (q != null)
                    q.thread = null;
                return s;
            }
            // 条件成立说明当前任务接近完成状态，这里让线程再释放cpu，进行下一次抢占cpu
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            // 条件成立：第一次自旋，当前线程还未创建创建waitnode对象
            else if (q == null)
                q = new WaitNode();
            // 条件成立：第二次自旋，waitNode已经对象，还未入队
            else if (!queued)
                // q.next = waiters把自己指向队列的头
                // cas方式设置当前waiters引用指向当前线程node，成功的话 queued == true 否则，可能其他线程先你一步入队了
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            // 正常情况下第三次自旋
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                // 当前get操作的线程就会被park掉了，线程状态会变成waiting状态
                // 除非有其他线程将你唤醒，或者将当前线程中断
                LockSupport.park(this);
        }
    }
```

## cancel方法

```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        // state == NEW 成立 表示当前任务处于运行中或者处于线程池 任务队列汇总
        // 第二条件通过cas的方法将状态改成INTERRUPTING
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;
        try {    // in case call to interrupt throws exception
            // 执行当前FutureTask的线程，有可能现在是null，是null的情况是：当前任务在队列中
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    // 条件成立说明当前线程runner正在执行task
                    if (t != null)
                        // 给runner一个中断信号，取决于你的程序是否响应中断，否则什么都不会发生
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            // 唤醒所有get()阻塞线程
            finishCompletion();
        }
        return true;
    }
```

