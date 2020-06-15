---
title: jdk动态代理的关键点思考
date: 2020-06-13 23:33:21
tags:
	- Java
---

## 问题：
动态代理生成的类在哪里？
[JDK动态代理文件$Proxy0.class的生成和查看](https://blog.csdn.net/ywlmsm1224811/article/details/92583559?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.nonecase)
```java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles","true");
```
<!--more-->

必须在main方法中测试，Junit不行(我没用Junit试，教程里这么写的)。
生成的class文件在当前工作目录下的
![](https://gitee.com/liying000/blogimg/raw/master/1266.jpg)


Idea直接打开
![](https://gitee.com/liying000/blogimg/raw/master/1267.jpg)
动态类继承了Proxy，实现了Rent接口（我们自己定义的接口）
我们在使用的时候：
```java
        ProxyInvocationHandler pih = new ProxyInvocationHandler();
        pih.setRent(host);//给pih设置真实角色的权力
        Rent proxy = (Rent)pih.getProxy();//把代理对象赋值给代理
```
Rent实际上是指向了getProxy⬅Proxy.newProxyInstance
而反汇编中可以看到，生成的动态类实际上实现了Rent接口，所以可以赋值给Rent的引用。
动态类实现的rent方法:
```java
public final void rent() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
```
主要操作：调用父类的h的invoke方法
![](https://gitee.com/liying000/blogimg/raw/master/1268.jpg)
h是InvocationHandler 对象

先看一下Proxy的newProxInstance
newProxyInstance()方法有三个参数： 
1、类加载器（ClassLoader）用来加载动态代理类。
> 我的理解：加载生成的Proxy0.class 

2、一个要实现的接口的数组。 
>在我们的程序里，只有一个接口：Rent（当然这个接口可以继承多个接口）`public interface Rent extends Rental,Rent3{}`用来保存所有接口。

3、一个 InvocationHandler 把所有方法的调用都转到代理上。

看的视频教程里的老师这么写：
```java
//InvocationHandler的实现类中
public Object getProxy(){
        return Proxy.newProxyInstance(this.getClass().getClassLoader(),rent.getClass().getInterfaces(),this);
    }
```
一直没有理解classLoader为什么会是InvocationHandler的实现类的clssLoader
[https://wiki.jikexueyuan.com/project/java-reflection/java-dynamic.html](https://wiki.jikexueyuan.com/project/java-reflection/java-dynamic.html)
这篇中这么写：
```java
public Object getProxy(){
        return Proxy.newProxyInstance(Rent.class.getClassLoader(),new Class[] {Rent.class},this);
    }
```
应该更合理吧？
既然生成的是Rent的实现类，那么用Rent的classLoader更合理？不过不同的类的classLoader有什么区别嘛？

> 更新： 不同类的ClassLoader是一样的

然后第二个变量是接口数组，理解（￣︶￣）↗　
第三个变量是 InvocationHandler的实现类的一个实例，理解（￣︶￣）↗　

那么：动态类到底怎么生成的呢？
(oﾟvﾟ)ノ没猜错的话要看newProxyInstance的方法实现了o_o ....

##newProxyInstance实现
```java
 public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
```
// 检查 h 不为空，否则抛异常 
 ```java
          Objects.requireNonNull(h);
 ```

   // 对传入的接口做安全检查

```   java
        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }
```
  /*
       * 实现动态代理的核心方法，动态代理的思路便是生成一个新类，刚刚getProxyClass0便成为了生成新类
         */
```java
        Class<?> cl = getProxyClass0(loader, intfs);
```
 /*
         * 根据生成的class通过反射获取构造函数对象并生成代理类实例
                  */

```java
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
生成类的部分在于：
```java
Class<?> cl = getProxyClass0(loader, intfs);
```
实现代码如下：
```java
/**
     * Generate a proxy class.  Must call the checkProxyAccess method
     * to perform permission checks before calling this.
     */
    private static Class<?> getProxyClass0(ClassLoader loader,
                                           Class<?>... interfaces) {
        if (interfaces.length > 65535) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        // If the proxy class defined by the given loader implementing
        // the given interfaces exists, this will simply return the cached copy;
        // otherwise, it will create the proxy class via the ProxyClassFactory
        return proxyClassCache.get(loader, interfaces);
    }
```
最后一句，有缓存直接从缓存中拿，没有缓存通过ProxyClassFactory产生一个（又是工厂模式）

结论：
只能生成动态代理类的一个实例，拿不到动态代理类，因为动态代理类在内存中。
那么：

## 何时生成动态代理类？
<img src="https://gitee.com/liying000/blogimg/raw/master/1269.jpg" style="zoom:80%;" />

对上面的这个图，我们简单来说说：客户端动态生成代理类，然后调用代理类的方法。
代理类内部调用handler.invoke（）方法
在invoke中呢，又指向目标类，调用目标类的方法
invoke在整个项目＋lib中的应用，红框中是Spring的

![](https://gitee.com/liying000/blogimg/raw/master/1271.jpg)


![](https://gitee.com/liying000/blogimg/raw/master/1270.jpg)



[https://zhuanlan.zhihu.com/p/42516717](https://zhuanlan.zhihu.com/p/42516717) 中举了下面的例子：

```java
package hello;

public interface People {
    public String work();
}

package hello;

public class Teacher implements People {

    @Override
    public String work() {
        System.out.println("老师教书育人");
        return "教书";
    }
}

package hello;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class WorkHandler implements InvocationHandler {
    private Object obj;
    public WorkHandler(){

    }
    public WorkHandler(Object obj){
        this.obj = obj;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before invoke");
        Object invoke = method.invoke(obj,args);
        System.out.println("after invoke");
        return invoke;
    }
}

package hello;


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;

public class Clinet {
    public static void main(String[] args) {
        People people = new Teacher();
        InvocationHandler ih = new WorkHandler(people);
        People proxy = (People) Proxy.newProxyInstance(ih.getClass().getClassLoader(),people.getClass().getInterfaces(),ih);
        proxy.work();
    }
}

```





