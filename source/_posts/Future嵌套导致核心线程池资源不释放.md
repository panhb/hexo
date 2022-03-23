---
title: Future嵌套导致核心线程池资源不释放
date: 2022-3-15 18:00:00
tags: [java,线程池]
---

```java       
import cn.hutool.core.thread.NamedThreadFactory;
import lombok.SneakyThrows;

import java.util.concurrent.Future;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * @author hongbo.pan
 * @date 2021/11/16
 */
public class Test {

    @SneakyThrows
    public static void main(String[] args) {
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 50, 60, 
                TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000), 
                new NamedThreadFactory("测试", false));
        Future<String> f1 = poolExecutor.submit(() -> {
            System.out.println("===========111111===========");
            Future<String> f2 = poolExecutor.submit(() -> {
                System.out.println("===========222222===========");
                return "1";
            });
            System.out.println(f2.get());
            return "2";
        });
        System.out.println(f1.get());
    }
}
```     

<!-- more -->     
只会输出   
```java       
===========111111===========
```        
jstack看一下线程dump情况，这里只列测试线程和主线程   
```java         
"测试1" #14 prio=5 os_prio=0 tid=0x000000001f3ce000 nid=0x47b0 waiting on condition [0x000000001fb9e000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000076bdc8700> (a java.util.concurrent.FutureTask)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.FutureTask.awaitDone(FutureTask.java:429)
        at java.util.concurrent.FutureTask.get(FutureTask.java:191)
        at Test.lambda$main$1(Test.java:26)
        at Test$$Lambda$1/1142020464.call(Unknown Source)
        at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
        at java.util.concurrent.FutureTask.run(FutureTask.java)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
        at java.lang.Thread.run(Thread.java:748)

   Locked ownable synchronizers:
        - <0x000000076bc8e2e0> (a java.util.concurrent.ThreadPoolExecutor$Worker)



"main" #1 prio=5 os_prio=0 tid=0x0000000002894000 nid=0x2ba4 waiting on condition [0x00000000022ff000]
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x000000076bc8b0d0> (a java.util.concurrent.FutureTask)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at java.util.concurrent.FutureTask.awaitDone(FutureTask.java:429)
        at java.util.concurrent.FutureTask.get(FutureTask.java:191)
        at Test.main(Test.java:29)

   Locked ownable synchronizers:
        - None
```             

执行步骤
1.创建线程池poolExecutor 
2.poolExecutor异步提交任务，线程池核心线程数为0，线程池创建**测试1线程**执行代码
3.输出===========111111===========
4.poolExecutor异步提交任务，**测试1线程继续执行代码f2.get()，测试1线程阻塞，等待f2返回结果，此时线程池核心数为1，核心线程数满载，线程池将任务放进队列，不会创建线程执行**
5.f1.get()阻塞main线程，等待返回结果
6.main线程和测试1线程无限期等待

解决方案：
**1.队列大小设为0（线程池失去缓冲，不推荐）**
**2.future.get设置超时时间（可以防止阻塞，没有解决问题）**
**3.corePoolSize设置为maxPoolSize（浪费资源）**
**4.将嵌套的future换一个线程池执行（需要排查代码）**   

submit嵌套execute
```java        
@SneakyThrows
    public static void main(String[] args) {
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 50, 60,
                TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000),
                new NamedThreadFactory("测试", false));
        Future<String> f1 = poolExecutor.submit(() -> {
            System.out.println("===========111111===========");
            poolExecutor.execute(() -> {
                System.out.println("===========222222===========");
            });
            return "2";
        });
        System.out.println(f1.get());
    }
```       

输出
```java      
===========111111===========
2
===========222222===========
```     

执行步骤
1.创建线程池poolExecutor 
2.poolExecutor异步提交任务，线程池核心线程数为0，线程池创建**测试1线程**执行代码
3.输出===========111111===========
4.poolExecutor异步提交任务，**测试1线程继续执行，返回结果2，此时线程池核心数为1，核心线程数满载，线程池将任务放进队列，不会创建线程执行**
5.main线程输出f1结果2
6.测试1线程资源释放，线程池从队列中取出任务交给测试1线程执行
7.输出===========222222===========

future嵌套不get
```java    
@SneakyThrows
    public static void main(String[] args) {
        ThreadPoolExecutor poolExecutor = new ThreadPoolExecutor(1, 50, 60,
                TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000),
                new NamedThreadFactory("测试", false));
        Future<String> f1 = poolExecutor.submit(() -> {
            System.out.println("===========111111===========");
            Future<String> f2 = poolExecutor.submit(() -> {
                System.out.println("===========222222===========");
                return "1";
            });
//            System.out.println(f2.get());
            return "2";
        });
        System.out.println(f1.get());
    }
```     

输出
```java    
===========111111===========
2
===========222222===========
```    