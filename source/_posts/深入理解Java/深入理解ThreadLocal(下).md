---
title: 深入理解ThreadLocal(下)
date: 2019-04-29 00:15:13
tags:
- java
- multithread
- threadlocal
- thread safe
- 线程
- 线程安全
categories:
- 深入理解Java
---
> 不可去名上理会。须求其所以然。  ——朱熹

ThreadLocal帮助存储线程私有变量，往往作为线程级别的全局变量使用（线程之内全局可见，线程之间互不可见），能够在多个方法之间传递状态，帮助方法减少参数。那么ThreadLocal是如何实现线程私有存储，所谓的ThreadLocal内存泄漏问题又是怎么回事儿呢？本文尝试从源码角度探索ThreadLocal的原理，之后从原理的角度，用一个例子复现ThreadLocal的内存泄漏问题，希望能知其然知其所以然。

# ThreadLocal如何实现线程私有
这个问题我在面试的时候经常问候选人，答对的真的很少。大部分的答案是：ThreadLocal维护了一个Map，用当前线程作为key，key在Map中对应的value就是这个线程的私有value。如果没有看过ThreadLocal的代码，让我们自己实现一个ThreadLocal的话，很有可能就是按照这个思路实现。其实在C语言中，就是用类似的方式实现线程私有存储的：先定义一个数组类型的全局变量，这个变量全局可见，所有线程可以访问，例如：
```c
global * thread_specific[THREAD_MAX_NUM]
```
然后每个线程定义一个线程私有的int型变量：
```c
__thread int thread_id;
```
启动线程的时候为每个线程分配全局唯一的id，线程将id存储在上面的私有变量里面。写入和读取线程私有变量的时候，通过线程id在`thread_specific`组中索引自己的私有变量。
Java里面也是能这么实现的，但是看了ThreadLocal代码就知道：Java并不是这么做的，为什么呢？

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
上面的代码，有个获取当前线程的Map方法调用，跟进去发现只有一行代码：
```java
   ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
   }
```
到这里基本清楚了，ThreadLocal实现线程私有的方式是：每个线程有一个线程私有的Map，用ThreadLocal实例作为key，到这个Map中索引线程自己的私有变量。Java为什么要用这种方式实现ThreadLocal呢，用这种方式就能防止程序员犯错，避免线程访问其他线程的变量了么？的确，还真是如此。`Thread.threadLocals`中的threadLocals是包级别可见的，普通用户代码无法直接访问（反射除外），因此也就无法直接修改其他线程的数据了，用这种方式正常情况下只能访问当前线程的私有数据。

这种实现策略体现了语言设计的思想差异，仔细想想还是蛮有意思的。C语言常用的方式，用对了没问题，但是这种方式并不限制线程访问和修改其他线程的变量，实际上如果愿意，一个线程可以访问其他所有线程的变量：只要遍历`thread_specific`数组就好了。C语言的设计者认为程序员是明智的，因此赋予程序员最大的自由。而Java并非如此，Java语言认为如果给程序员犯错的机会，那么就一定会犯错，无论看上去是如何不可能或者离谱的错误，因此应该尽量避免给犯错的机会，例如引入多种变量方法可见性机制、垃圾回收、数组禁止越界等。

# ThreadLocal与内存泄漏

# ThreadLocal与泛型
