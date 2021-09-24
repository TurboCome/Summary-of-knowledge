## AQS 详解

参考：https://mp.weixin.qq.com/s/trsjgUFRrz40Simq2VKxTA ； https://mp.weixin.qq.com/s/iNz6sTen2CSOdLE0j7qu9A



#### 三个线程(线程1，2，3)同时来加锁/释放锁：

#### 目录

1.线程一加锁成功时 AQS内部实现
2.线程二/三加锁失败时 AQS中等待队列的数据模型
3.线程一释放锁，及线程二获取锁实现原理
4.通过线程场景来讲解Condition中await()和signal()实现原理



#### 1.线程一加锁成功：

如果同时有三个线程并发抢占锁，此时线程一抢占锁成功，线程二和线程三抢占锁失败，具体执行流程如下：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursn4dvwij60xm0i8mz302.jpg" alt="1E28D15C-7C36-4A72-9F39-B8EA197A02FB" style="zoom:60%;" />

此时AQS内部数据为：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursnhf7xnj60zc09u0tq02.jpg" alt="933E683E-0920-41D0-A601-25C5C8898215" style="zoom:67%;" />



#### 2.线程二、线程三加锁失败：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursodunv4j615m0dqjtl02.jpg" alt="DA3FB080-3481-408D-B49B-E83AB742E490" style="zoom:65%;" />

有图可以看出，等待队列中的节点Node是一个 双向链表
线程二抢占锁失败,按照真实场景来分析，线程一抢占锁成功后，state变为1，线程二通过 CAS修改state变量必然会失败，此时AQS中FIFO队列中数据如图:

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursosc28ej61460deq4q02.jpg" alt="05B538CD-F5AD-4E3C-B3A8-8000D55BC3E1" style="zoom:67%;" />

线程一将state修改为1，所以线程二通过 CAS修改state的值不会成功，加锁失败。线程二执行 tryAcquire()后会返回false，接着执行addWaiter(Node.EXCLUSIVE)逻辑，将自己加入到一个FIFO等待队列中。

#### 3.线程一释放锁

线程一释放锁，释放锁后会 唤醒head节点的后置节点，就是线程二。线程二设置为head节点，然后空置之前的head节点数据，被空置的节点数据等着被垃圾回收。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursq4l5p7j60yo0jgdi902.jpg" alt="73FBFF18-A055-4AC6-AB79-2AE515C0B494" style="zoom:67%;" />



线程二 释放锁 / 线程三加锁：当线程二释放锁时，会唤醒被挂起的线程三，流程和上面大致相同，被唤醒的线程三会再次尝试加锁

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursr1qi48j60zo0d40ug02.jpg" alt="9E6853B0-BEB4-4D12-B09D-CC43AD0DF73F" style="zoom:67%;" />



#### 4.Contition实现原理：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursr8lel9j618s0maadf02.jpg" alt="0A1A1211-CA16-4C20-B028-C49F8390B9E4" style="zoom:67%;" />

await()方法中首先调用 addConditionWaiter()将当前线程加入到Condition队列中，执行完可以看到 Condition队列中的数据：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursrociquj618s0c2dhh02.jpg" alt="7AEDFCE0-0186-4AC5-B6B6-4C1A38F85CAC" style="zoom:67%;" />

​		这里会用当前线程创建一个 Node节点，waitStatus为CONDITION。 接着会释放该节点的锁，调用之前解析过的release()方法，释放锁之后，会唤醒被挂起的线程二，线程二会继续尝试获取锁。接着调用 isOnSyncQueue()方法判断当前节点是否为 Condition队列中的头部节点，如果是则调用LockSupport.park(this)挂起 Condition中当前线程。此时线程一被挂起，线程二获取锁成功。

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gurss3nurgj617u0m6jum02.jpg" alt="CCD0588F-1A68-4699-B803-35B5797B18A9" style="zoom:67%;" />

​		线程二执行 signal()方法：首先我们考虑下线程二已经获取到锁，此时AQS等待队列中已经没有数据。
这里先从transferForSignal()方法来看，Condition队列中只有线程一创建的一个Node节点，且waitStatue为CONDITION，先通过CAS修改当前节点waitStatus为0，然后执行enq()方法将当前线程加入到等待队列中，并返回当前线程的前置节点。加入等待队列的代码在上面也已经分析过，此时等待队列中数据如下图：

<img src="https://tva1.sinaimg.cn/large/008i3skNly1gursshoev3j61880go76702.jpg" alt="75794B2C-981A-45AB-9F7A-AC4410FE3E79" style="zoom:67%;" />

接着开始通过CAS修改当前节点的前置节点waitStatus为SIGNAL，并且唤醒当前线程。等线程二释放锁后，线程一会继续重试获取锁，结束。

