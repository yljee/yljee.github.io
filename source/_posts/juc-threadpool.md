---
title: juc-threadpool
date: 2020-10-26 21:24:43
tags:
---

##### ThreadPoolExecutor

###### ctl
+ 是存储两个概念域的原子整数
  * worker数量  有效线程数
    + 2^29-1 最大线程数
  * 运行状态（runState）  线程池的状态
    + 最高三位


###### runState的状态
+ RUNNING		接收新任务
+ SHUTDOWN  	不收新任务，执行队列任务
+ STOP			不收新任务，不执行队列任务，中断执行中的任务
+ TIDYING   	所有任务终止，worker数归零，会调用terminated()
+ TERMINATED	terminated()执行完毕


###### runState的变化
+ RUNNING -> SHUTDOWN
+ (RUNNING or SHUTDOWN) -> STOP
+ SHUTDOWN -> TIDYING
+ STOP -> TIDYING
+ TIDYING -> TERMINATED


##### Worker内部类
+ 实现Runnable接口、继承AQS类
+ 在run()方法中调用runWorker(this)


###### Worker对象的存储
~~~
  private final HashSet<Worker> workers = new HashSet<Worker>();
~~~
+ 注意事项
  + 所有worker都存放在set中
  + 只有**获取到锁**的时候才可以访问


###### ReentrantLock
~~~
  private final ReentrantLock mainLock = new ReentrantLock();
~~~
+ 主要目的是使得中断闲置线程的过程变得有序


##### 核心方法execute
###### 源码
~~~
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
~~~

###### execute方法的三个执行步骤
+ 来新任务时，未达到核心线程数，则创建核心线程
+ 如果已经存在了足量的核心线程，则新来的任务加入队列
  +  double-check 如果第一次检查后，线程池状态不在RUNNING，则进行回滚，清除刚才添加的任务
+ 如果队列满了，此时会添加新的worker  

###### addWorker方法
+ 返回是否成功添加worker
+ Worker通过new来创建
+ 将创建的Worker添加到内置的set中


###### 阻塞队列
~~~
  private final BlockingQueue<Runnable> workQueue;
~~~
+ 队列主要用来保存要提供给worker的任务



##### 核心方法runWorker
+ 方法重复从队列中取出任务并执行
+ 取出Worker对象创建时传入的任务，并执行
+ 在执行任务的run方法之前，会调用beforeExecute()方法
+ 在执行任务的run方法之后，会调用afterExecute()方法
+ 任务为空时，会调用getTask()方法从队列中取任务，并循环这个过程
~~~
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

~~~



##### 线程池的构造方法与参数

~~~
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler){
                                ...
                              }
~~~
+ 核心线程数
+ 最大线程数
+ 闲置存活时间
+ 时间单位
+ 阻塞队列
+ 线程工厂    
  * 一般采用默认的Executors.defaultThreadFactory()
+ 拒绝策略
  * DiscardOldestPolicy   直接丢弃队列头部元素，线程池接收该任务
  * AbortPolicy           直接抛出RejectedExecutionException
  * CallerRunsPolicy      由调用线程执行
  * DiscardPolicy         什么都不做，忽略任务
  