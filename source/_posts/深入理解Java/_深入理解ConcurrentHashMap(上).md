---
title: 深入理解ConcurrentHashMap(上)
date: 2019-05-16 23:05:49
tags:
- 原创
- java
- multithread
- ConcurrentHashMap
- thread safe
- 线程
- 线程安全
categories:
- 深入理解Java
---
> 不要把鸡蛋放在一个篮子里。 ——中国谚语

ConcurrentHashMap，顾名思义，是并发场景下的哈希表，支持多线程并发读写的Map。从Java1.5开始由大神Doug Lea引入JDK，最新的Java8做了比较大的改动和升级，笔者尝试讲清楚Java8版本的ConcurrentHashMap的实现和原理。

# 与Hashtable的关系
Hashtable是Java1.5之前提供的线程安全的哈希表，那么ConcurrentHashMap与Hashtable的关系是怎样的呢？一句话来说绝大多数场景，ConcurrentHashMap与Hashtable是可以互换的，从Hashtable换成ConcurrentHashMap的好处是可以提升代码的可伸缩性和性能。因此在需要多线程安全的哈希表时，应该尽量使用ConcurrentHashMap，至于为什么ConcurrentHashMap能提升性能，正是笔者尝试通过阅读代码来发现和解读的。有没有一些场景，Hashtable与ConcurrentHashMap不能互换呢，的确是有的。从实现原理上Hashtable通过锁全表的方式实现多线程读写安全，但是ConcurrentHashMap没有提供锁全表的能力，所有的读操作是无锁的，写操作只锁部分数据而不锁全表，如果依赖Hashtable锁全表的特定实现，是无法用ConcurrentHashMap进行替换的。

#
