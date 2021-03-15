# <h1 align="center">Java多线程源码阅读及思考</h1>

 <p align="center">
 <img src="https://img.shields.io/badge/java-source-red"/>
 <img src="https://img.shields.io/badge/java-concurrent-green"/>
 <img src="https://img.shields.io/badge/Java-Self--study-blue"/>
</p>
 

<h2 align="center">前言</h2>
<p>本文主要用于对Java并发知识的学习整理，以便于之后的复习，含有个人理解，如有错误，希望您能给予指正，以便于修改。</p>
本文主要基于ChiuCheng 个人博客 (https://segmentfault.com/a/1190000016058789) 学习，理解之后又自行整理。

<h2>目录</h2>

<a href="#0">独占锁</a>  

<a href="#0">Reentrantlock</a>  

<a href="#1">1. 核心变量</a>  

<a href="#2">2. 构造函数</a>  

<a href="#3">3. 等待队列</a>  

<a href="#4">4. 锁的获取</a>  

&nbsp;&nbsp;&nbsp;<a href="#3.1">4.1 tryAcquire(arg)函数</a>  
<a id="1"/>

&nbsp;&nbsp;&nbsp;<a href="#3.2">4.2 addWaiter(Node mode)函数</a>  

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<a href="#3.2.1">4.2.1尾分叉</a>  

&nbsp;&nbsp;&nbsp;<a href="#3.3">4.3  acquireQueued(final Node node, int arg)函数</a>  
<a id="1"/>

&nbsp;&nbsp;&nbsp;<a href="#4.4">4.4  selfInterrupt()函数</a>  
<a id="0"/>

<a href="#5">5. 锁的释放</a>  

&nbsp;&nbsp;&nbsp; <a href="#5.1">5.1 tryRelease(arg)函数</a>

&nbsp;&nbsp;&nbsp; <a href="#5.2">5.2 unparkSuccessor(Node node)函数</a>

<a href="#100">参考</a>

<a id="2"/>

<h2>独占锁</h3>

<h2>Reentrantlock</h2>

<h3>1 核心变量</h3>
private final Sync sync;  

ReentrantLock只有个sync属性，这个属性提供了所有的实现，ReentrantLock所有的Lock方法的实现都调用了sync的方法，sync继承自AQS.

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


<h3>3 等待队列</h3>
在AQS中，等待队列的实现是一个双向链表，被称为sync queue，它表示所有等待锁的线程的集合，类似于synchronized里的wait set。

在并发编程队列一般用于**将竞争锁失败的线程包装成某种类型的数据结构保存到等待队列中**，首先我们来队列中的节点是什么样的结构：
```
static final class Node {
...
...
    // 节点所代表的线程
    volatile Thread thread;
    
    // 双向链表，每个节点需要保存自己的前驱节点和后继节点的引用
    volatile Node prev;
    volatile Node next;

    // 线程所处的等待锁的状态，初始化时，该值为0
    volatile int waitStatus;
    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;

    // 该属性用于条件队列或者共享锁
    Node nextWaiter;
...
...
}
```
在Node类中有一个状态变量waitStatus，它表示了当前Node所代表的线程的等待锁的状态，在独占锁模式下只需要关注CANCELLED SIGNAL两种状态。这里还有一个nextWaiter属性，它在独占锁模式下永远为null，仅仅起到一个标记作用，没有实际意义。

说完队列中的Node属性，我们接着说回由这些节点构成的等待队列，在AQS中的队列是一个CLH队列，它的head节点永远是一个哑结点（dummy node), 它不储存任何线程信息（某些情况下与持有锁的线程相关），因此head所指向的Node的thread属性永远是null。从头节点往后的所有节点才代表了所有等待锁的线程，因此头节点之后的节点才可以被算作等待队列。
![image](https://user-images.githubusercontent.com/79728538/110680551-5ec95200-819e-11eb-93d6-f8262cb0da6f.png)  

<span>图片来自https://segmentfault.com/a/1190000016058789</span>

再总结下图中Node的属性：
1.thread：储存Node所代表的线程
2.waitStatus：表示节点所处的等待状态，共享锁模式下只需关注三种状态：SIGNAL CANCELLED 初始态(0)
3.prev next：节点的前驱和后继
4.nextWaiter：进作为标记，值永远为null，表示当前处于独占锁模式


<a id="4"/>

<h3>4 锁的获取</h3>

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



<h3>4.1 tryAcquire(arg)函数</h3>

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


<h3>4.2 addWaiter(Node mode)函数</h3>

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

假如队列为空，我们需要先新建一个头节点，然后进入下一轮循环。在下一轮循环中，队列已经不为null了，此时才可以再将我们包装了当前线程的Node加到这个空节点后面。 
**此处要注意，头节点并没有储存任何线程信息，头节点有时与获得锁的线程相关，相关线程已经获得了同步状态，在执行相应的业务逻辑了**


<h3>4.2.1 尾分叉</h3>

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

**在第二步发生时，第三步可能还没发生，但是，CAS已经操作成功（第二步已经发生），竞争已经结束，但此时原来旧的尾节点的next值可能还是null（第三步还没有发生），假如有一个线程从前向后遍历这个队列就会漏掉这个新插入的节点，这明显不合理，但是tail已经成功修改，tail的前置节点也已经修改成功（第一步已经发生）从后向前遍历就可以遍历到所有节点**

例如在后面的unparkSuccessor(Node node)方法中
<a id="3.3"/>
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
最后，addWaiter(Node.EXCLUSIVE)方法最终返回了代表了当前线程的Node节点。

<h3>4.3 acquireQueued(final Node node, int arg)函数</h3>

acquireQueued的基本逻辑是；
(1) addWaiter 方法已经成功将包装了当前Thread的节点添加到了等待队列
(2) 该方法中将再次尝试去获取锁
(3) 在再次尝试获取锁失败后, 判断是否需要把当前线程挂起

**为什么前面已经获取锁失败了此处还要再一次获取锁呢?** 首先我们需要理解这里获取锁是基于一定条件的：当前节点的前驱节点就是HEAD节点，我们之前说过，head节点不储存任何任何线程信息，或者他代表了持有锁的线程，如果当前节点的前驱节点是head说明当前节点已经是等待队列里的第一个节点。

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 在当前节点的前驱就是HEAD节点时, 再次尝试获取锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //在获取锁失败后, 判断是否需要把当前线程挂起
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
假如前驱节点是头节点，而且获取锁成功了，就会调用setHead()函数：
```
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```
**从这段代码，我们可以很清楚的看到，一个节点在成为头节点之后，就会清除它储存的线程信息。这相当于一种变相的出队操作，其实等待队列指的是该队列里除head以外的节点，这里的setHead也不需要进行CAS操作，因为tryAcquire(arg)获取锁成功之后才会执行if里面的代码，只有一个线程可以获取锁成功**

若获取锁失败就会执行shouldParkAfterFailedAcquire()函数，根据前驱节点的**waitStatus**值，判断是否应该将线程挂起。当一个线程挂起之前，他必须要确保自己的前驱节点的waitStatus为SIGNAL。
之前我们提到了在独占锁中，只涉及到了CANCELLED和SIGNAL状态。
1.CANCELLED状态表示Node所代表的当前线程已经取消了排队。
2.SIGNAL状态不代表当前节点的状态，而是当前节点的下一个节点的状态，他是被后继节点设置的。当一个节点的waitStatus被置为SIGNAL，就说明它的后继节点已经被挂起了，因此在当前节点释放了锁或者放弃获取锁时，如果它的waitStatus属性为SIGNAL，它需要唤醒它的后继节点。  

理解了CANCELLED和SIGNAL状态之后，我们来看shouldParkAfterFailedAcquire(）函数：
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; // 前驱节点的ws
    if (ws == Node.SIGNAL)
        // 如果前驱节点的状态是SIGNAL则直接挂起
        return true;
    if (ws > 0) {
        // 当前节点的 ws > 0, 则为 Node.CANCELLED,说明前驱节点放弃等待锁，
        // 此时我们需要向前遍历找到一个还在等待锁的节点（ws <= 0）并且将本节点挂在这个节点后面。
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 如果前驱节点的状态不是SIGNAL也不是CANCELLED，用CAS设置前驱节点的ws为 Node.SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
假如上述操作都没有成功，执行到返回false，会继续回到循环中再次尝试获取锁——这是因为此时我们的前驱节点可能已经变了。假如执行成功返回true则会调用parkAndCheckInterrupt,将线程挂起, 等待被唤醒.

<a id="4.4"/>
<h3>4.4 selfInterrupt()函数</h3>

<a id="5"/>

该方法由AQS实现, 用于中断当前线程。在整个抢锁过程中，都是不响应中断的。假如在抢锁的过程中发生了中断, AQS会记录有没有有发生过中断，如果返回的时候发现在抢锁过程中曾经发生过中断，则在退出acquire方法之前，调用selfInterrupt自我中断一下，将这个发生在抢锁过程中的中断“推迟”到抢锁结束以后再发生。

<h3>5. 锁的释放</h3>

JAVA的内置锁在退出临界区之后是会自动释放锁的, 但是显式锁是需要自己显式的释放的, 所以在加锁之后一定不要忘记在finally块中进行显式的锁释放:

```
Lock lock = new ReentrantLock();
...
lock.lock();
try {
    // 更新对象
    //捕获异常
} finally {
    lock.unlock();
}

```
**此处一定要注意lock.unlock()一定要在finally里面，而lock.lock()要在try之外，因为假如lock.lock()在try之内，在获取锁的过程中，如果抛出异常，没有获取锁成功，还是会调用释放锁。**

下面我们来阅读独占锁，ReentrantLock的源码，**因为锁的释放操作，对于公平锁和非公平锁是一样的，所以释放锁的逻辑没有放在FairSync 或 NonfairSync 里面， 而是直接定义在 sync类中。**

```
public void unlock() {
    sync.release(1);
}
```
<a id="5.1"/>
```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
可以看到在这个方法中，主要涉及到两个子函数的调用tryRelease(arg)，和unparkSuccessor(h)函数，下面我们逐行分析这两个函数的源码。


<h3>5.1 tryRelease(arg)函数</h3>

```
protected final boolean tryRelease(int releases) {
    
    // 首先将当前持有锁的线程个数减1(回溯到调用源头sync.release(1)可知, releases的值为1)
    // 这里的操作主要是针对可重入锁的情况下, c可能大于1
    int c = getState() - releases; 
    
    // 释放锁的线程当前必须是持有锁的线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    
    // 如果c为0了, 说明锁已经完全释放了
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

<a id="5.2"/>
因为只有获取了锁的线程才会执行释放锁的逻辑，而且此处的代码还是独占锁，所以并不需要CAS操作。

<h3>5.2 unparkSuccessor(Node node)函数</h3>

unparkSuccessor(h)用于唤醒后继线程，此处我们可以看到对head节点的waitStatus的判断，之前我们在获取锁失败进入队列失败时讲过，**如果一个线程被挂起了, 它的前驱节点的 waitStatus值必然是Node.SIGNAL.**

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
**此处为什么要判断h.waitStatus != 0呢**

等待队列节点的waitstatus初始值为0，但是从shouldParkAfterFailedAcquire 函数中可以了解到，只要有一个新的节点入队，就会将前驱节点的waitStatus设为Node.SIGNAL。所以当一个head节点的waitStatus为0, 说明这个head节点后面没有在挂起等待中的后继节点了(如果有的话, head的ws就会被后继节点设为Node.SIGNAL了), 自然也就不要执行 unparkSuccessor 操作了.

接下来就是唤醒后继线程的函数，在释放锁成功之后就需要加入CAS操作了：

```
private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    
    // 如果head节点的ws比0小, 则直接将它设为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    // 通常情况下, 要唤醒的就是自己的后继节点，如果后继节点在等待锁, 那就直接唤醒它
    // 但是后继节点可能已经取消等待锁，此时就需要从尾节点开始向前找起, 直到找到距离head节点最近等待锁的节点(ws<=0)
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t; // 注意! 这里找到了之并有return, 而是继续向前找
    }
    // 如果找到了还在等待锁的节点，则唤醒它
    if (s != null)
        LockSupport.unpark(s.thread);
}
```
**此处我们可以看到从尾节点向前遍历的方式，在**<a href="#3.2.1">尾分叉</a> **里我们已经讲过其中原理了**。
在释放锁之后，一个等待中的线程就会从之前讲过的parkAndCheckInterrupt()函数中的挂起处被唤醒，然后继续执行。
```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 在这里被挂起了, 唤醒之后就能继续往下执行了
    return Thread.interrupted();
}
```

<a id="100"/>
<h2>参考</h2>

本文参考有以下博客，有如有侵权，请联系我删除：

1. ChiuCheng 个人博客：https://segmentfault.com/a/1190000016058789。
2. 美团技术团队 家琪：不可不说的Java“锁”事：https://tech.meituan.com/2018/11/15/java-lock.html





