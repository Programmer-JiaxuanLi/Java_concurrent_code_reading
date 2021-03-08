# Java多线程源码阅读及思考

<h1 align="center">前言</h1>
<p>本文主要用于对Java并发知识的学习整理，以便于之后的复习。本文主要目的是个人笔记，含有个人理解，如有错误，希望您能给予指正，以便于修改。</p>
本文参考的文章和博客链接有，如有侵权，请联系我删除：

1. [ChiuCheng 个人博客：https://segmentfault.com/a/1190000016058789](https://segmentfault.com/a/1190000016058789)。

https://github.com/<OWNER>/<REPOSITORY>/workflows/<WORKFLOW_NAME>/badge.svg

 
![https://img.shields.io/badge/java-source-red](https://img.shields.io/badge/java-source-red) ![https://img.shields.io/badge/java-concurrent-green](https://img.shields.io/badge/java-concurrent-green)  ![https://img.shields.io/badge/Java-Self--study-blue](https://img.shields.io/badge/Java-Self--study-blue) 

<h2>1.Reentrantlock</h2>
<h4>1.1 接口对比</h4>

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

<h4>核心变量</h4>
private final Sync sync;
ReentrantLock只有个sync属性，这个属性提供了所有的实现，我们上面介绍ReentrantLock对Lock接口的实现的时候就说到，它对所有的Lock方法的实现都调用了sync的方法，这个sync就是ReentrantLock的属性，它继承了AQS.

<h4>构造函数</h4>
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
含参构造函数用于指定是否为公平锁，无参默认为启用非公平锁。
原因是股票所

