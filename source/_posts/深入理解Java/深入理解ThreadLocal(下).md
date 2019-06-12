---
title: 深入理解ThreadLocal(下)
date: 2019-04-29 00:15:13
categories:
- 深入理解Java
tags:
- 原创
- java
- multithread
- ThreadLocal
- thread safe
- 线程
- 线程安全
---
> 不可去名上理会。须求其所以然。  ——朱熹

ThreadLocal帮助存储线程私有变量，往往作为线程级别的全局变量使用（线程之内全局可见，线程之间互不可见），能够在多个方法之间传递状态，帮助方法减少参数。那么ThreadLocal是如何实现线程私有存储，所谓的ThreadLocal内存泄漏问题又是怎么回事儿呢？本文尝试从源码角度探索ThreadLocal的原理，之后从原理的角度，用一个例子复现ThreadLocal的内存泄漏问题，希望能知其然知其所以然。

# ThreadLocal如何实现线程私有
这个问题可能很多人的答案是：ThreadLocal维护了一个Map，用当前线程作为key，对应的value就是这个线程的私有value。如果没有看过ThreadLocal的代码，让我们自己实现一个ThreadLocal的话，很有可能就是按照这个思路实现。其实在C语言中，就是用类似的方式实现线程私有存储的：先定义一个数组类型的全局变量，这个变量全局可见，所有线程可以访问，例如：
```c
foo_bar_type * thread_specific[THREAD_MAX_NUM]
```
然后每个线程定义一个线程私有的int型变量：
```c
__thread int thread_id;
```
启动线程的时候为每个线程分配全局唯一的id，存储在`thread_id`中。写入和读取线程私有变量的时候，通过`thread_id`在`thread_specific`组中索引自己的私有变量。这种方式有个问题：线程可以自由访问其他线程的私有变量，只要程序员愿意，其实可以随意修改其他线程的“私有”变量，从Java的角度看，这种方式其实很危险。

因此看了ThreadLocal代码就知道：Java并不是这么做的，那么Java是做么做到线程私有存储的呢，有没有其他的缺点呢？

先看看Java是怎么实现，直接上代码，这是ThreadLocal的get方法：
```java
    public T get() {
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 从当前线程获得一个Map
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            // 用当前ThreadLocal实例作为key，查出线程的私有变量
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        // 初始化当前线程的ThreadLocalMap
        return setInitialValue();
    }
```
上面的代码，先获得当前线程，再调用`getMap(t)`获取当前线程的私有Map，跟进去发现只有一行代码：
```java
   ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
   }
```
到这里基本清楚了，ThreadLocal实现线程私有的方式是：每个线程有一个自己私有的Map，用ThreadLocal实例作为key，到自己的私有Map中查找线程私有变量。那么Java为什么要用这种方式实现ThreadLocal呢，用这种方式就能防止程序员犯错，避免线程访问其他线程的变量了么？的确，还真是如此。`Thread.threadLocals`中的`threadLocals`是包级别可见的，普通用户代码无法直接访问（反射除外），因此也就无法直接修改其他线程的数据了，用这种方式正常情况下线程只能访问自己的私有数据。

这种实现策略体现了编程语言设计的思想差异，仔细想想还是蛮有意思的。C语言实现线程私有的方式，直白而简单，用对了没问题，但是这种方式并不限制线程访问和修改其他线程的变量，比较“危险”。C语言的设计者认为程序员是明智的，因此赋予程序员最大的自由。而Java并非如此，Java语言认为如果给程序员犯错的机会，无论看上去是如何不可能或者离谱的错误，一定会有人犯这样的错。因此语言应该尽量避免给程序员犯错的机会，例如除了ThreadLocal的实现方式，Java拥有多种变量和方法的可见性机制、内存自动回收的垃圾回收器、数组越界检查等，都是希望杜绝一些基本的犯错机会。

# ThreadLocalMap
ThreadLocal使用的Map是当前线程的私有的Map`threadLocals`，它的类型是`ThreadLocalMap`，这个Map与我们常用的`HashMap`有很大的不同，它是专门为ThreadLocal设计的，用户代码无法使用，`ThreadLocalMap`是包级别可见的，它的方法都是私有的，只有它的外部类ThreadLocal能够使用。`threadLocals`是线程的属性，只要线程一直存在，作为线程的强引用属性，`threadLocals`也就一直不能被GC回收，因此它的生命周期可能很长，针对这一特点，`ThreadLocalMap`有一些特别的设计，它的的整体结构如下图：
![ThreadLocalMap](ThreadLocalMap.jpg "ThreadLocalMap")
ThreadLocalMap内部使用一个数组存储Entry数据，数组的默认长度是16（`ThreadLocalMap.INITIAL_CAPACITY`，必须是2的整数次幂），这与HashMap相似，不同的是，在put操作遇到冲突的时候，使用开放地址探测解决，而不是HashMap的链表方式。例如

此外，`ThreadLocalMap`与`HashMap`的`Entry`设计也有很大不同。`ThreadLocalMap`的生命周期往往很长，可能导致一些应该被回收的内存，由于`ThreadLocalMap`的引用而无法回收，导致内存泄漏。为了解决这个问题，`ThreadLocalMap.Entry`的key是弱引用，上图中entry到key的的线是虚线表示了这种关系，在虚拟机内存不足发生GC的时候，弱引用不阻碍回收。但是`ThreadLocalMap.Entry`只有key是弱引用，value是强引用，这是因为`ThreadLocalMap.Entry`能否可以被回收取决于key，只要key被外部引用，整个entry必须保持可用，因此value必须是强引用，如果value也是弱引用，可能导致用户put到`ThreadLocal`中的变量随时被虚拟机回收。

`ThreadLocalMap`与`HashMap`另一个很大的不同就是key的相等判断。Java的集合类使用`equals`判断两个对象是否相等，而`ThreadLocalMap`使用`==`（必须不等于null并且是同一个`ThreadLocal`实例），这么做的原因是：对`ThreadLocal`用户而言，ThreadLocal之间的隔离是实例级别的，两个不同的ThreadLocal实例，是相互隔离互不影响的，如果使用`equals`就可能导致两个不同的`ThreadLocal`被认为“相等”从而关联相同的变量相互干扰。

`ThreadLocalMap`与`HashMap`还有一个不同点就是解决哈希冲突的方式，`HashMap`使用链表解决哈希冲突（当链表长度大于8时优化成红黑树），而`ThreadLocalMap`使用开放地址方式解决哈希冲突。从功能上看，我认为这两种方式都能实现`ThreadLocalMap`的能力，但是为什么这里用开放地址探测方式解决哈希冲突呢？我们可以对比下这两种哈希冲突解决方法：

| 特性             | 链表式 | 开放地址探测式     |
|------------------|--------|--------------------|
| cache友好        | 差     | 好                 |
| 内存占用         | 低     | 高                 |
| 查询写入性能     | 中     | 低装载高，高装载低 |
| 默认扩容装载率阈 | 75%    | 66.6%              |
| 通用场景适配     | 强     | 弱                 |

开发地址探测方式，由于使用数组存储对象，对cache友好，在装载率比较低的情况下性能比链表要高，但是随着装载率的提高查询写入效率急剧下降，每次几乎都要遍历全表，因此为了保证性能，开放地址探测方式解决哈希冲突，默认的扩容装载率比较低，相对而言就会占用更大的内存，通用性不如链表好。`ThreadLocal`的场景，一般变量数量不会太多，而`HashMap`可能动辄几万甚至几十万上百万的KV，因此`HashMap`不适合用开放地址探测，反过来，`ThreadLocal`使用开放地址探测，在付出极地的内存代价之后，能获得更高的性能，我认为这是`ThreadLocalMap`使用开放地址探测解决哈希冲突的根本原因。
# ThreadLocal与内存泄漏
尽管`ThreadLocalMap.Entry`的key是弱引用，但是value并不是，可能导致key被回收，而value仍然GC ROOT强可达无法回收。此时`ThreadLocal`没有额外操作的话，value不会从`ThreadLocalMap`中删除，这实际上导致了内存泄漏：
1. `ThreadLocalMap.Entry.value`对用户代码是无用的，用户代码已经无法访问到该内存区域
2. `ThreadLocalMap.Entry.value`是GC ROOT强可达，内存无法回收
这种内存泄漏的场景，可以用下面的代码复现：
```java
/**
 * Run with jvm parameters: -Xms200m -Xmx200m
 */
public class ThreadLocalTest {
    static class BigObject {
        private byte[] data;

        public BigObject() {
            // 100M
            this.data = new byte[100 << 20];
        }
    }

    public static void main(String[] args) throws InterruptedException {
        foobar();

        try {
            new BigObject();
        } catch (OutOfMemoryError e) {
            // 预计抛出次异常，线程睡眠供外部探查内存使用情况
            e.printStackTrace();
            Thread.sleep(10000000L);
        }
    }

    private static void foobar() {
        new ThreadLocal<BigObject>().set(new BigObject());
    }
}
```
为了方便复现问题，执行的时候把JVM的堆设置小点儿，例如`-Xms200m -Xmx200m`，执行上面的代码，预期抛出如下的OutOfMemoryError：
```text
java.lang.OutOfMemoryError: Java heap space
	at xxx.java.ThreadLocalTest$BigObject.<init>(ThreadLocalTest.java:12)
	at xxx.java.ThreadLocalTest.main(ThreadLocalTest.java:20)
```
然后主线程睡眠，此时可以用Visualvm分析JVM进程的内存情况，如下图：
![ThreadLocal内存泄漏](java-threadlocal-memory-leak.jpg "ThreadLocal内存泄漏")
与预期的结果一致：我们刻意创建的大数组占用了98.4%的内存，下面我们看看类的引用关系：
![GC ROOT强可达value，导致内存泄漏](thread-gcroot-strong-reference.jpg "GC ROOT强可达value，导致内存泄漏")
可以看到GC ROOT的引用链：
Thread->Thread$ThreadLocalMap->table->Thread$ThreadLocalMap#Entry，与我们之前代码的分析是一致的，此外，Entry的弱引用key（字段名称是referent，弱引用需要继承WeakReference实现，referent是WeakReference的属性）是null，表示这个`ThreadLocal`实例已经被GC垃圾回收掉了。

# 总结
`ThreadLocal`是Java提供的线程私有存储(Thread-local storage TLS)方案，本文尝试分析了其内部实现，相比C语言常用的TLS方案，对程序员更友好，此外为了提升性能，内部的Map使用了开放地址探测方式解决哈希冲突，为了避免不必要的内存占用，Map的Entry使用弱引用管理key。在使用`ThreadLocal`的时候，需要注意可能的内存泄漏问题，笔者使用了一个简单的例子复现了内存泄漏场景，在使用`ThreadLocal`的时候建议遵循两个原则：
1. `ThreadLocal`实例一定是用于多线程共享的，否则不需要`ThreadLocal`的
2. 在明确不需要使用某个`ThreadLocal`内变量的时候，可以显示调用`ThreadLocal.remove`方法释放内存。
3. 作为一个变量容器，`ThreadLocal`也是泛型类型，帮助在编译器发现问题，具体使用的时候也一定要使用泛型

# 参考资料
1. Wikipedia-Memory Leak: https://en.wikipedia.org/wiki/Memory_leak
2. visualvm: https://visualvm.github.io/
