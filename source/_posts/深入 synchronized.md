---
title: 深入 synchronized
date: 2018/6/15 9:27:0
tags: [Java,Java 多线程]
categories: [Java 多线程]
---

Java 的锁具体表现为 `synchronized` 和 `Lock`，`synchronized` 依赖的是对象的内置锁（或者叫监视器锁），而 `Lock` 依赖的是 `AQS`，本文只讨论基于监视器锁的互斥同步实现。  

<!--more-->  

# 锁的内存语义

锁除了能让临界区互斥执行外，还有一个重要的语义就是发送消息。锁的获取和释放与 `volatile` 的读和写有相同的内存语义。  

线程 A 释放锁时，会将共享变量的值同步回主内存中，实质上是线程 A 向接下来将要获取锁的某个线程发出了消息（线程 A 对共享变量所做修改的消息）。  

线程 B 获取到锁时，会将该线程的本地工作内存置为无效，临界区中共享变量的值需要从主内存读取，实质上是线程 B 接收了之前某个线程发出的消息（之前的某个线程在释放锁之前对共享变量所做修改的消息）。  

线程 A 释放锁，随后线程 B 获取这个锁，这个过程实质上是线程 A 通过主内存向线程 B 发送消息。  

# synchronized 的使用

`synchronized` 有三种使用方式：  

- 普通同步方法  
锁对象是当前实例对象，原理是虚拟机通过方法常量池的方法表结构中的 `ACC_SYNCHRONIZED` 访问标志得知一个方法是否为同步方法。在方法调用时，调用指令会检查方法的 `ACC_SYNCHRONIZED` 访问标志是否被设置，如果设置了，则执行线程需要先成功持有管程，然后才能执行方法，当方法完成，释放管程。  
- 静态同步方法  
锁对象是当前类的 class 对象，原理和普通同步方法一样。  
- 同步代码块  
锁对象是括号中指定的对象，使用 `monitorenter` 和 `monitorexit` 指令实现。  

## 一些规则

- 当一个线程访问一个同步方法时，其他线程可以正常访问非同步方法。  
- 同一个锁的同步代码块或同步方法同一时刻只能被同一个线程访问。  
- 当一个线程访问一个同步方法时，其他线程不能访问该锁对象的其他同步方法。  
```java
public class Demo {

    public synchronized void doSomething() {
        System.out.println(Thread.currentThread().getName() 
        + " doSomething start");
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() 
        + " doSomething end");
    }

    public synchronized void doAnotherThing() {
        System.out.println(Thread.currentThread().getName() 
        + " doAnotherThing start");
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() 
        + " doAnotherThing end");
    }

    public static void main(String[] args) {
        Demo demo = new Demo();
        Thread t1 = new Thread(() -> demo.doSomething(), "thread-1");
        Thread t2 = new Thread(() -> demo.doAnotherThing(), "thread-2");

        t1.start();
        t2.start();
    }
}
```

- 由于 String 字面量会在编译后放入运行时常量池中，所以两个相同的 String 字面量是同一个对象，在使用 String 字面量作为锁对象时要注意。  
- 线程之间需要相互等待对方释放已持有的锁时，会形成死锁。  
```java
public static void main(String[] args) {
    final Object lock1 = new Object();
    final Object lock2 = new Object();

    Thread t1 = new Thread(() -> {
        synchronized (lock1) {
            System.out.println(Thread.currentThread().getName() 
            + " get lock1");

            synchronized (lock2) {
                System.out.println(Thread.currentThread().getName() 
                + "get lock2");
            }
        }
    }, "thread-1");

    Thread t2 = new Thread(() -> {
        synchronized (lock2) {
            System.out.println(Thread.currentThread().getName() 
            + " get lock2");

            synchronized (lock1) {
                System.out.println(Thread.currentThread().getName() 
                + "get lock1");
            }
        }
    }, "thread-2");

    t1.start();
    t2.start();
}
```

# 实现原理

Java 的同步机制是对 `Monitor Object` 设计模式的封装，关于该模式可以参考[这篇文章 ](https://www.ibm.com/developerworks/cn/java/j-lo-synchronized/)。  

我们已经知道 `synchronized` 代码块在编译后，会在代码块前后生成 `monitorenter` 和 `monitorexit` 两个字节码指令；同步方法在编译后会在 Class 文件常量池中的方法表结构中存储 `ACC_SYNCHRONIZED` 符号引用，在方法调用时调用指令会检查方法表中 `ACC_SYNCHRONIZED` 的访问标志是否被设置。  

这两种实现都需要一个引用参数作为锁对象，这与 Java 对于 `Monitor Object` 模式的封装有关，Java 将这种模式内置到语言层面，即每个 `Object` 都是一个 `Monitor Object`，都内置一个监视器锁。  

## monitor record

`monitor record` 这个概念来源于 David Dice 的一篇文章，该文章在 2001 年发表，由于 `JDK 1.6` 发行于 2006 年，因此笔者推测 `monitor record` 是后来的锁记录，即 `lock record` 的原型。  

`monitor record` 是线程私有的数据结构，每一个线程都有一个可用的 `monitor record` 列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个 `monitor record` 关联（对象头中的 `MarkWord` 中的 `LockWord` 指向 `monitor record` 的起始地址），同时 `monitor record` 中有一个 Owner 字段存放拥有该锁的线程的唯一标识，标识该锁被这个线程占用。  

![monitor record](https://img.nekolr.com/images/2018/06/19/q6b.png)

- Owner  
初始时为 NULL，标识当前没有任何线程拥有该 `monitor record`，当线程成功拥有该锁后，保存线程唯一标识，所释放时重新设置为 NULL。  
- EntryQ  
关联一个系统互斥锁（semaphore），阻塞所有竞争该锁失败的线程。  
- RcThis  
表示 blocked 或 waiting 在该 `monitor record` 上的所有线程的个数。  
- Nest  
用来实现重入锁的计数。  
- hashCode  
保存从对象头拷贝过来的 hashCode 值（可能还包含 GC 分代年龄）。  
- Candidate  
用来避免不必要的阻塞或等待线程唤醒。有两种可能的值：0 表示没有需要唤醒的线程，1 表示需要唤醒一个继任线程来竞争锁。因为每一次只有一个线程能够成功拥有该锁，如果每次前一个释放锁的线程都唤醒所有正在阻塞或等待的线程，这会引起不必要的上下文切换（从阻塞或等待到就绪，然后因为竞争锁失败又被阻塞），从而导致性能下降。  

## Java Monitor 的工作机理

![工作机理 ](https://img.nekolr.com/images/2018/06/19/v6n.jpg)

线程如果获得监视锁成功，将成为该监视者对象的拥有者。在任一时刻内，监视者对象只属于一个活动线程（Owner）。拥有者线程可以调用 `wait()` 方法自动释放监视锁，进入等待状态。  

# 锁优化

在 JDK 1.6 版本中，HotSpot 虚拟机开发团队花费了大量的精力进行了各种锁优化，使得 `synchronized` 的性能与 `ReentrantLock` 已经基本持平。  

## 自旋锁

互斥同步对性能影响最大的是阻塞，挂起线程和恢复线程都需要内核参与，这些操作给系统的并发性能带来很大的压力。同时，虚拟机的开发团队注意到很多应用中，共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和恢复线程不值得，因此一种方式就是让请求锁的线程在失败后并不会挂起，而是去执行忙循环（自旋），直到获取到锁。  

自旋锁在 `JDK 1.6` 中默认开启。自旋锁并不能代替阻塞，自旋等待虽然避免了线程切换的开销，但是它忙等是要占用处理器时间的，如果锁被占用的时间很短，那么自旋等待的效果会很好；如果锁被占用的时间很长，那么自旋只会白白浪费处理器资源，而不会做任何有用的工作。因此自旋必须有一定的限度，自旋超过了限定的次数仍然没有成功获得锁，就应当使用传统的方式去挂起线程。  

## 自适应自旋锁

在 `JDK 1.6` 引入了自适应的自旋锁，自适应意味着自旋的时间不再固定，而是根据上一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。  

如果在同一个锁对象上，自旋等待刚刚成功获取过锁，并且持有锁的线程正在运行中，那么虚拟机就会认为这次自旋也很有可能再次成功，进而它将允许自旋等待更长的时间，比如 100 个循环。如果对于一个锁，自旋很少成功，那么后续获取这个锁时可能会省略掉自旋过程。  
## 锁消除

锁消除是指虚拟机的即时编译器在运行时，对于一些代码上要求同步，但是检测到不可能存在共享数据竞争，这样的同步锁会被消除。锁消除的主要判断依据来源于逃逸分析的数据支持，如果判断一段代码中，堆上的所有数据都不会逃逸出去从而被其他线程访问到，那么就可以把它们当做栈上数据对待，认为它们是线程私有的。  

```java
public void append(String s1, String s2) {
    StringBuffer stringBuffer = new StringBuffer();
    stringBuffer.append(s1).append(s2);
}
```

如上代码，`StringBuffer` 在方法内部，不存在共享数据争用，因此运行时编译器会认为 `append()` 方法的加锁是无意义的，会在编译时删除相关加锁操作。当然，在明知不会存在数据争用的情况下，好的处理方式是避免在代码中使用加锁的处理。  

## 锁粗化

如果代码中有一系列的操作都对同一个对象反复加锁和解锁，甚至加锁是出现在循环体中的，那么即使没有线程竞争，频繁地进行互斥同步操作也会带来不必要的性能损耗。如果虚拟机检测到有这样的操作，会将加锁同步的范围扩展（粗化）到整个操作序列的外部，这样只需要加锁一次就可以了。  

## 轻量级锁

在 `JDK 1.6` 引入，轻量级是相对于使用操作系统互斥量来实现的传统锁（重量级锁）而言，轻量级锁并不是用来替代重量级锁的，它的目的是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量带来的性能损耗。  

**加锁过程：**  

在代码进入同步块时，如果当前同步对象没有被锁定（锁标志位为 01），则虚拟机首先在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，存储锁对象当前的 `Mark Word` 的拷贝，这份拷贝称为 `Displaced Mark Word`。  

虚拟机尝试使用 CAS 操作将对象的 `Mark Word` 更新为指向 `Lock Record` 的指针。  
- 如果更新成功，那么当前线程就获得了该对象的锁，并将对象的 `Mark Word` 的锁标志位更新为 00。  
- 如果更新失败，虚拟机首先检查对象的 `Mark Word` 是否指向当前线程的栈帧（锁记录地址），如果指向则说明当前线程已经拥有了该锁，将锁计数器加一，直接进入同步块继续执行；否则说明这个锁对象已经被其他线程抢占了，当前线程便尝试自旋来获取锁，如果自旋超时，则轻量级锁会膨胀为重量级锁。  

**解锁过程：**  

虚拟机尝试使用 CAS 操作将 `Displaced Mark Word` 替换回对象的 `Mark Word`。  
- 如果替换成功，则整个同步过程完成。  
- 如果替换失败，说明有其他线程尝试获取该锁，锁会膨胀为重量级锁，需要在释放锁的同时唤醒被阻塞的线程。  

![轻量级锁过程 ](https://img.nekolr.com/images/2018/06/20/DAQ.png)

轻量级锁提升性能的依据是：对于绝大部分的锁，在整个同步周期内都是不存在竞争的。如果没有竞争，轻量级锁使用 CAS 操作避免了使用互斥量带来的开销。但是如果存在竞争，除了互斥量的开销，还额外发生了 CAS 操作，这时轻量级锁会比传统的重量级锁更慢。  

## 偏向锁

偏向锁在 `JDK 1.6` 引入，如果说轻量级锁是在无竞争情况下使用 CAS 操作来消除同步使用的互斥量，那么偏向锁就是在无竞争情况下把整个同步都消除，连 CAS 操作都取消了（除了第一次操作）。  

**偏向锁初始化：**  

当锁对象第一次被线程获取时，虚拟机会将对象头中的锁标志位设置为 01，并使用 CAS 操作将偏向的线程 ID 存储到对象头和栈帧中的锁记录中，后续该线程进入和退出临界区都不需要使用 CAS 操作来加锁和解锁，而是先简单检查对象头的 `Mark Word` 中是否存储偏向的线程 ID。  
- 如果存储着，则表示线程已经获得了锁。  
- 如果没有存储，则需要检查对象头中的锁标志位是否为 01，即偏向锁标志，如果没有设置，说明此时使用的一定是更重的锁，因此使用 CAS 操作来竞争锁。如果设置了，则尝试使用 CAS 操作将偏向线程 ID 放入对象头。  

如果 CAS 操作失败，说明当前存在多个线程竞争锁，当达到全局安全点（safepoint，这个时间点上没有正在执行的字节码），线程被挂起，撤销偏向锁并升级为轻量级锁，升级完成后阻塞在安全点的线程继续执行同步代码块。  

**偏向锁撤销：**  

偏向锁只有在其他线程竞争锁时才会撤销，撤销需要等待全局安全点（这个时间点上没有正在执行的字节码）。首先暂停拥有偏向锁的线程并检查线程是否存活。  
- 如果线程仍然存活，则遍历偏向对象的锁记录，锁记录和对象头的 `Mark Word` 要么重新偏向于其他线程，要么恢复到无锁或对象不适合偏向锁，最后唤醒暂停的线程。  
- 如果线程不处于活动状态，则将对象头设置为无锁状态。  

![偏向锁过程 ](https://img.nekolr.com/images/2018/06/20/MR4.png)

如果确定锁通常存在竞争，可以使用 JVM 参数 `-XX:-UseBiasedLocking=false` 关闭偏向锁，默认会进入轻量级锁。  

# 参考

> [探索 Java 同步机制: Monitor Object 并发模式在 Java 同步机制中的实现 ](https://www.ibm.com/developerworks/cn/java/j-lo-synchronized/)

> [关于 synchronized 的 Monitor Object 机制的研究 ](https://blog.csdn.net/m_xiaoer/article/details/73274642)

> 《深入理解 Java 虚拟机:JVM 高级特性与最佳实践》  

> [David Dice: Implementing Fast Java Monitors with Relaxed-Locks](https://www.usenix.org/legacy/event/jvm01/full_papers/dice/dice.pdf)

> [JVM 内部细节之一：synchronized 关键字及实现细节（轻量级锁 Lightweight Locking）](http://www.cnblogs.com/javaminer/p/3889023.html)