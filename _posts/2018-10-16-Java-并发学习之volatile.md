### Java 并发学习之 volatile

#### 一、volatile的用法

对于`volatile`关键字最广为人知的用法应该是用在双检锁的单例模式中了：

```java
public class Singleton{
    private Singleton(){}
    
    private static volatile INSTANCE = null;
    
    public statis Singleton getInstance(){
        if(INSTANCE == null){
            synchronized(Singleton.class){
                if(INSTANCE == null){
                    INSTANCE = new Singleton();//非原子性操作
                }
            }
        }
        return INSTANCE;
    }
}
```

上述代码实现了[单例模式](https://jin-h.github.io/2018/09/14/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F-%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F/)，在这段代码中使用`volatile`关键字的原因是为了保证变量`INSTANCE`的可见性，因为`INSTANCE = new Singleton()`不是原子性操作，所以要保证这个过程是对于其余线程是可见的。

#### 二、volatile关键字的特性

##### 1.内存可见性

![jmm](https://ws4.sinaimg.cn/large/006tNbRwly1fwa1wbwhpaj30sg0lc0u8.jpg)

在介绍内存可见性之前我们可以先了解一下`happens-before`语义，在`Java`语言规范中规定了[happens-before](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.5)语义。

如果我们有两个操作x、y，我们可以用hb(x,y)来表示x happens-before y

- 如果x，y是同一个线程的两个操作，并且在代码中x先于y，则hb(x,y)
- 对象构造器的最后一行指令happens-before于finalizer方法的第一行指令
- 如果x操作与接下来的y操作构成同步，则hb(x,y)
- 如果hb(x,y),hb(y,z)则有hb(x,z)。即传递性

Java线程先从主存中获取共享变量的值，然后存储一份副本在工作内存之中，线程执行完毕再把结果刷新到主存中。这样的话，如果多个线程获取到共享变量都做了修改，就会出现并发问题。volatile通过内存屏障让工作内存中的副本失效，这样每次更改都能让其他线程见到最新修改过的值。

##### 2.顺序性

顺序性说的是现代编译器在编译的时候会对指令重排序以达到提高cpu运行效率的目的。如果一个操作不是原子性操作，那么编译成字节码之后就有可能出现并发问题。

#### 三、实现原理

在jvm中`volatile`是通过***[内存屏障](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C)***来实现的，通过反编译`class`文件可以看到汇编中使用`lock`指令来实现内存屏障。

> 语义上，内存屏障之前的所有写操作都要写入内存；
>
> 内存屏障之后的读操作都可以获得同步屏障之前的写操作的结果。
>
> 因此，对于敏感的程序块，写操作之后、读操作之前可以插入内存屏障

#### 四、总结

##### 1.***`volatile`无法保证原子性***

##### 2.`volatile`适用于对写操作不依赖于当前值

#### 五、引用

##### 1.[内存屏障](https://zh.wikipedia.org/wiki/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C)

##### 2.[`happens-before`语义](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4.5)
