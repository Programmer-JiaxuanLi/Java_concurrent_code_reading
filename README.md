# Java多线程源码阅读及思考

<h1 align="center">前言</h1>
<p>本文主要用于对Java并发知识的学习整理，以便于之后的复习。本文主要目的是个人笔记，含有个人理解，如有错误，希望您能给予指正，以便于修改。</p>
本文参考的文章和博客链接有，如有侵权，请联系我删除：

1. [ChiuCheng 个人博客：https://segmentfault.com/a/1190000016058789](https://segmentfault.com/a/1190000016058789)。
2. 美团技术团队 家琪：不可不说的Java“锁”事 https://tech.meituan.com/2018/11/15/java-lock.html

 
![https://img.shields.io/badge/java-source-red](https://img.shields.io/badge/java-source-red) ![https://img.shields.io/badge/java-concurrent-green](https://img.shields.io/badge/java-concurrent-green)  ![https://img.shields.io/badge/Java-Self--study-blue](https://img.shields.io/badge/Java-Self--study-blue) 

<h2>Reentrantlock</h2>

<h4>1 接口对比</h4>

<table>
  <thead>
    <tr>
      <th>Lock 接口</th>
      <th>ReentrantLock 实现</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>lock()</td>
      <td>sync.lock()</td>
    </tr>
    <tr>
      <td>lockInterruptibly()</td>
      <td>sync.acquireInterruptibly(1)</td>
    </tr>
    <tr>
      <td>tryLock()</td>
      <td>sync.nonfairTryAcquire(1)</td>
    </tr>
    <tr>
      <td>
        tryLock(long time, TimeUnit unit)
      </td>
      <td>
        sync.tryAcquireNanos(1, unit.toNanos(timeout))
      </td>
    </tr>
    <tr>
      <td>
        unlock()
      </td>
      <td>
        sync.release(1)
      </td>
    </tr>
    <tr>
      <td>
        newCondition()
      </td>
      <td>
        sync.newCondition()
      </td>
    </tr>
  </tbody>
</table>

<h4>2 核心变量</h4>
private final Sync sync;
ReentrantLock只有个sync属性，这个属性提供了所有的实现，我们上面介绍ReentrantLock对Lock接口的实现的时候就说到，它对所有的Lock方法的实现都调用了sync的方法，这个sync就是ReentrantLock的属性，它继承了AQS.

<h4>构造函数</h4>

 ```
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

**含参构造函数用于指定是否为公平锁，无参默认为启用非公平锁**。公平锁的优点在于等待锁的线程不会饿死，缺点是整体吞吐率相对非公平锁比较低，等待队列中除了第一个线程都会被阻塞，CPU唤醒阻塞线程的开销比非公平锁大。非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁可以减少唤起线程的开销，整体的吞吐效率高。

<h4>3 锁子的获取</h4>

我们先以公平锁为例，来分析锁的获取
```
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    //获取锁
    final void lock() {
        acquire(1);
    }
    ...
}
```
公平锁的lock方法调用的acquire来自于父类AQS
```
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
该方法主要涉及了四个函数的调用：
（1）tryAcquire(arg)（涉及了获取锁的具体逻辑）
（2）addWaiter(Node mode)（在锁获取失败后，进入等待队列）
（3）acquireQueued(final Node node, int arg)（在进入等待队列后，假如前驱节点是head，就继续尝试获取锁，否则就挂起）
（4）selfInterrupt（假如抢锁过程中发生了中断，不响应，在退出acquire方法之前自我中断一下，即将中断推迟到抢锁结束后）

<h4>3.1 tryAcquire(arg)函数</h4>

tryAcquire(arg)函数的主要逻辑是：
1.如果锁没有被占用, 尝试以公平的方式获取锁
2.如果锁已经被占用, 检查是不是锁重入
3.获取成功则返回true，失败则返回false

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState(); 
    //在独占锁中，如果c==0 说明当前锁是avaiable的,可以尝试获取锁 因为是实现公平锁, 所以在抢占之前首先看看队列中有线程自己前面的Node，如果没有人在排队, 则通过CAS方式获取锁
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires))
        // !hasQueuedPredecessors()
        {
            setExclusiveOwnerThread(current); // 将当前线程设置为占用锁的线程
            return true;
        }
    }
    
    // 如果 c>0 说明锁已经被占用了 如果是可重入锁, 这个时候检查占用锁的线程是不是就是当前线程,是的话,说明已经拿到了锁, 直接重入就行
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        /* setState方法如下：
        protected final void setState(int newState) {
            state = newState;
        }
        */
        return true;
    }
    
    // 到这里说明有人占用了锁, 并且占用锁的不是当前线程, 则获取锁失败
    return false;
}
```








