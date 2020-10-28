---
title: juc-basic
date: 2020-10-26 21:24:12
tags: juc

---

#####Future
+ Future类型的对象用来存储异步计算的结果
+ 调用get方法获取值，如果未计算完会*阻塞*当前线程
+ 可以调用cancel方法进行取消

#####RunnableFuture
+ 实现了Future和Runnable接口

#####FutureTask
+ 实现了RunnableFuture接口
+ 可取消的异步计算
+ 调用get方法获取值，如果未计算完会*阻塞*当前线程
+ 