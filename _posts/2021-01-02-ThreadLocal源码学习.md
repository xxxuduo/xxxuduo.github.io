---
title: ThreadLocal 源码学习
layout: post
categories: [Java]
image: /assets/img/notes/image-20210118205659318.png
description: "Welcome"
---



ThreadLocal的作用是提供线程内的局部变量，不同的线程之间不会互相干扰，这种变量在线程的生命周期内起作用，减少通一个线程内多个函数或组件之间一些公共变量传递的复杂度。

总结下来就是：

1. 线程并发：在多线程并发的场景下
2. 传递数据：我们可以通过ThreadLocal在同一线程，不同组件中传递公共变量
3. 线程隔离：每个线程的变量都是独立的，不会互相影响

## 基本使用

在可能有并发访问的对象中设置ThreadLocal变量，然后通过ThreadLocal的set和get方法进行针对某一个线程的变量进行修改和访问。

并发问题出现：

```java
package threadlocaltest;

public class Content {

    String content;
    public void setContent(String s) {
        this.content = s;
    }
    public String getContent() {
        return content;
    }

    public static void main(String[] args) {
        Content c1 = new Content();
        for(int i = 0; i < 6; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    c1.setContent(Thread.currentThread().getName() + "的内容");
                    System.out.println("------------------------------");
                    System.out.println(Thread.currentThread().getName() + " ----> " + c1.getContent());
                }
            });
            t.setName("Thread " + i);
            t.start();
        }
    }
}
```

output

> Thread 0 ----> Thread 3的内容
>
> Thread 4 ----> Thread 4的内容
>
> Thread 3 ----> Thread 5的内容
>
> Thread 2 ----> Thread 5的内容
>
> Thread 1 ----> Thread 5的内容
>
> Thread 5 ----> Thread 5的内容

通过加入ThreadLocal变量，代码可以改写为

```java
package threadlocaltest;

public class Content {
    ThreadLocal<String> localVal = new ThreadLocal<>();
    String content;
    public void setContent(String s) {
        localVal.set(s);
    }
    public String getContent() {
        return localVal.get();
    }

    public static void main(String[] args) {
        Content c1 = new Content();
        for(int i = 0; i < 6; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    c1.setContent(Thread.currentThread().getName() + "的内容");
//                    System.out.println("------------------------------");
                    System.out.println(Thread.currentThread().getName() + " ----> " + c1.getContent());
                }
            });
            t.setName("Thread " + i);
            t.start();
        }
    }
}
```

output

>Thread 0 ----> Thread 0的内容
>Thread 1 ----> Thread 1的内容
>Thread 2 ----> Thread 2的内容
>Thread 5 ----> Thread 5的内容
>Thread 3 ----> Thread 3的内容
>Thread 4 ----> Thread 4的内容

## ThreadLocal vs Synchronized

|        | synchronized                                                 | ThreadLocal                                                  |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理   | 同步机制采用“以时间换空间“的方式，只提供了一份变量，让不同的线程排队访问 | ThreadLocal采用”以空间换时间“的方式，为每一个线程都提供了一份变量的副本，从而实现同时访问而相不干扰 |
| 侧重点 | 多个线程之间访问资源的同步                                   | 多线程中让每个线程之间的数据互相隔离                         |

ThreadLocal有两个突出的有事：

1. 传递数据：保存每个线程绑定的数据，在需要的地方可以直接获取，避免参数直接传递带来的代码耦合问题
2. 线程隔离：各线程之间的数据相互隔离却又具备并发性，避免同步方法带来的性能损失。

## ThreadLcoal内部结构

在JDK8中ThreadLocal的设计是：每个Thread都维护一个ThreadLocalMap，这个map的key是ThreadLocal实例本身，value才是真正要存储的值Object。

![image-20210118205659318](/assets/img/notes/image-20210118205659318.png)

![image-20210118205723760](/assets/img/notes/image-20210118205723760.png)

以上是JDK8和之前版本设计的区别，JDK8的具体过程如下：

1. 每个thread内部都有一个map
2. map里面存储threadlocal对象（key）和线程变量副本（value）
3. Thread内部的map是由ThreadLocal维护的，由ThreadLocal负责向map获取和设置线程的变量值
4. 对于不同的线程，每次获取副本值时，别的线程并不能获取当前线程的副本值，形成了副本的隔离，互不干扰。

这样设计的好处是：

1. 每个Map存储的Entry数量变少，因为ThreadLocal的数量是远远小于线程数量的
2. 当Thread销毁的时候，ThreadLocalMap也会随之销毁，减少内存使用。

### ThreadLocal源码分析

#### set方法

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
	public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }


    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }

```

流程如下：

1. 首先获取当前线程，并故居当前线程获取一个Map
2. 如果获取的map不为空，则将参数设置到Map中（当前ThreadLocal的引用作为key）
3. 如果Map为空，则给该线程创建map，并设置初始值

Thread类里维护了一个ThreadLocal

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```

#### get方法

```java
    /**
     * Returns the value in the current thread's copy of this
     * thread-local variable.  If the variable has no value for the
     * current thread, it is first initialized to the value returned
     * by an invocation of the {@link #initialValue} method.
     *
     * @return the current thread's value of this thread-local
     */    
	public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }

    /**
     * Variant of set() to establish initialValue. Used instead
     * of set() in case user has overridden the set() method.
     *
     * @return the initial value
     */
    private T setInitialValue() {
        // 调用initialValue获取初始化的值
        // 此方法可以被子类重写，如果不重写默认返回null
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            // 1.当前线程Thread不存在ThreadLocalMap对象
            // 2.调用createMap进行ThreadLocalMap对象的初始化
            // 3.并将t(当前线程)和value(t对应的值)作为第一个entry存放至ThreadLocalMap中
            createMap(t, value);
        return value;
    }
```

流程如下：

1. 首先获取当前线程，根据当前线程获取一个Map
2. 如果获取的map不为空，则在map中以threadLocal的引用作为Key在map中获取对应的entry e，否则转到4
3. 如果e不为null，则返回e.value,否则转到4
4. map为空或者e为空，则通过initialVlaue函数获取初始值value然后用ThreadLocal的引用和value作为firstKey和firstValue创建一个新的Map

先获取当前线程的ThreadLocalMap变量，如果存在则返回值，不存在则创建并返回初始值。

#### remove方法

```java
    /**
     * Removes the current thread's value for this thread-local
     * variable.  If this thread-local variable is subsequently
     * {@linkplain #get read} by the current thread, its value will be
     * reinitialized by invoking its {@link #initialValue} method,
     * unless its value is {@linkplain #set set} by the current thread
     * in the interim.  This may result in multiple invocations of the
     * {@code initialValue} method in the current thread.
     *
     * @since 1.5
     */
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

1. 获取当前线程，并根据当前线程获取一个map
2. 如果获取的map不为空，则移除当前ThreadLocal对象对应的entry

#### InitialValue

```java
    /**
     * Returns the current thread's "initial value" for this
     * thread-local variable.  This method will be invoked the first
     * time a thread accesses the variable with the {@link #get}
     * method, unless the thread previously invoked the {@link #set}
     * method, in which case the {@code initialValue} method will not
     * be invoked for the thread.  Normally, this method is invoked at
     * most once per thread, but it may be invoked again in case of
     * subsequent invocations of {@link #remove} followed by {@link #get}.
     *
     * <p>This implementation simply returns {@code null}; if the
     * programmer desires thread-local variables to have an initial
     * value other than {@code null}, {@code ThreadLocal} must be
     * subclassed, and this method overridden.  Typically, an
     * anonymous inner class will be used.
     *
     * @return the initial value for this thread-local
     */
    protected T initialValue() {
        return null;
    }
```

1. 这个方法是一个延迟调用方法，从上面的代码我们得知，在set方法还未调用而先调用了get方法时才只能，并且仅执行一次。
2. 这个方法缺省实现直接返回一个null
3. 如果想要以个除null之外的初始值，可以重写此方法。（此方法是一个protected的方法，显然是为了让子类覆盖而设计的）

### ThreadLocalMap源码分析

ThreadLocalMap是ThreadLocal的内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也是独立实现的。

![image-20210118214117759](/assets/img/notes/image-20210118214117759.png)

#### 成员变量

```java
        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0

        /**
         * Set the resize threshold to maintain at worst a 2/3 load factor.
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }

        /**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         */
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }

```

#### 存储结构-Entry

```java
        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

Entry中的eky只能是ThreadLocal对象，已经在构造方法中写死了。另外，Entry继承了**WeakReferrence**，也就是key（ThreadLocal）是弱引用，其目的是将ThreadLocal对象的声明周期和线程声明周期解绑。

#### 构造方法

```java
        /**
         * Construct a new map initially containing (firstKey, firstValue).
         * ThreadLocalMaps are constructed lazily, so we only create
         * one when we have at least one entry to put in it.
         */
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

```

> 重点 int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

```java
    /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;
```

##### 关于firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

这里定义了一个AtomicInteger类型，每次获取当前值并加上HASH_INCREMENT，这个值跟斐波那契数列有关，其主要目的就是为了让哈希码能均匀的分布在2的n此房的数组里，也就是Entry[] table中，这样做可以尽量避免hash冲突

##### 关于& (INITIAL_CAPACITY - 1)

计算hash的时候里面采用了hashCode&(size - 1)的算法，这相当于取模运算hashCode % size的一个更搞笑的实现。正式因为这种算法，我们要求size必须是2的整次幂，这也能保证在索引不越界的前提下，使得hash冲突的次数减小。

#### set方法

```java
        /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                // 如果存在，覆盖
                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                                replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }

        /**
         * Increment i modulo len.
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

```

1. 根据key计算出索引i，然后查找位置上的Entry
2. 如果entry已经存在并且key等于传入key，直接覆盖
3. 如果entry存在，但是key为null，调用replaceStaleEntry来更换这个key为空的entry
4. 不断玄幻检测，知道遇到为null的地方，这时候要是还没在循环过程中return，那么久在这个null的位置新建一个Entry并且插入，size+1
5. 最后调用cleanSomeSlots，清理key为null的entry，最后返回是否清理了entry，接下来再来判断sz是否>=thredhold达到rehash的条件，达到的话就调用rehash函数执行一次权标的扫描清理

```java
        /**
         * Replace a stale entry encountered during a set operation
         * with an entry for the specified key.  The value passed in
         * the value parameter is stored in the entry, whether or not
         * an entry already exists for the specified key.
         *
         * As a side effect, this method expunges all stale entries in the
         * "run" containing the stale entry.  (A run is a sequence of entries
         * between two null slots.)
         *
         * @param  key the key
         * @param  value the value to be associated with key
         * @param  staleSlot index of the first stale entry encountered while
         *         searching for key.
         */
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```



ThreadLocalMap使用线性探测法来解决哈希冲突。

该方法一次探测下一个地址，直到有空的地址后插入，弱整个空间都找不到空余的地址，则产生一处。

### 弱引用和内存泄漏

在使用ThreadLocal的过程中发现有内存泄漏的情况发生，猜测这个内存泄漏跟Entry中使用了弱引用的key有关系。其实这个理解不对。

#### 内存泄漏的相关概念

- Memory overflow，没有足够的内存提供申请者使用
- Memory leak，指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的额浪费，导致运行速度减慢甚至系统崩溃等严重后果。内存泄漏的堆积终将导致内存溢出。

#### 弱引用的相关概念

强引用（Strong Reference），就是我常见的普通对象引用，只要还有强引用指向一个对象，就证明对象还活着，垃圾回收旧不会回收这种对象。

弱引用（Weak Reference），垃圾回收器一旦发现了值具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。

#### 如果key使用强引用

![image-20210118215755270](/assets/img/notes/image-20210118215755270.png)

1. 假设在业务代码中使用完ThreadLocal，threadLocal Ref被回收了。
2. 但是因为threadLocalMap的Entry强引用了threadLocal，造成了threadLocal无法被回收。
3. 在没有手动删除这个Entry以及CuccrentThread依然运行的前提下，始终有强引用链threadRef->currentThread->threadLocalMap->entry，导致Entry内存泄漏、

也就是说，ThreadLocalMap中的key使用了强引用，是无法完全避免内存泄漏的。

#### 如果Key使用弱引用



![image-20210118221702719](/assets/img/notes/image-20210118221702719.png)

1. 还是假设在业务代码中使用完ThreadLocal，threadLocal Ref被回收了
2. 由于THreadLocalMap值持有THreadLocal的弱引用，没没有任何强引用指向threadLocal实例，所以threadLocal就可以顺利地被gc回收，此时Entry中的key = null。
3. 但是在没有手动删除这个Entry以及CurrentThread依然运行的前提下，在有强引用链 threadRef -> currentThread-> threadLocalMap -> entry -> value， value不会被回收，而这块value永远不会被访问到了，导致value 内存泄漏

也就是说，ThreadLocalMap中的key使用了弱引用，也有可能内存泄漏。

## 需要能回答的问题

Java中的引用类型有哪几种?

每种引用类型的特点是什么?

每种引用类型的应用场景是什么? ThreadLocal你了解吗

ThreadLocal应用在什么地方?  Spring事务方面应用到了

ThreadLocal会产生内存泄漏你了解吗?
