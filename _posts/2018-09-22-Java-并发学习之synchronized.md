---
layout:     post
title:      Java 并发学习之 synchronized
date:       2018-09-22
author:     redscarf
catalog: true
tags:
	- Java
	- 并发
---

### `Java` 并发学习之`synchronized`关键字

#### 一、前言

该篇文章是基于我的日常使用经验及看书学习的总结，文章中难免会出现疏漏或者理解不正确的地方，如果读者发现问题，可通过邮箱(hongweijin1993#gmail.com)与我联系，谢谢。

#### 二、`synchronized`三个特性

##### 1.***原子性***

此处的原子性说的是保证了代码块或者方法为最小执行单位，要么全部成功，要么全部失败，不存在中间状态。

##### 2.***内存可见性***

谈到可见性的时候，我们应该首先清楚，在jvm中分为***主存***和***工作内存***，工作内存是当前线程私有的区域，而主存是所有线程共享的区域。

使用`synchronized`关键字的时候会创建一个内存屏障，内存屏障指令保证了所有CPU操作结果都直接刷新到主存中。

对于***共享变量***来说，每个线程的工作内存中都会存有一个副本，当a线程修改了共享变量的值之后，会先缓存到当前线程的工作内存中。当加锁之后，缓存会失效，从而保证每次修改都会刷新到主存中。同时读取共享变量也是直接从主存中读取，这样就能保证thread1修改了变量的值之后，thread2能够在读取的时候读到最新的值。

***可见性中的的可见其实是对进入到同步块中的线程可见。***

 `synchronized`关键字对于非同步块中的变量也有可能做到内存可见，不过目前没有权威的说法来支持，不过有个[demo可以看一看](http://www.cnblogs.com/cookiezhi/p/5774583.html)。

##### 3.***有序性***

有序性是指程序按照代码的先后顺序执行，***`synchronized`关键字无法禁止指令重排序和处理器优化(这块内容可以看看volatile的语意)。***

> 如果在本线程内观察，所有操作都是天然有序的。如果在一个线程中观察另一个线程，所有操作都是无序的。

#### 三、`synchronized`的使用

##### 1.用法方法上

###### 1.1 普通的实例方法

```java
public class SynchronizedMethod{
    //对实例对象加锁，每次new对象都会生成新的实例对象
    public synchronized void println(){
        System.out.println("synchronized instance method");
    }    
}
```

###### 1.2  静态方法

```java
public class SynchronizedStaticMethod{
    //是对Class对象加锁
    public synchronized static void println2(){
        System.out.println("synchronized static method");
    }
    //SynchronizedTest.class 是Class对象，只有一个
}
```

###### 1.3 有继承关系的方法(其实也就是普通的实例方法，此处仅用来说明内置锁的重入)

```java
class Widget{
    //method2
    public synchronized void doSomething{
        System.err.println("----widget----");
    }
}
public class class LoggingWidget extends Widget{
    //method1
    public synchronized void doSomething{
        System.err.println("----logging widget----");
        super.doSomething();
    }
    
    public static void main(String[] args){
        Widget widget = new LoggingWidget();
        widget.doSomething();
    }
}
```

上面的代码块选自`Java并发编程实战`，此时执行`main`函数时，锁住的是`LoggingWidget`对象。

> 1.当主线程执行到method1时，获取到了LoggingWidget对象上的锁。
>
> 2.当主线程执行到method2时，再次遇到synchronized关键字，此时重入
>
> 3.当执行完method2时，退出一次
>
> 4.当执行完method1时，再退出一次

可能会产生疑问就在与上面的第二条，锁重入了。我们知道synchronized加在实例方法上的时候，锁住的是实例对象，此时线程持有了该实例对象上的锁，当再次遇到synchronized的时候，锁还是原有实例的，这个时候锁重入了。

##### 2.用在代码块上

###### 2.1 锁住Class对象

```java
public class SynchronizedClass{
    public void println(){
        synchronized(SynchronizedClass.class){
            System.out.println("synchronized Class object");
        }
    }
}
```

###### 2.2 锁住实例对象

```java
public class SynchronizedInstance{
    public void println(){
        //对实例对象加锁，this换成其他对象也是同理
        synchronized(this){
            System.out.println("synchronized instance objcet");
        }
    }
}
```

##### 3.总结

> 1.`Java`中每个对象都会有一个`monitor`，`synchronized`实际上是通过`monitor`锁住对象的
>
>  2.`Class`对象和实例对象的区别(实例对象是每次`new`出来的对象，`Class`对象是` ClassName.class`对象)
>
> 3.同一个类中两个`synchronized static`方法之间构成同步，因为锁住的都是`Class`对象
>
> 4.同一个类中`synchronized static`方法和`synchronized`方法不构成同步，因为一个锁住`Class`对象，一个锁住实例对象
>
> 5.谁调用加锁的实例方法，就获取谁身上的锁
>
> 6.内置锁是可以重入的。

#### 四、`synchronized`的实现原理

##### 1.当`synchronized`用在***代码块***上的时候，我们使用`javap`指令反编译(`javap -c SynchronizedBlock`)上面代码的时候会发现两个汇编指令：`monitorenter`、`monitorexit`

```java
public class SynchronizedBlock{
    public void test(){
        synchronized(this){}
    }
}
```

![image-20180921140348995](https://ws3.sinaimg.cn/large/006tNbRwgy1fvh4kqq5o1j31kw0qxwkk.jpg)

从上图我们可以看到反编译后的指令，这里面出现了一条`monitorenter`指令和两条`monitorexit`指令，`monitorenter`很好理解，通过这条指令获取到锁，然后进入代码块。那么两条`monitorexit`指令作何解释？

这是因为加锁时可能出现两种退出同步块的方式，一种是正常执行完退出，另一种就是同步块中出现异常，此时也会退出同步块。这两条指令第一条是用来正常退出的，第二条是编译器自己加上来用于异常退出同步块的。

##### 2.当`synchronized`用在方法上的时候，反编译会看到方法上被加上`ACC_SYNCHRONIZED`指令，我们对如下代码使用`javap -v SynchronizedMethod`指令反编译

```java
public class SynchronizedMethod{
    public synchronized void test(){}
}
```

![image-20180921135746665](https://ws1.sinaimg.cn/large/006tNbRwgy1fvh4egpyf0j31kw0zgak5.jpg)

从上图的红框中我们可以看到该关键字，在不同的jvm中会做不同的实现，此处参考[R大的读书笔记](https://book.douban.com/annotation/29492994/)。

#### 五、参考资料

##### 1.[Java并发编程实战](https://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)

##### 2.[Java语言规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.5)

##### 3.[R大的笔记](https://book.douban.com/annotation/29492994/)

##### 4.[JavaDoop博客](https://javadoop.com/post/java-memory-model#synchronized%20%E5%85%B3%E9%94%AE%E5%AD%97)

##### 5.[码瘾少年博客](http://www.cnblogs.com/cookiezhi/p/5774583.html)