---
title: 深入理解ThreadLocal(上)
date: 2019-04-27 22:00:41
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

Java的ThreadLocal是Java语言提供的线程私有变量解决方案，一般的业务开发可能很少用到，但是在框架（例如Spring）或者中间件（例如分布式调用跟踪）等系统中非常常见。深入理解ThreadLocal对理解框架或者中间件的代码很有帮助，是Java进阶的必经之路。笔者一直认为理解原理的最好方法就是看代码。笔者在公司笔者面试了不少候选人，大部分同学对ThreadLocal的理解往往不准确，因此者希望写一篇短文，分享一下自己的理解。

## 为什么要用ThreadLocal
在什么场景下需要使用ThreadLocal呢，首先一定是多线程场景，如果没有用到多线程，一定是不需要ThreadLocal的。引入ThreadLocal意味着引入更多的复杂性，如果是单线程场景，完全没有必要使用ThreadLocal。而且我认为能用单线程解决的，就不要用多线程。实际上像是Redis，为了保持简单，就是用的单线程（主流程是单线程的，其他线程只是辅助，例如定期调用操作系统接口强制缓存落盘，但是辅助线程不参与主流程的逻辑），Redis使用一个线程基于IO多路复用（epoll，kqueue）服务所有的客户端，这种简单性虽然牺牲了利用CPU多核心并发的能力，但是得到了非常简单的架构实现和健壮性，CPU并发利用的问题可以通过多实例的方式绕过。笔者一直认为简单是架构的目标，简单能够带来更好的可维护性和健壮性。

如果必须使用多线程，在多线程场景下，用ThreadLocal往往真正的目的只有一个：
利用ThreadLocal变量存储线程私有的全局变量，从而减少方法之间的参数传递和声明。例如Spring的事务管理器实现，由于JDBC的一个局部事务内的所有操作必须共用一个数据库连接，因此在执行事务范围内的多个数据库操作时，必须确保使用相同的数据库连接。为了做到这一点，可以在每个数据库操作方法生命上增加一个数据库连接参数，显然这种方式对客户不友好，为了避免这种问题，Spring使用ThreadLocal存储数据库连接，由于ThreadLocal变量全局可见，而每个线程又相互隔离，正好解决了这个问题。在实际业务开发的时候，也有这种case，典型的就是用ThreadLocal存储用户信息，这样下游的服务可以假定用户已经登录，从ThreadLocal中获得用户信息，而不是每个方法都需要增加一个用户参数一层层传递下来。

## 怎么用ThreadLocal
ThreadLocal是一个泛型类，就像Java的集合类，ThreadLocal也是一个容器，只是这个容器为每个线程存储一个变量，多线程时每个线程访问自己的变量，互不干扰。
```java
// 创建一个ThreadLocal实例
ThreadLocal<FooBar> threadLocal = new ThreadLocal<FooBar>();
// 然后set变量
FooBar threadFooBar = ...
threadLocal.set(threadFooBar)
...
// 其他地方就能通过threadLocal实例的get方法获得之前set的变量了
FooBar threadFooBar = threadLocal.get()
```
可以看到ThreadLocal对外暴露的接口还是比较简单的，似乎用起来很容易，但是就像有句话说的：
> 生活从来都不容易，当你觉得容易的时候，一定是有人在替你承担属于你的那份不容易。

ThreadLocal的实现其实有一定复杂性，而且ThreadLocal要真正用好用对，其实是不容易的，下面讲讲我认为的ThreadLocal最佳实践。

## 最佳实践
一般我们建议把ThreadLocal变量声明成`private static final`，然后通过`public`的静态方法向外部提供服务。例如下面的场景：为线程分配一个整数类型的id：
```java
import java.util.concurrent.atomic.AtomicInteger;

public class ThreadId {
    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
        new ThreadLocal<Integer>() {
            @Override protected Integer initialValue() {
                return nextId.getAndIncrement();
        }
    };

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }
    // Remove value
    public void remove() {
        threadId.remove();
    }
}
```
之所以声明成private，是为了隐藏数据模型，对外部暴露必要的接口即可；声明成static，是为了在多线程之间共享，可以通过static的方式对外暴露服务，这样所有线程都可以直接访问服务；声明成final主要是避免了多线程的内存可见性问题，确保所有线程访问的ThreadLocal实例是同一个。

除了声明方式，最好提供`remove`方法，如果在ThreadLocal中存储的变量占用的内存很大，那么在确定不用的时候，最好手动删除，这是因为在线程一直running的情况下，ThreadLocal可能导致内存一直被占用而得不到释放，这其实是ThreadLocal的一个坑，具体见下篇文章解析。
