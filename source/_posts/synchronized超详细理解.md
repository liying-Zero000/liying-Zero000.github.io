---
title: synchronized超详细理解
date: 2020-08-09 19:13:27
tags:
	- Java
---

synchronized是一个重量级锁？

同步就要用synchronized？

synchronized与Lock的区别？

synchronized跟Lock都是Java中对Mesa管程模型的一种支持，不同点在于，synchronized的条件变量是隐含的，而且只有一个，Lock对象可以new很多个condition出来。

<!--more-->

# 答题要点

## 锁的状态

锁的状态总共有四种：无锁状态、偏向锁、轻量级锁和重量级锁。随着锁的竞争，锁可以从偏向锁升级到轻量级锁，再升级的重量级锁（但是锁的升级是单向的，也就是说只能从低到高升级，不会出现锁的降级）。JDK 1.6 中默认是开启偏向锁和轻量级锁的，我们也可以通过 - XX:-UseBiasedLocking 来禁用偏向锁。

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596685984418-8846cd80-536f-4f74-8b57-5da81d2b5017.png)



## 状态转换

![image](https://gitee.com/liying000/blogimg/raw/master/1596628111797-cec635ac-1ce6-4314-8d78-d15dc5f169e1.jpeg)





**JVM 一般是这样使用锁和 Mark Word 的：**

1，当一个对象**没有被当成锁**时，这就是一个普通的对象，Mark Word 记录对象的 HashCode，锁标志位是 01，是否偏向锁那一位是 0。

2，当对象被当做**同步锁**，并有一个线程 A 抢到了锁时，锁标志位还是 01，但是否偏向锁那一位改成 1，前 23bit 记录抢到锁的线程 id，表示进入偏向锁状态。

3，当线程 A 再次试图来获得锁时，JVM 发现同步锁对象的标志位是 01，是否偏向锁是 1，也就是偏向状态，Mark Word 中记录的线程 id 就是线程 A 自己的 id，表示线程 A 已经获得了这个偏向锁，可以执行同步锁的代码。

4，当线程 B 试图获得这个锁时，JVM 发现同步锁处于偏向状态，但是 Mark Word 中的线程 id 记录的不是 B，那么线程 B 会先用 CAS 操作试图获得锁，这里的获得锁操作是有可能成功的，因为线程 A 一般不会自动释放偏向锁。如果抢锁成功，就把 Mark Word 里的线程 id 改为线程 B 的 id，代表线程 B 获得了这个偏向锁，可以执行同步锁代码。如果抢锁失败，则继续执行步骤 5。

5，偏向锁状态抢锁失败，代表当前锁有一定的竞争，偏向锁将升级为轻量级锁。JVM 会在当前线程的线程栈中开辟一块单独的空间，里面保存指向对象锁 Mark Word 的指针，同时在对象锁 Mark Word 中保存指向这片空间的指针。上述两个保存操作都是 CAS 操作，如果保存成功，代表线程抢到了同步锁，就把 Mark Word 中的锁标志位改成 00，可以执行同步锁代码。如果保存失败，表示抢锁失败，竞争太激烈，继续执行步骤 6。

6，轻量级锁抢锁失败，JVM 会使用自旋锁，自旋锁不是一个锁状态，只是代表不断的重试，尝试抢锁。从 JDK1.7 开始，自旋锁默认启用，自旋次数是自适应的。如果抢锁成功则执行同步锁代码，如果失败则继续执行步骤 7。

7，自旋锁重试之后如果抢锁依然失败，同步锁会升级至重量级锁，锁标志位改为 10。**在这个状态下，未抢到锁的线程都会被阻塞。**





# synchronized整个过程



代码：

```java
class NewBuffer{
    Object lock = new Object();
    public void test(){
        synchronized (lock){
            System.out.println("test");
        }
    }
}
```



javap -c NewBuffer(已编译为class文件)



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596858063254-ffa5019d-3efc-4b9e-a3da-1db74504af9d.png)

代码

```java
public class TestSync {
    public synchronized void doSome(){
        System.out.println("test synchronized");
    }
}
```

javap -c TestSync

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596858458995-cf128107-132c-4fbb-a155-23f108cea852.png)



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596858756781-1369dc53-d8a0-4d9b-a23c-4224b3e6d10d.png)



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596858815467-a023cf69-74bc-448e-90b2-fe485549e55e.png)





![image.png](https://gitee.com/liying000/blogimg/raw/master/1596858875733-167385d5-5090-4e5c-aafd-7dce1dd1e2f3.png)



## monitorenter指令

Each object is associated with a monitor. A monitor is locked if and only if has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:

- if the entry count of the monitor associated with the objectref is zero, the thread enters the monitor and sets its entry count count to one. The thread is then the owner od the monitor
- if the threead already owns the monitor associated with the objectref, it reenters the monitor, incementing its entry count.(也就是属于刚从wait set唤醒，进入到entry set的线程)
- if another thread is already owns the monitor associated with the objectref, the Thread blocks until the monitor's entryset is zero, then tries again to gain ownership.



object是如何跟monitor联系起来的?



object的初始化是先注册本地方法



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596877356860-640ff3de-077b-47c4-8b69-74321a8f1ae7.png)



我！！！知道了！！！！

虽然每个类都是Object的子类，但是monitor绑定到对象上！！！不是Object干的！！！！！



## Java对象的创建过程



### 创建

在语言层面上，创建对象通常仅仅是一个 new 关键字而已。

Java虚拟机遇到一个字节码 new 指令（class字节码）

首先去检查这个指令的参数能否在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。

如果没有，先执行相应的类加载过程。



在类加载检查通过后，虚拟机为新生对象分配内存。对象所需内存的大小在类加载完成后便可完全确定（堆要分配多大的位置给这个对象？）



分配内存后设置对象头。



对象内存布局：

对象头

- 用于存储对象自身的运行时数据：MarkWord

- - 哈希码
  - GC分代年龄
  - 锁状态标志
  - 线程持有的锁
  - 偏向线程ID
  - 偏向时间戳

- 类型指针

Java虚拟机通过这个指针来确定该对象是哪个类的信息。

实例数据

在程序代码里所定义的各种类型的字段内容，无论是从父类继承下来的，还是在子类中定义的字段都必须记录下来。

对齐填充



然后是构造函数，也就是INIT函数



在HotSpot中，oopDesc类定义在[oop.hpp](https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3afe944ca74c3093e7448d6b889cd20d1/src/share/vm/oops/oop.hpp)中，instanceOopDesc定义在[instanceOop.hpp](https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3afe944ca74c3093e7448d6b889cd20d1/src/share/vm/oops/instanceOop.hpp)中（普通对象），arrayOopDesc定义在[arrayOop.hpp](https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3afe944ca74c3093e7448d6b889cd20d1/src/share/vm/oops/arrayOop.hpp)中（数组对象）。(hpp是C++头文件格式)



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596618978017-07123437-e09e-483d-9f62-f1f7bdfb2116.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1596619680426-4d3fea01-3e10-418a-84f7-09da8816f700.png)



**Mark Word**：`instanceOopDesc` 中的 `_mark` 成员，允许压缩。它用于存储对象的运行时记录信息，如哈希值、GC 分代年龄(Age)、锁状态标志（偏向锁、轻量级锁、重量级锁）、线程持有的锁、偏向线程 ID、偏向时间戳等。

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596619951656-c7d64abf-8357-4c51-a62a-60b24255fde2.png)



## 继续monitorenter学习

> 讲解来自于《深入理解Java虚拟机》



monitorenter和monitor这两个字节码指令都需要一个**reference类型的参数**来指明要**锁定和解锁的对象。**



如果synchronized指明了对象参数，那么就用这个对象的引用作为reference。

```java
class Test{

Object lock = new Object();
public void test(){
    synchronized(lock){
        XXX;
    }
}

//或者
public void test2(){
    synchronized(this){
    
    }
} 
}
```

如果没有明确制定，那就根据synchronized修饰的方法类型（如实例方法或者类方法）。来决定是取代码所在的对象实例还是取类型对象的Class对象作为线程要持有的锁。



```java
class Test{
    public synchronized void test(){
        XXX;
    }

//或者
    public synchronized static void test2(){
    XXX;
    } 
}
```

synchronized其实有三种

- 一个新建的对象，定义一个lock
- 当前线程对象，用this或者 synchronized实例方法
- 当前类对象，synchronized了static方法

前两种都是锁住一个普通对象。



enter命令之后，当前线程进入monitor对象的entryset，如果没有竞争，直接进入monitor，然后执行，如果被wait，那么进入waitset，被notify之后再次进入entryset。



## 自旋锁与自适应自旋



互斥同步对性能的最大影响是阻塞的实现，挂起线程和恢复线程的操作都需要转入内核态中完成。



现象：共享数据的锁定状态只会持续很短的一段时间。



所以可以让后来的请求锁的线程“稍等一会”,但是不放弃处理器的执行时间。

自旋的默认值是十次。



无论是默认值还是用户指定自旋，对所有锁来说都是相同的。



JDK6中加入了自旋锁的优化，引入了自适应的自旋。自适应意味着自旋的时间不再是固定的，而是由前一次在同一个锁上的自旋时间以及锁的拥有者的状态来决定的。

在同一个锁对象上

- 自旋等待刚成功获取过锁，并且持有锁的进程正在运行，虚拟机就会认为这次自旋也很有可能再次成功，进而允许自旋等待持续相对长的时间
- 如果对于某个锁，自旋很少成功获得过锁，那在以后要获得这个锁将有可能直接忽略自旋过程



## 轻量级锁

JDK6

传统的使用操作系统互斥量来实现的锁：重量级



轻量级锁并不是用来代替重量级锁的。



设计初衷：在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。



由于对象头是与对象本身定义的数据结构无关的额外存储成本，考虑从到Java虚拟机的空间使用效率，Mark Word被设计成一个非固定的动态数据结构。



根据对象的状态复用自己的存储空间。





![image.png](https://gitee.com/liying000/blogimg/raw/master/1596889477651-b2a55d42-2c9b-425b-a473-93c64a99ccb0.png)

> 也就是说，不同的状态下，mark word 的长度是不同的 ?



### 轻量级锁工作过程：

在代码即将进入同步块的时候，如果此同步代码块没有被锁定(锁标志位为："01")，JVM首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于**存储锁对象目前的Mark Word 的拷贝**（官方命名为 Displaced Mark Word）

![image](https://gitee.com/liying000/blogimg/raw/master/1596890575755-fa6c5d24-9688-4c1b-8447-9ad8e3af0047.png)



```c++
public:
  // Constants
  enum { age_bits                 = 4,
         lock_bits                = 2,
         biased_lock_bits         = 1,
         max_hash_bits            = BitsPerWord - age_bits - lock_bits - biased_lock_bits,
         hash_bits                = max_hash_bits > 31 ? 31 : max_hash_bits,
         cms_bits                 = LP64_ONLY(1) NOT_LP64(0),
         epoch_bits               = 2
  };

  // The biased locking code currently requires that the age bits be
  // contiguous to the lock bits.
  enum { lock_shift               = 0,
         biased_lock_shift        = lock_bits,
         age_shift                = lock_bits + biased_lock_bits,
         cms_shift                = age_shift + age_bits,
         hash_shift               = cms_shift + cms_bits,
         epoch_shift              = hash_shift
  };

  enum { lock_mask                = right_n_bits(lock_bits),
         lock_mask_in_place       = lock_mask << lock_shift,
         biased_lock_mask         = right_n_bits(lock_bits + biased_lock_bits),
         biased_lock_mask_in_place= biased_lock_mask << lock_shift,
         biased_lock_bit_in_place = 1 << biased_lock_shift,
         age_mask                 = right_n_bits(age_bits),
         age_mask_in_place        = age_mask << age_shift,
         epoch_mask               = right_n_bits(epoch_bits),
         epoch_mask_in_place      = epoch_mask << epoch_shift,
         cms_mask                 = right_n_bits(cms_bits),
         cms_mask_in_place        = cms_mask << cms_shift
#ifndef _WIN64
         ,hash_mask               = right_n_bits(hash_bits),
         hash_mask_in_place       = (address_word)hash_mask << hash_shift
#endif
  };
```

##  

然后，JVM将使用**CAS**操作尝试把**对象的 Mark Word 更新为指向Lock Record的指针**。

> 当前无法获取对象的hashcode吗？



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596942884890-60b570c6-dde3-40f2-8ec9-fca056c931cb.png)

抢锁的过程是不是抢这个指针的过程？

如果更新成功，该线程拥有了这个对象的锁，并且把对象Mark Word的锁标志位，转变为"00".



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596942067624-becb6788-d656-4245-b593-a4486058e739.png)

如果更新失败了，那就存在一条线程与当前线程竞争。



JVM首先检查对象的Mark Word是否指向当前线程的栈帧，如果是，说明当前线程已经拥有了对这个对象的锁，直接进入同步块执行。(也就是同一个线程，不同的方法，想要同一个锁？)



如果不是当前线程，那么轻量级锁膨胀为重量级锁，此时Mark word中存储指向重量级锁（**互斥量**）（信号量吗？）的指针，后边等待的线程进入阻塞状态。



## 偏向锁

JDK6引入的优化。



偏向锁的意思是这个锁会偏向于第一个获得它的线程。如果在接下来的过程中，该锁一直没有被其他线程获取，则持有偏向锁的线程永远不需要再进行同步（也就是不需要撤销锁？）

##  

偏向锁获取：对象的处于无锁状态，那就CAS尝试获取锁，获取成功，markword中保存线程id，偏向位置1，锁状态为01。



1. 由来

HotSpot的作者发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次取得，为了让线程获得锁的代价更低而引入了偏向锁。



1. 偏向锁的撤销

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。



## Hashcode的问题

> 好的文章，一定会解答前面出现的疑问

我也想知道hashcode怎么存的。

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596854823873-f2a14c72-0e08-4b71-830e-ce5af6a7271f.png)



> 在Java语言中，一个对象如果计算过hash码，就应该一直保持该值不变（强烈推荐但不强制，因为用户可以重载hashcode方法按自己的意愿返回哈希码），否则很多依赖对象哈希码的API都可能存在出错风险。而作为绝大多数对象哈希码来源的Object::hashcode()方法，返回的是对象的一致性哈希码（Identity Hash code），这个值是能强制保证不变的，它通过在对象头中存储计算结果来保证第一次计算之后，再次调用该方法取到的哈希码值永远不会再发生改变。
>
> 因此，当一个对象已经计算过一致性哈希码之后，它就再也无法进入偏向锁的状态了，而当一个对象当前正处于偏向锁状态，又收到需要计算其一致性哈希码（Object::hashcode或者System::identityHashcode(Object)）时，它的偏向锁状态会被立即撤销，并且锁会膨胀为重量级锁，在重量级锁的实现中，对象头指向了重量级锁的位置，代表重量级锁的ObjectMonitor类里面有字段可以记录非加锁状态(标志位为“01”)下的mark word，其中自然可以存储原来的哈希码。
>
> ——《深入理解Java虚拟机》







# 细节问题

## CAS操作

[Java基础：CAS操作](https://www.jianshu.com/p/bab16d7c83a2)



在Java并发应用中通常指CompareAndSwap，即比较并交换。

在Java中，所有的CAS操作都是Unsafe类实现的。



CAS可以理解为一个无阻塞多线程争抢资源的模型

1. CAS是一个原子操作，它比较一个内存位置的值并且只有相等时修改这个内存位置的值为新的值，保证了新的值总是基于最新的信息计算的，如果有其他线程在这期间修改了这个值则CAS失败。CAS返回是否成功或者内存位置原来的值用于判断CAS是否成功
2. JVM中的CAS操作是利用了处理器提供的CMPXCHG指令实现的。



优点：

- 竞争不大的时候系统开销小

缺点：

- 循环时间长开销大
- ABA问题（无法交替进入？）
- 只能保证一个共享变量的原子操作



底层原理比较复杂，新开一篇。



## wait使用的标准范式。

if和while的区别



notify 唤醒的那一刻，线程**【曾经/曾经/曾经】**要求的条件得到了满足，从这一刻开始，到去条件等队列中唤醒线程，再到再次尝试获取锁是`有时间差`的，当再次获取到锁时，线程曾经要求的条件是**不一定**满足，所以需要**重新**进行条件判断，所以需要将 `if` 判断改成 `while` 判断



> 我：没说到点子上，线程被notify，然后取锁池队列，再获取锁。时间差在这里。



> 一个线程可以从挂起状态变为可运行状态（也就是被唤醒），即使线程没有被其他线程调用notify/notify()方法进行通知，或被中断，或者等待超时，这就是所谓的虚假唤醒。虽然虚假唤醒很少发生，但要防患于未然，做法就是不停地去测试该线程的唤醒条件是否满足。
>
> -《Java并发编程之美》

为什么while可以不断测试？

因为wait在while循环内部，被notify之后会再次进入循环条件进行判断。





## notify和notifAll

`notify `函数

随机唤醒一个在WaitSet中的线程

`notifyAll`函数

唤醒所有的WaitSet中的线程，全部放入锁池

## 什么时候使用notify

1. 所有等待线程拥有相同的等待条件
2. 所有等待线程被唤醒后，执行相同的操作。
3. 只需要唤醒一个线程。



为什么？

线程池使用了`notify`。。。（存在必合理）