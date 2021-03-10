# <h1 align="center">Java多线程源码阅读及思考</h1>

 <p align="center">
 <img src="https://img.shields.io/badge/java-source-red"/>
 <img src="https://img.shields.io/badge/java-concurrent-green"/>
 <img src="https://img.shields.io/badge/Java-Self--study-blue"/>
</p>
 

<h2 align="center">前言</h2>
<p>本文主要用于对Java并发知识的学习整理，以便于之后的复习，含有个人理解，如有错误，希望您能给予指正，以便于修改。</p>
本文参考的文章和博客链接有，如有侵权，请联系我删除：

1. ChiuCheng 个人博客：https://segmentfault.com/a/1190000016058789。
2. 美团技术团队 家琪：不可不说的Java“锁”事：https://tech.meituan.com/2018/11/15/java-lock.html


<h2>Reentrantlock</h2>

<h2>目录</h2>

<a href="#1">1. 核心变量</a>  

<a href="#2">2. 构造函数</a>  

<a href="#3">3. 锁的获取</a>  

&nbsp;&nbsp;&nbsp;<a href="#3.1">3.1 tryAcquire(arg)函数</a>  
<a id="1"/>

&nbsp;&nbsp;&nbsp;<a href="#3.2">3.2 addWaiter(Node mode)函数</a>  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<a href="#3.2.1">3.2.1尾分叉</a>  


<a id="2"/>

<h3>1 核心变量</h3>
private final Sync sync;  

ReentrantLock只有个sync属性，这个属性提供了所有的实现，我们上面介绍ReentrantLock对Lock接口的实现的时候就说到，它对所有的Lock方法的实现都调用了sync的方法，sync继承自AQS.

<h3>2 构造函数</h3>

 ```
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
<a id="3"/>

**含参构造函数用于指定是否为公平锁，无参默认为启用非公平锁**。公平锁的优点在于等待锁的线程不会饿死，缺点是整体吞吐率相对非公平锁比较低，等待队列中除了第一个线程都会被阻塞，CPU唤醒阻塞线程的开销比非公平锁大。非公平锁是多个线程加锁时直接尝试获取锁，获取不到才会到等待队列的队尾等待。但如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁可以减少唤起线程的开销，整体的吞吐效率高。


<h3>3 锁的获取</h3>

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
<a id="3.1"/>
该方法主要涉及了四个函数的调用：
（1）tryAcquire(arg)（涉及了获取锁的具体逻辑）
（2）addWaiter(Node mode)（在锁获取失败后，进入等待队列）
（3）acquireQueued(final Node node, int arg)（在进入等待队列后，假如前驱节点是head，就继续尝试获取锁，否则就挂起）
（4）selfInterrupt（假如抢锁过程中发生了中断，不响应，在退出acquire方法之前自我中断一下，即将中断推迟到抢锁结束后）



<h3>3.1 tryAcquire(arg)函数</h3>

tryAcquire(arg)函数的主要逻辑是：
1.如果锁没有被占用, 尝试以公平的方式获取锁
2.如果锁已经被占用, 检查是不是锁重入
3.获取成功则返回true，失败则返回false

```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState(); 
    //在独占锁中，如果c==0 说明当前锁还没有被占用, 可以尝试获取锁 因为是实现公平锁, 所以在抢占之前首先看看队列中有线程自己前面的Node，
    //如果没有人在排队, 则通过CAS方式获取锁
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires))
        // !hasQueuedPredecessors()判断自己之前是否还有节点等待，如果没有cas获取锁
        {
            setExclusiveOwnerThread(current); 
            // 在AQS中，通过exclusiveOwnerThread属性记录了当前拥有锁的线程
            //, setExclusiveOwnerThread(current)将拥有锁的线程设置为当前线程
            return true;
        }
    }
    
    // 如果 c>0 说明锁已经被占用了 如果是可重入锁, 这个时候检查占用锁的线程是不是就是当前线程,是的话,说明已经拿到了锁, 直接重入就行
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        //此处不需要cas获取锁，因为占有锁的正是本线程，所以对state的修改是安全的
        setState(nextc);
        /* setState方法如下：
        protected final void setState(int newState) {
            state = newState;
        }
        */
        return true;
    }
    
    // 获取锁失败
    return false;
}
```
<a id="3.2"/>
其实获取锁中的核心方法就是通过 compareAndSetState(0, acquires) 将锁的state状态改写，因为是cas操作，因此可以保证只有一个线程能成功，成功之后，再将
拥有锁的线程改写为当前线程。对于可重入锁，则对比获取锁的线程是不是当前线程，是就直接获取。


<h3>3.2 addWaiter(Node mode)函数</h3>

执行到此方法, 尝试获取锁失败, 就要将当前线程包装成Node，加到等待锁的队列中去, 因为是FIFO队列, 所以要加在队尾。
```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode); //将当前线程包装成Node
    static final Node EXCLUSIVE = null;
    Node pred = tail;
    // 如果队列不为空, 则用CAS方式将当前节点设为尾节点
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    // 代码会执行到这里, 只有两种情况:
    //    1. 队列为空
    //    2. CAS失败
    // 注意, 这里是并发条件下, 所以什么都有可能发生, 尤其注意CAS失败后也会来到这里
    enq(node); //将节点插入队列
    return node;
}
```
在这个方法中，我们首先会尝试直接入队，但是因为目前是在并发条件下，所以有可能同一时刻，有多个线程都在尝试入队，导致compareAndSetTail(pred, node)操作失败——因为有可能其他线程已经成为了新的尾节点，导致尾节点不再是我们之前看到的那个pred了。如果入队失败了，接下来我们就需要调用enq(node)方法，在该方法中我们将通过自旋+CAS的方式，确保当前节点入队。

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {
            // 如果是空队列, 首先进行初始化
            if (compareAndSetHead(new Node()))
                tail = head; // 这里仅仅是将尾节点指向dummy节点，并没有返回
        } else {
        // 如果队列不空，尝试将节点加到队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```
<a id="3.2.1"/>

假如队列为空，我们需要先新建一个头节点，然后进入下一轮循环。在下一轮循环中，队列已经不为null了，此时才可以再将我们包装了当前线程的Node加到这个空节点后面。**此处要注意，头节点并没有储存线程信息，不代表如何线程**


<h3>3.2.1 尾分叉</h3>

**在将节点储存到队列中时，会有一个很有趣的现象，叫做尾分叉，理解尾分叉是看懂遍历等待队列的关键**
队列不空时，将节点加入队列有三步：
1.设置node的前驱节点为当前的尾节点：node.prev = t. 
2.修改tail属性，使它指向当前节点. 
3.修改原来的尾节点，使它的next指向当前节点. 

```
} else {
// 到这里说明队列已经不是空的了, 这个时候再继续尝试将节点加到队尾
    node.prev = t;
    if (compareAndSetTail(t, node)) {
        t.next = node;
        return t
    }
}
```

![image](https://user-images.githubusercontent.com/79728538/110553394-849f1a00-80fe-11eb-9523-05833ef53b9c.png)

<span>图片来自https://segmentfault.com/a/1190000016058789</span>

这三步不是原子操作，第二步是一个CAS操作，多线程环境下，只有一个线程能够成功修改tail，第二步和第三步是原子的，第二步发生后第三步才会发生.其他所有线程只能发生第一步，因此就会出现尾分叉现象：

![image](https://user-images.githubusercontent.com/79728538/110553778-24f53e80-80ff-11eb-8917-8a66be296736.png)

<span>图片来自https://segmentfault.com/a/1190000016058789</span>

**在第二步发生时，第三步可能还没发生，但是，CAS已经操作成功（第二步发生），竞争已经结束，但此时原来旧的尾节点的next值可能还是null（第三步还没有发生），假如有一个线程从前向后遍历这个队列就会漏掉这个新插入的节点，这明显不合理，但是tail已经成功修改，tail的前置节点也已经修改成功（第一步发生）从后向前遍历就可以遍历到所有节点**

例如在后面的unparkSuccessor(Node node)方法中
```
private void unparkSuccessor(Node node) {
...
...
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
        //从后向前遍历
            if (t.waitStatus <= 0)
                s = t;
    }
...
...
}
```









