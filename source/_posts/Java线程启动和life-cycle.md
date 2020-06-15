---
title: Java线程启动和life cycle
date: 2020-05-27 00:10:46
tags:
    - Java
    - 多线程
---
## Java线程VS操作系统线程
概述：Java线程跟操作系统线程是1:1的关系
<!--more-->
```
new Thread() {
			public void run() {
				System.out.println("new Thread");
			}
		}.start();
```
查看start的实现：
```
public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
....
```
执行的主要部分是start0()这个方法
![start0()](https://gitee.com/liying000/blogimg/raw/master/1256.jpg)
start0()是`native`函数。。。是通过C/C++实现的。native方法是通过java中的JNI实现的。JNI是Java Native Interface的 缩写。从Java 1.1开始，Java Native Interface (JNI)标准成为java平台的一部分，它允许Java代码和其他语言写的代码进行交互。

```
/* openjdk\jdk\src\share\native\java\lang\Thread.c */
...
static JNINativeMethod methods[] = {
    {"start0",           "()V",        (void *)&JVM_StartThread},
    {"stop0",            "(" OBJ ")V", (void *)&JVM_StopThread}
...
};
...
JNIEXPORT void JNICALL
Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
}
```
Java_java_lang_Thread_registerNatives()实现了多个函数的注册（我：这些函数到底是怎么注册的¯\_(ツ)_/¯，困扰很久了），其中`"start0",           "()V",        (void *)&JVM_StartThread`，`JVM_StartThread`方法位于src/hotspot/share/prims/jvm.cpp，对应代码如下：
```
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  bool throw_illegal_thread_state = false;

  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {
    // 获取互斥锁
    MutexLocker mu(Threads_lock);

    // 线程状态检查，确保尚未启动
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
    
      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));

      NOT_LP64(if (size > SIZE_MAX) size = SIZE_MAX;)
      size_t sz = size > 0 ? (size_t) size : 0;

     
      native_thread = new JavaThread(&thread_entry, sz);

      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        native_thread->prepare(jthread);
      }
    }
  }

  Thread::start(native_thread);

JVM_END
```
下面这部分解读出自[知乎](https://zhuanlan.zhihu.com/p/34414549)

1：判断Java线程是否已经启动，如果已经启动过，则会抛异常。
```
if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
 }
```
2.如果第一步判断中，Java线程没有启动过，则会开始创建Java线程。
```
jlong size = java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
size_t sz = size > 0 ? (size_t) size : 0;
native_thread = new JavaThread(&thread_entry, sz);
```
Java线程的创建过程主要就在JavaThread的构造函数中：
```c++
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz) :
                       Thread()
{
  initialize();
  _jni_attach_state = _not_attaching_via_jni;
  set_entry_point(entry_point);
  os::ThreadType thr_type = os::java_thread;
  thr_type = entry_point == &compiler_thread_entry ? os::compiler_thread :
                                                     os::java_thread;
  os::create_thread(this, thr_type, stack_sz);
}

```
最后一句os::create_thread(this, thr_type, stack_sz) 便开始真正的创建Java线程对应的内核线程。
```
bool os::create_thread(Thread* thread, ThreadType thr_type,
                       size_t req_stack_size) {
    ......
    pthread_t tid;
    int ret = pthread_create(&tid, &attr, (void* (*)(void*)) thread_native_entry, thread);
    ......
    return true;
}
```
上面这个方法主要就是利用pthread_create（）来创建线程。其中第三个参数thread_native_entry便是新起的线程运行的初始地址，其为定义在os_bsd.cpp中的一个方法指针，而第四个参数thread即thread_native_entry的参数：
```
static void *thread_native_entry(Thread *thread) {
  ......
  thread->run();
  ......
  return 0;
}
```
>os:终于出现run方法了
新线程创建后就会从thread_native_entry（）开始运行，thread_native_entry（）中调用了thread->run（）：
```
// thread.cpp
void JavaThread::run() {
  ......
  thread_main_inner();
}
```
就到这吧，大体上就是这样一个过程
![](https://gitee.com/liying000/blogimg/raw/master/1257.jpg)
## Java线程状态切换
那么JVM中线程的状态是如何转换的？下面有几张图QAQ

![](https://gitee.com/liying000/blogimg/raw/master/v2-326a2be9b86b1446d75b6f52f54c98fb_720w.jpg)
![](https://gitee.com/liying000/blogimg/raw/master/1258.jpg)

下图参考《并发编程的艺术》
![](https://gitee.com/liying000/blogimg/raw/master/v2-20edff079dc147b795e08261be1161f4_720w.jpg)
下面这张图来自[Core Java](https://www.google.co.in/search?q=Core+Java+Vol+1%2C+9th+Edition%2C+Horstmann%2C+Cay+S.+%26+Cornell%2C+Gary_2013&oq=Core+Java+Vol+1%2C+9th+Edition%2C+Horstmann%2C+Cay+S.+%26+Cornell%2C+Gary_2013&aqs=chrome..69i57.383j0j7&sourceid=chrome&ie=UTF-8)
![](https://gitee.com/liying000/blogimg/raw/master/threadLifeCycle.jpg)
Stack Overflow上的图
![image.png](https://gitee.com/liying000/blogimg/raw/master/1259.jpg)
bitstechnotes个人站
![image.png](https://gitee.com/liying000/blogimg/raw/master/1260.jpg)
![image.png](https://gitee.com/liying000/blogimg/raw/master/1262.jpg)

![image.png](https://gitee.com/liying000/blogimg/raw/master/1261.jpg)

![](https://gitee.com/liying000/blogimg/raw/master/1263.jpg)
![image.png](https://gitee.com/liying000/blogimg/raw/master/1264.jpg)

廖大大的状态转移图更简单：
![](https://gitee.com/liying000/blogimg/raw/master/1265.jpg)
当线程启动后，它可以在Runnable、Blocked、Waiting和Timed Waiting这几个状态之间切换，直到最后变成Terminated状态，线程终止。

Thread.State枚举类定义了Java线程的六个状态：
A thread state.  A thread can be in one of the following states:
```java
public enum State {
        NEW,
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
    }
```
**These states are virtual machine states which do not reflect any operating system thread states.**
也就是说，Java线程的状态是在JVM中的状态，跟操作系统的线程状态是没有关系的。

### NEW
 NEW is waiting to be executed
新创建了一个Thread对象，但是还没有start：
```java
Thread thread = new Thread(new MyRunnable());
System.out.println(thread.getState());
```
结果：`NEW`
>神奇
### Runnable
```java
class MyThread extends Thread{

	@Override
	public void run() {
		// TODO Auto-generated method stub
		System.out.println("thread start");
		System.out.println("State of thread in run() - "+this.getState());
		System.out.println("thread end");
	}
	
}
public class Main {
	public static void main(String[] args) {
		Thread thread = new MyThread();
		System.out.println("State of thread after creating it - "+ thread.getState());
		thread.start();
	}
}
```
结果：
```java
State of thread after creating it - NEW
thread start
State of thread in run() - RUNNABLE
thread end
```
在start之前，state是NEW，调用start执行run方法，线程就是RUNNABLE 状态。

### BLOCKED
A running thread may enter the blocked state as it waits for a monitor lock to be freed. It may also be blocked as it waits to reenter a monitor lock after being asked to wait using the Thread.wait() method.
[讲解](https://www.javabrahman.com/corejava/understanding-thread-life-cycle-thread-states-in-java-tutorial-with-examples/)
```
public class Main {
	private static Object lock = new Object();
	public static void main(String[] args) {
		
		Thread thread1 = new Thread(()-> {
            synchronized (lock){
                try {
                    Thread.sleep(Integer.MAX_VALUE);
                }
                catch (InterruptedException e) {}
            }
        });
        Thread thread2 = new Thread(()-> {
            synchronized (lock){
            }
        }
        );
        thread1.start();
        try {
			Thread.sleep(50);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
        System.out.println(thread1.getState());//TIMED_WAITING
        thread2.start();
        try {
			Thread.sleep(50);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
        System.out.println(thread2.getState());		//BLOKED
		
	}
}
```
[https://www.youtube.com/watch?v=TCd8QIS-2KI](https://www.youtube.com/watch?v=TCd8QIS-2KI)
### TIMED WAITING
The reason why a thread enters a finite waiting state is that one of the following methods is invoked:

Thread.sleep(long millis)
Object wait (long timeout) with timeout
Thread#join(long millis) with timeout
LockSupport.parkNanos(Object blocker, long nanos)
LockSupport.parkUntil(Object blocker, long nanos)
[https://www.cnblogs.com/paddix/p/5381958.html](https://www.cnblogs.com/paddix/p/5381958.html)