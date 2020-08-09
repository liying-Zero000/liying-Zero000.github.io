---
title: CAS理解与初步使用
date: 2020-08-09 19:13:44
tags:
	- Java
---

在synchronized中，锁竞争跟提交markword都用到了CAS，而CAS也是Java中无锁，乐观锁的具体实现

<!--more-->

JDK11 CompareAndSet

JDK8 CompareAndSwap



无锁、自旋锁、乐观锁，都差不多是CAS



不存在死锁



没有阻塞，线程不切换，性能高。

```
cas(V,Expeccted,NewValue)//V 变量，Expected：期望的值，NewValue：拟修改的新值
```

CAS有3个操作数，分别是内存位置V、旧的预期值A和拟修改的新值B



## 简单讲解

if V==E{ //其他线程没有修改

V = NewValue

}else{

try again or fail;

}

## 源码分析

当且仅当V符合预期值A的时候，用新值更新V的值，否则什么都不做。

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596956845743-aa905d9b-5259-4fdb-944c-fed9c7e3a826.png)

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596956908183-7faa872e-df98-4962-a99d-301978734008.png)

都是调用了unsafe类的getAndXXXInt

```
public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

先是var5等于this.getIntVolatile(var1,var2); //变量变成volatile的？获取旧的值

然后 compareAndSwapInt

false，继续do

true，退出





核心代码：

```
while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
```

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596957669540-7f89378b-692b-4cfa-9cf5-2296b41de657.png)

native方法……

1) compareAndSwapInt方法

![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1596958191338-0686a37e-2186-4af6-97fc-c81c3ea511c3.png)

http://www.docjar.com/html/api/sun/misc/Unsafe.java.html



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596958295543-3eb24541-456b-4304-92f8-7f237c722ebb.png)

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596958715406-827675e1-6c67-4852-b3e8-3c2f1babd0b8.png)



```
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  
  //获取对象的变量的地址?
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```



作者：HappyFeet

链接：https://juejin.im/post/6844904036940906510

来源：掘金

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



2）`atomic_****_****.inline.hpp`



- 如果是运行在 windows_x86（windows 系统，x86 处理器）下

```
atomic_windows_x86.inline.hpp`：`openjdk/hotspot/src/os_cpu/windows_x86/vm/atomic_windows_x86.inline.hpp
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
```

[点击查看源码](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os_cpu/windows_x86/vm/atomic_windows_x86.inline.hpp#l216)

- 如果是运行在 linux_x86 （linux 系统，x86 处理器）下

```
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

[点击查看源码](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/os_cpu/linux_x86/vm/atomic_linux_x86.inline.hpp#l93)



`__asm`表示汇编的开始

`volatile` 表示禁止编译器优化

`LOCK_IF_MP` 是个内联函数

```
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
复制代码
```

以上几个参考自占小狼的[深入浅出 CAS](https://www.jianshu.com/p/fb6e91b013cc)。

`os::is_MP()` 这个函数是判断当前系统是否是多核处理器。



就只关注了这一行 `LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"` ，毕竟后面的也看不懂。。。

这里相当于是如果当前系统是多核处理器则会在添加 lock 指令前缀，否则就不加。



关于 `lock` 指令前缀说明：

- 在《深入理解 Java 虚拟机》中，作者解释 volatile 的内存可见性原理时提到（371 页）：“关键在于 lock 前缀，查询 IA32 手册，它的作用是使得本 CPU 的 Cache 写入了内存，该写入动作也会引起别的 CPU 或者别的内核无效化（ Invalidate ）其 Cache，这种操作相当于对 Cache 中的变量做了一次前面介绍 Java 内存模式中所说的 ' store 和 write ' 操作。”
- [intel IA32 手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf) 中 8.1 LOCKED ATOMIC OPERATIONS 关于 lock 前缀的含义：

- - 保证原子操作
  - 总线锁定，通过使用 LOCK＃ 信号和 LOCK 指令前缀
  - 高速缓存一致性协议，确保可以对高速缓存的数据结构执行原子操作（缓存锁定）

- lock 指令前缀也具有禁止指令重排序作用：可以通过阅读 [intel IA32 手册](https://www.intel.com/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-vol-3a-part-1-manual.pdf)中 8.2.2 Memory Ordering in P6 and More Recent Processor Families 和 8.2.3 Examples Illustrating the Memory-Ordering Principles 两节的内容得出。



> 我：两个系统的lock指令都是用来实现缓存一致性。。。。



其中，**`cmpxchg` 是硬件级别的原子操作指令，`lock` 前缀保证这个指令执行结果的内存可见性和禁止指令的重排序。**

> 关于 `lock cmpxchg` 的一些个人理解：由于 `lock` 指令前缀会锁定总线（或者是缓存锁定），所以在该 CPU 执行时总线是处于独占状态，该 CPU 通过总线广播一条 `read invalidate` 信息，通过高速缓存一致性协议（MESI），将其余 CPU 中该数据的 Cache 置为 `invalid` 状态（如果存在该数据的 Cache ），从而获得了对该数据的独占权，之后再执行 `cmpxchg` 原子操作指令修改该数据，完成对数据的修改。



作者：HappyFeet

链接：https://juejin.im/post/6844904036940906510

来源：掘金

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



分割

## ABA问题

关键步骤是比较当前的值是否与共享变量的值是否一样

step 1：共享变量 int 初始值为2 //基础数据类型

step 2：线程A （1）get int = 2 ，(2)2-1 暂停！

step 3: 线程B在此时，把在堆中的共享变量int的值改为1，又改回2

step 4：A往堆里写 2-1的结果，没有问题。



基础数据类型没有问题。



引用类型的问题

step 1：共享变量 为一个对象

step 2：线程A （1）get 对象，(2)修改对象 暂停！

step 3: 线程B在此时，把在堆中的共享修改对象的某些属性

step 4：线程A再写回修改结果。它判断还是相同的对象……



但是其实！！！！对象被修改了。。。。



### 解决方案

给数据加版本号

在新的AtomicStampedReference里面，给compareAndSet加了新的参数 stamp。



## CAS使用

可以用来手写一个锁

```
package interview;

import java.util.Queue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

public class YingLokc implements Lock {
    AtomicReference<Thread> owner = new AtomicReference<>();
    Queue<Thread> waiters = new LinkedBlockingQueue<>();
    @Override
    public void lock() {
        while (!owner.compareAndSet(null,Thread.currentThread())){
            waiters.add(Thread.currentThread());
            //阻塞线程
            LockSupport.park();
            waiters.remove(Thread.currentThread());
        }
    }
    @Override
    public void unlock() {
        if (!owner.compareAndSet(Thread.currentThread(),null)){
            for (Thread thread:waiters){
                LockSupport.unpark(thread);
            }
        }
    }
}
```



LockSupport的park是直接阻塞线程，重量级锁，unpark之后会从断点恢复。



另一个例子

[java基础：java CAS操作](https://www.jianshu.com/p/bab16d7c83a2)

```
package com.company.reentrantLock;

import java.util.concurrent.atomic.AtomicBoolean;
/**
 * Created by wxwall on 2017/6/2.
 */
public class AtomicBooleanTest implements Runnable{
    public static AtomicBoolean exits = new AtomicBoolean(true);
    public static void main(String[] args) {
        AtomicBooleanTest abd = new AtomicBooleanTest();
        Thread t1 = new Thread(abd);
        Thread t2 = new Thread(abd);
        t1.start();
        t2.start();
    }

    @Override
    public void run() {
        System.out.println("begin run");
        System.out.println("real " + exits.get());
        if(exits.compareAndSet(true,false)){
            System.out.println(Thread.currentThread().getName() + "  " + exits.get() );
            exits.set(true);
        }else{
            run();
        }
    }
}
```

多个线程争抢一个资源



执行结果：

```
begin run
real true
Thread-1  false
begin run
real true
Thread-0  false
```



![image.png](https://cdn.nlark.com/yuque/0/2020/png/1762736/1596968773549-bff2eb69-c991-4017-821a-4b74d6a18ae9.png)



不管是AtomicBoolean还是AtomicInteger还是AtomicReference，都有compareAndSet方法

从最简单的AtomicBoolean看

把compreAndSet拆开为compare和Set方法理解

是compare之后，就马上设置共享内存为false（CAS操作是底层支持的原子操作，所以其他的不能修改共享内存中的值）。

这个时候其他线程无论怎么走都走不到 共享内存为true时的代码块。





![image](https://gitee.com/liying000/blogimg/raw/master/1596968627455-1d87cebd-9faa-4c4c-b2d8-7675e6902129.jpg)





如果把if中的set(true)拿掉，也就是，有一个线程设置为共享资源为false成功之后没有改回来



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596971002787-1badb5f5-950a-42bb-beda-e05520407e01.png)



深递归导致栈被耗尽了……………………

也就是一直等待hhhhhhh；

把else的代码改为输出信息



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596971138842-fdbf990c-83e8-4b22-8415-09345c07118d.png)



![image.png](https://gitee.com/liying000/blogimg/raw/master/1596971361084-80daa965-40e7-4daf-bf9b-7fdb6be7c156.png)

谁能抢到临界资源是不确定的。

抢到了就底层操作系统支持的cmpxchg操作了，这原子操作了。



bingo！！！周日回宿舍看电影啦ヾ(•ω•`)o。





## 疑问：

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596969649101-6cd2c47f-7094-4dc6-8372-1dbb60cc65d4.png)

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596969665291-9819a4b9-3d62-4d99-85af-a9364a6fca2a.png)

![image.png](https://gitee.com/liying000/blogimg/raw/master/1596969681445-ecce847c-891b-4342-a9ff-03b3b9ae800a.png)

好多atomic类的方法是相同的，为什么没有设计一个共用的接口？