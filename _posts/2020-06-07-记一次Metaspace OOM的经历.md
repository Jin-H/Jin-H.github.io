---
layout: post
title:      记一次 Metaspace OOM 的经历
subtitle:   
date:       2020-06-07
author:     redscarf
header-img:
catalog:    true
tags:
    - jvm
---

##### 缘起
 最近公司预发布环境有个服务总是不定时的爆出 OOM 的问题，报错信息如下：
```java
2020-05-25 11:49:14.348 ERROR 20553 --- [XNIO-1 task-8,159037855394497900,125420103360839681,241478959341376246,WE.1715.04] c.s.core.http.service.ExceptionHandlers  : 
Bug: Service have unknown exception org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.OutOfMemoryError: Metaspace     
at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1055)    
at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:943)  
at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006)  
at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898)
...// 这里省略掉没啥关系的异常信息
at java.lang.Thread.run(Thread.java:748) Caused by: java.lang.OutOfMemoryError: Metaspace
```

为了保证服务可用，当即找运维把 -XX:MaxMetaspaceSize 参数修改为 256m ，然后着手开始排查问题。

##### 明确 JVM 启动参数
由上述的报错信息我们可以很明显的看出来，是 Metaspace 爆掉了，所以先查询 JVM 启动参数，可以先看一看是否是因为参数配置不合理造成的。可以通过以下命令查询：
```arg
// 这里的 ${server_unique_flag} 代表着服务名称
jps | grep ${server_unique_flag} | awk '{print $1}' | xargs jinfo -flags
```
查询到的结果如下所示(已经手工格式化过了)：
```jinfo-flags
Attaching to process ID 28139, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.144-b01
Non-default VM flags: 
-XX:CICompilerCount=3 
-XX:InitialHeapSize=134217728 
-XX:+ManagementServer 
-XX:MaxDirectMemorySize=268435456 
-XX:MaxHeapSize=1073741824 
-XX:MaxMetaspaceSize=268435456 
-XX:MaxNewSize=357564416 
-XX:MetaspaceSize=104857600 
-XX:MinHeapDeltaBytes=524288 
-XX:NewSize=44564480 
-XX:OldSize=89653248 
-XX:+UseCompressedClassPointers 
-XX:+UseCompressedOops 
-XX:+UseParallelGC
Command line:  
-Xms128m -Xmx1024m \
-XX:MetaspaceSize=100m \
-XX:MaxMetaspaceSize=256m \
-XX:MaxDirectMemorySize=256m \
-Djava.security.egd=file:/dev/./urandom \
-Dfile.encoding=UTF-8 \
-Djava.rmi.server.hostname=${IP} \
-Dcom.sun.management.jmxremote.port=${port} \
-Dcom.sun.management.jmxremote.authenticate=true \
-Dcom.sun.management.jmxremote.ssl=false
```

可以看到相关的 Metaspace 参数有两个
* -XX:MaxMetaspaceSize=256m // 设置 metaspace 最大空间
* -XX:MetaspaceSize=100m 


##### 猜测可能引起问题的原因
在猜测可能引起问题的原因之前我们必须明确，metaspace 是用来做什么的。

> Metaspace 区域位于堆外，所以理论上它的最大内存大小取决于当前物理机的内存大小，当然我们也可以使用 `MaxMetaspaceSize` 参数来限定，以免服务吞噬内存引起别的问题。
Metaspace 主要用于存储 class metadata，class metadata 用于记录一个 Java 类在 JVM 中的信息，

* Klass 结构
* method metadata，包括方法的字节码、局部变量表、异常表、参数信息等
* 常量池
* 注解
* 方法计数器，记录方法被执行的次数
* 其他

基于这些信息以及对服务业务的理解，初步判断是有类加载器不断加载类导致最终 OOM 了。
在此基础上找运维把 GC 日志的全套参数加上，这里就不再赘述 JVM 关于 GC 日志的参数了。

##### 定位问题
当有了猜想之后问题就简单很多了，接下来只需要验证猜想是否成立即可。
使用如下命令可以看到 JVM 当前加载了多少类：
```class-loaded
jps | grep ${server_unique_flag} | awk '{print $1}' | xargs jstat -class
```
当即发现每运行一次，Loaded 的数量就会增长，这也印证了上述猜想。
接下来的方式就更简单了，通过分析服务日志，发现服务主要用于处理接收到的 mq 消息，而在处理的代码中发现一段代码不同寻常：
```parser-config
public static <T> T toCamelCaseObject(String json, Class<T> clazz) throws IOException {
        // 每调用一次，就会 new 一个对象出来
        ParserConfig parserConfig = new ParserConfig();
        parserConfig.propertyNamingStrategy = com.alibaba.fastjson.PropertyNamingStrategy.CamelCase;
        return JSONObject.parseObject(json,clazz,parserConfig);
} 
```
在这个问题解决之后，笔者非常好奇，为什么只有这个环境会出现 Metaspace OOM，而其他的环境不会出现这样的问题，最终查询到是因为当前环境其余服务异常了，导致大量的 mq 消息过来了，
最终引爆了 Metaspace。

##### 解决问题

其实 fastjson 的 `ParserConfig` 已经提供了一个 static 类型的 ParserConfig，所以我们并不需要每次都 new 对象，直接使用 `global` 即可。
```global-instance
public static ParserConfig getGlobalInstance() {
        return global;
}
public static ParserConfig     global                = new ParserConfig();
```

##### 释疑

之所以每次 new 对象就会造成类加载变多，看一下源码即可知道
```ParserConfig
public ParserConfig(){
        this(false);
}

public ParserConfig(boolean fieldBase){
    this(null, null, fieldBase);
}

private ParserConfig(ASMDeserializerFactory asmFactory, ClassLoader parentClassLoader, boolean fieldBased){
        this.fieldBased = fieldBased;
        if (asmFactory == null && !ASMUtils.IS_ANDROID) {
            try {
                if (parentClassLoader == null) {
                    // 罪魁祸首，每次都会 new ASMClassLoader
                    asmFactory = new ASMDeserializerFactory(new ASMClassLoader());
                } else {
                    asmFactory = new ASMDeserializerFactory(parentClassLoader);
                }
            } catch (ExceptionInInitializerError error) {
                // skip
            } catch (AccessControlException error) {
                // skip
            } catch (NoClassDefFoundError error) {
                // skip
            }
        }

        this.asmFactory = asmFactory;

        if (asmFactory == null) {
            asmEnable = false;
        }

        initDeserializers();

        addItemsToDeny(DENYS);
        addItemsToDeny0(DENYS_INTERNAL);
        addItemsToAccept(AUTO_TYPE_ACCEPT_LIST);

    }
```

在此基础上，翻阅 [Java 语言规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.3.1.1)中，对于 static 类型变量的阐述。
* If a field is declared static, there exists exactly one incarnation of the field, 
no matter how many instances (possibly zero) of the class may eventually be created. 
A static field, sometimes called a class variable, is incarnated when the class is initialized (§12.4).
* A field that is not declared static (sometimes called a non-static field) is called an instance variable. 
Whenever a new instance of a class is created (§12.5), a new variable associated with that instance is created 
for every instance variable declared in that class or any of its superclasses.

因为 `global` 是 static 类型的，所以无论有多少个 `ParserConfig` 最终被创建，都只有会创建一个 global，所以这样就避免了每次都会生成新的 ClassLoader。

如果 `ParserConfig` 没有提供 `global`，我们也可以使用单例模式的方式，创建一个。

##### 服务器 jdk 版本如下所示：


```jdk-version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
```

##### 参考资料及博客

* [Javadoop博客](https://www.javadoop.com/post/metaspace)
* [The Java Language Specification, Java SE 8 Edition](https://docs.oracle.com/javase/specs/jls/se8/html/index.html)