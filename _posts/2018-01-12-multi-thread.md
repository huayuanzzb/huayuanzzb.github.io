---
layout: post
title:  "多线程"
author: recaton
categories: [ java ]
#image: assets/images/7.jpg
---

在多线程环境下，经常遇到多个线程要访问同一个资源的情况，每一个线程都可能对公共资源有读写操作，必须有一种机制能保证公共资源的正确性，这种机制通常是锁。

java 的synchronized实际上就是这样一种锁，它可以修饰方法，也可以修饰一个代码块。

当我们使用synchronized时，java内部使用monitor提供synchronization机制, monitor绑定到一个对象上。
当某个线程成功获取锁，我们称该线程进入了monitor。
在monitor中的线程退出monitor之前，其他试图进入monitor的线程只能等待。

synchronized 使用方式
1. 修饰实例对象
```java
private synchronized void test() {
    // do some work in this block
}
```
2. 修饰static方法
```java
private static synchronized void test() {
    // do some work in this block
}
```
3. 修饰代码块
```java
// monitor绑定在lock_object上
synchronized(lock_object){
    // do some work in this block
}
```
// TOOD 关于synchronized的继续补充
...

// 记录Object对象中的方法。
解决了公共资源的问题，随之而来便是线程间的通信问题，Object对象有三个方法wait(), notify(), notifyAll()便是为协调多线程间的工作而生的。
需要注意的是，这三个方法都必须配合synchronize才能工作，否则会报```java.lang.IllegalMonitorStateException```

* wait():使持有该对象锁的线程等待
* notify(): 由操作系统根据线程优先级选择一个正在该对象锁上等待的线程，并将其唤醒
* notifyAll(): 唤醒正在该对象锁上等待的所有线程
```java
import java.util.Date;

public class TestThread {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        Thread thread1 = new Thread(new TestThreadInternal(lock));
        Thread thread2 = new Thread(new TestThreadInternal(lock));
        thread1.start();
        thread2.start();
//        notifyBatch(lock);
        notifyOnebyOne(lock);
    }

    private static void notifyOnebyOne(Object lock) throws InterruptedException{
        for (int i = 0; i < 2; i++) {
            Thread.sleep(2000);
            synchronized (lock) {
                lock.notify();
            }
        }
    }

    private static void notifyBatch(Object lock) throws InterruptedException {
        Thread.sleep(2000);
        synchronized (lock) {
            lock.notifyAll();
        }
    }

    static class TestThreadInternal implements Runnable {

        Object lock;

        TestThreadInternal(Object lock) {
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock) {
                try {
                    System.out.println(String.format("%s - %s before wait...", new Date(), Thread.currentThread().getName()));
                    lock.wait();
                    System.out.println(String.format("%s - %s after wait...", new Date(), Thread.currentThread().getName()));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

}
```
调用```notifyOneByOne()```输出
```shell
Fri Jan 25 15:50:08 CST 2019 - Thread-0 before wait...
Fri Jan 25 15:50:08 CST 2019 - Thread-1 before wait...
Fri Jan 25 15:50:10 CST 2019 - Thread-0 after wait...
Fri Jan 25 15:50:12 CST 2019 - Thread-1 after wait...

Process finished with exit code 0
```
调用```notifyBatch()```输出
```shell
Fri Jan 25 15:54:51 CST 2019 - Thread-0 before wait...
Fri Jan 25 15:54:51 CST 2019 - Thread-1 before wait...
Fri Jan 25 15:54:53 CST 2019 - Thread-1 after wait...
Fri Jan 25 15:54:53 CST 2019 - Thread-0 after wait...

Process finished with exit code 0
```

上述代码中，可以看出```wait()```会抛出```java.lang.InterruptedException``` 查看```wait()```方法注释，当其他线程试图interrupt正在wait的线程时抛出。
```java
import java.util.Date;

public class TestThread {

    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();
        Thread thread1 = new Thread(new TestThreadInternal(lock));
        thread1.start();
        Thread.sleep(1000);
        // main线程interrupt正在wait的线程(thread1), 将会抛出InterruptedException
        thread1.interrupt();
    }

    static class TestThreadInternal implements Runnable {

        Object lock;

        TestThreadInternal(Object lock) {
            this.lock = lock;
        }

        @Override
        public void run() {
            synchronized (lock) {
                try {
                    System.out.println(String.format("%s - %s before wait...", new Date(), Thread.currentThread().getName()));
                    lock.wait();
                    System.out.println(String.format("%s - %s after wait...", new Date(), Thread.currentThread().getName()));
                } catch (InterruptedException e) {
                    System.out.println(String.format("%s - %s catch a InterruptedException", new Date(), Thread.currentThread().getName()));
                }
            }
        }
    }

}
```

那么什么是interrupt呢? 字面意思是“中断”，但其并不能中断正在执行的线程。
```interrupt()```是```java.lang.Thread```提供的方法，这个方法只是修改了线程内部的一个标志位(boolean类型),可以用过```isInterrupted()```方法查看。线程内部会检查该标志位做一些事情。
*```interrupt()```不会对正在执行的线程或者还未开始执行的线程产 任何影响
* ```interrupt()```只会对处于等待的线程产生影响

有哪些情况可以使线程进入等待呢？
来自于```Thread.interrupt()```代码注释的解释：
1. 如果线程被```object.wait()```, ```thread.join()```, ```thread.sleep()```block, 调用```interrupt()```会重置线程的interrupted状态，并抛出```InterruptedException```
2. 如果线程被```java.nio.channels.InterruptibleChannel```block, 调用```interrupt()```会关闭channel，重置线程的interrupted状态，并抛出```java.nio.channels.ClosedByInterruptException```
3. 如果线程被```java.nio.channels.Selector```block, 调用```interrupt()```会重置线程的interrupted状态，并直接返回

上面的描述中又牵扯出了```thread.join()```，这个方法是要告诉调用线程，要等待被调用线程结束再继续执行。
```java
import java.util.Date;

public class TestThread {

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(new TestThreadInternal());
        thread1.start();
        thread1.join();
        System.out.println(String.format("main thread will exit"));
    }

    static class TestThreadInternal implements Runnable {

        @Override
        public void run() {
            try{
                System.out.println(String.format("%s - %s start sleep", new Date(), Thread.currentThread().getName()));
                Thread.sleep(3000);
                System.out.println(String.format("%s - %s wake up", new Date(), Thread.currentThread().getName()));
            }catch (InterruptedException e){
                e.printStackTrace();
            }
            }
        }
}
```
如果没有```thread1.join()```，输出
```shell
main thread will exit
Fri Jan 25 17:26:20 CST 2019 - Thread-0 start sleep
Fri Jan 25 17:26:23 CST 2019 - Thread-0 wake up
```
如果有```thread1.join()```，输出
```shell
Fri Jan 25 17:24:49 CST 2019 - Thread-0 start sleep
Fri Jan 25 17:24:52 CST 2019 - Thread-0 wake up
main thread will exit
```

```ReentrantLock```有四个获取锁的方法：
* ```void lock();```: 在等待获取锁的过程中休眠并禁止一切线程调度
* ```void lockInterruptibly() throws InterruptedException;```: 在等待获取锁的过程中可以被中断
* ```boolean tryLock();```: 获取到锁返回true, 否则返回false
* ```boolean tryLock(long time, TimeUnit unit) throws InterruptedException;```: 在指定时间内等待获取锁；过程中可被中断
假如线程A和线程B使用同一个锁LOCK，此时线程A首先获取到锁LOCK.lock()，并且始终持有不释放。如果此时B要去获取锁，有四种方式：

* LOCK.lock(): 此方式会始终处于等待中，即使调用B.interrupt()也不能中断，除非线程A调用LOCK.unlock()释放锁。

* LOCK.lockInterruptibly(): 此方式会等待，但当调用B.interrupt()会被中断等待，并抛出InterruptedException异常，否则会与lock()一样始终处于等待中，直到线程A释放锁。

* LOCK.tryLock(): 该处不会等待，获取不到锁并直接返回false，去执行下面的逻辑。

* LOCK.tryLock(10, TimeUnit.SECONDS)：该处会在10秒时间内处于等待中，但当调用B.interrupt()会被中断等待，并抛出InterruptedException。10秒时间内如果线程A释放锁，会获取到锁并返回true，否则10秒过后会获取不到锁并返回false，去执行下面的逻辑。

