id: java3
title: Java多线程
categories: core java
tags: [thread,java]
---

## 什么是线程
> 线程是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

上面是网络上对“线程”的解释，可以看到线程具有以下特点：
- 被包含在进程中， 那么问题来了：什么是进程？大学老师曾问过同样的问题，当时我在座位上答道：运行中的程序。*百度百科* 进程（Process）是计算机中的程序关于某数据集合上的一次运行活动，是系统进行资源分配和调度的基本单位，是操作系统结构的基础。
- 线程是单一顺序的控制流，朴素来说线程中执行的代码都是有确定的先后顺序的，每一步执行的环境都是确定的，没有不确定的因素。
- 线程可以并发执行，在一个java进程中的不同任务可以不必彼此等待，常见的场景有阻塞式IO、网络通信、界面交互等。

## 线程的基本语法
在JAVA中创建一个线程并执行任务有两种方式，继承`Thread`类或实现`Runnable`接口并重写`run()`方法。

### 继承Thread
```java
class OneThread extends Thread {
    int sleep;
    OneThread(int sleep) {
        this.sleep = sleep;
    }
    @Override
    public void run() {
        while (true) {
            System.out.println("大家好,我是一个线程,我的名字叫:" + getName());

            try {
                TimeUnit.MILLISECONDS.sleep(sleep);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
在`run`方法中编写线程的任务，请随意！
启动线程：
```java
OneThread oneThread1 = new OneThread(411);
oneThread1.run();
```
很抱歉，上面是**错误**的启动方式，正确启动线程一定要调用`start()`方法：
```java
OneThread oneThread1 = new OneThread(411);
OneThread oneThread2 = new OneThread(557);
oneThread1.start();
oneThread2.start();
```
感兴趣的娃子可以自己去执行看看；

### 实现Runnable接口
```java
class OneRunnable implements Runnable {
   int sleep;
   OneRunnable(int sleep) {
       this.sleep = sleep;
   }

   @Override
   public void run() {
       while (true) {
           System.out.println("大家好,我是一个线程,我的名字叫:" + Thread.currentThread().getName());

           try {
               TimeUnit.MILLISECONDS.sleep(sleep);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       }
   }
}
```
仔细看看**区别**，区别不大哈哈。
如何启动呢？
```java
new Thread(new OneRunnable(411)).start();
new Thread(new OneRunnable(557)).start();
```

### 为什么两种都可以？
我们先看`Thread`类的构造方法之一：
```java
public Thread(Runnable target) {
    // 线程的默认名字就是这么来的
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
...
  this.target = target;
...
}
```
构造`Thread`对象时出入的`Runnable`被保存到`target`对象了，然后呢？ 回顾上面创建线程的方法发现，奇怪的地方是`run()`
```java
@Override
public void run() {
    if (target != null) {
        target.run();
    }
}
```
真相大白了，原来`Thread`默认的run方法就是调用`target.run()`。这就是为什么继承`Thread`和实现`Runnable`都可以创建线程。
细心的娃子们会注意到，`Thread.run()`上面有`@Override`， 为啥呢？
```java
public class Thread implements Runnable {...}
```
`Thread`也是`Runnable`的一个实现类。

## 线程的状态
![Java线程状态](http://7xrbi6.com1.z0.glb.clouddn.com/img-%E7%BA%BF%E7%A8%8B%E7%8A%B6%E6%80%81.jpg)

### 新建
新建了一个线程对象。 `new Thread()`
### 可运行
也叫就绪状态。线程对象创建后被调用了`start()`方法后进入**可运行**状态。此时表示线程随时可以执行，等待线程调度选中获得CPU使用权
### 运行
在可运行状态的线程获得了CPU时间片，执行程序代码
### 阻塞
阻塞状态是指线程因为某种原因放弃了CPU的使用权，暂停运行。知道线程进入**可运行**状态。阻塞分三种情况：
- 等待阻塞：**运行** 的线程执行了`o.wait()`方法， JVM会把该线程放到等待队列中。
- 同步阻塞：**运行** 的线程在获取对象的同步锁时，若该对象的锁被其他线程占用，则JVM会把改下昵称放入锁池中。
- 其他阻塞：**运行** 的线程执行`Thread.sleep(long ms)`或`t.join()`方法，或者发出了I/O请求时， JVM会阻塞该线程。
### 死亡
线程`run()`结束或者遭遇一场，则结束该线程的生命周期。死亡的线程不能复生。

## 共享
在多线程环境，如果多线程一起访问公共资源没有任何限制，事情就不可控了。具体例子就不列举了，大家都懂。
因此在多线程环境，我们要用一种方式来控制资源在关键时刻只能被单一线程依次占有，不能一起上。

### volatile
如果一个域声明为`volatile`，那么只要对这个域产生了写操作，所有的读操作就都可以看到这个修改。
在64位JVM中， long和double的读取和写入不是原子性的， 请加上`volatile`。

### Lock
手动的加锁与释放，这很灵活
```java
public int next() {
   lock.lock();
   try {
       ++ currentEvenValue;
       Thread.yield();
       ++ currentEvenValue;
       return currentEvenValue;
   } finally {
       lock.unlock();
   }
}
```
一定要在`finally`里释放哦，`return`语句要在释放锁之前，不然不是完全的原子性。

### synchronized
最常用的加锁方式。可使同步块内的代码原子性执行。
```java
// 同步方法
public synchronized void test() {...}
// 同步块
synchronized(obj) {...}
```

### 锁的是什么？
能被锁的只能是对象， 那么像上述`lock`和`synchronized`锁定的是什么呢？是当前对象`this`。其实整个过程是，线程1执行到同步方法时，去获得对象的锁，如果能获得则继续执行，并在执行结束时释放锁；如果锁已被别的线程占有则等待，等待别的线程释放锁。
所以下面的代码执行情况是？
```java
public class SyncTask {
    public synchronized void f1(){
        System.out.println("f1 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("f1 end");
    }
    public synchronized void f2(){
        System.out.println("f2 start");
    }

    public static void main(String[] args) {
        SyncTask task = new SyncTask();
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(() -> task.f1());
        executorService.execute(() -> task.f2());
        executorService.shutdown();
    }
}

output:
f1 start
f1 end
f2 start
```
此处`f1()`和`f2()`的锁都是对象本身， 所以线程1先执行`f1()`时获得了`task`的锁， 线程2执行`f2()`时去请求此锁只能等待，就出现了好像顺序执行的情况。
稍作处理，我们可以看到不同的情况：
```java
public class SyncTask {
    Object a = new Object();
    public void f3(){
        synchronized (a) {
            System.out.println("f1 start");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("f1 end");
        }
    }
    public synchronized void f2(){
        System.out.println("f2 start");
    }

    public static void main(String[] args) {
        SyncTask task = new SyncTask();
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(() -> task.f3());
        executorService.execute(() -> task.f2());
        executorService.shutdown();
    }
}

output:
f1 start
f2 start
f1 end
```
可以看到线程同时执行了。因为线程1此时锁的对象是`a`， 线程2锁的对象上`task`，两者没有冲突无需等待。
那么下面的情况会怎样？娃子们自己试试吧
```java
public class SyncTask {
    Object b;

    public void f3(){
        synchronized (b) {
            System.out.println("f1 start");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("f1 end");
        }
    }
    ...
}
```

## 线程的一些小点
### 守护线程
JAVA中线程有两类**用户线程**和**守护线程**。
所谓的守护线程，是指用户程序在运行的时候后台提供的一种通用服务的线程，比如用于垃圾回收的垃圾回收线程。这类线程并不是用户线程不可或缺的部分，只是用于提供服务的"服务线程"。
```java
// 设置守护线程
thread.setDaemon(true);
```
需要注意的地方：
- `thread.setDaemon(true)`必须在`thread.start()`之前设置，否则会跑出一个异常。你不能把正在运行的常规线程设置为守护线程。
- 在守护线程中产生的线程也是守护线程。
- 我们自己产生的守护线程应该避免访问一些类似于文件、数据库等固有资源，因为由于JVM没有用户线程之后，守护线程会马上终止。
对于第二点我们可以通过`Thread`类的源码来证明：
```java
class Thread {
  init() {
    ...
    this.daemon = parent.isDaemon();
    ...
  }
}
```
### 线程优先级
线程优先级取值是1-10， 越大优先级越高。
线程的优先级无法保障线程的执行次序。只不过，优先级高的线程获取CPU资源的概率较大，优先级低的并非没机会执行。
```java
thread.setPriority(10);
```
