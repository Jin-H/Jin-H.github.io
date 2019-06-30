#### 最近写代码的时候遇上不少坑，是时候记录一下成长的过程了

#### 问题简述

>  Spring MVC 整合 Mybatis 之后始终无法通过 spring 的 property 注入 basePackage 的值，也就是说 spring 容器无法扫描到 mapper 文件所在的包。

![image-20190630134459229](http://ww4.sinaimg.cn/large/006tNc79ly1g4j4s0x40sj30hg00sjrb.jpg)

无论怎么修改 basePackage的值，都无法注入进去

![image-20190630134315202](http://ww1.sinaimg.cn/large/006tNc79ly1g4j4q7smpmj317s03wwfp.jpg)



#### 问题分析

##### 1.  spring 解析placeHolder的时候出错了

> 但是尝试别的属性注入都是没问题的，所以这个结论可以推翻

##### 2.  placeHolder解析流程没走到

> 如果别的占位符都可以解析，那么只能说明 MapperScannerConfigurer 对象有些特殊，解析流程不太普通

#### debug 流程

#####  MapperScannerConfigurer 对象

> 为了简化篇幅，这里只展示部分有必要的代码块了

```java
public class MapperScannerConfigurer implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
  /**
   * 这里忽略其余字段
   */
  private String basePackage;
  //注意该字段
  private boolean processPropertyPlaceHolders; 
  
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }
  }
  
  
}
```

##### BeanDefinitionRegistryPostProcessor 接口

> 继承自 BeanFactoryPostProcessor
>
> 用于在初始化之前修改容器中的  bean definition register 

![image-20190630132654320](http://ww3.sinaimg.cn/large/006tNc79ly1g4j49buc5ej317o0c2ad5.jpg)

如果对于 `spring` 容器  `bean` 初始化流程稍微了解一点的同学应该可以想到在 `AbstractApplicationContext` 中有一个 `refresh` 方法，在 `refresh` 方法中有 `invokeBeanFactoryPostProcessors`

![image-20190630133155686](http://ww2.sinaimg.cn/large/006tNc79ly1g4j4efojylj30m20nun0o.jpg)

进入这个方法查看一下，最终我们发现

![image-20190630133617185](http://ww1.sinaimg.cn/large/006tNc79ly1g4j4iz2nhxj310u0htwjk.jpg)

显然我们看到了我们所需要的信息，那就再进去看看

![image-20190630133709385](http://ww3.sinaimg.cn/large/006tNc79ly1g4j4jvf1jaj30sg0gytd0.jpg)

走到这一步基本上我们可以看出一些端倪了，如果容器初始化做了手脚的话，肯定在这里了，仔细看 if 分支中的方法名 `processPropertyPlaceHolders` 不出意外的话，应该就是这个了

![image-20190630141743332](http://ww1.sinaimg.cn/large/006tNc79ly1g4j5q35g2rj30px0a576z.jpg)

果然，这里会重新更新 basePackage 的值

#### 验证结果

> 我们多加一行代码，通过 setter 方法注入 MapperScannerConfigurer 对象的 processPropertyPlaceHolders 属性

![image-20190630134018362](http://ww3.sinaimg.cn/large/006tNc79ly1g4j4n5a72dj3190056gn7.jpg)

![image-20190630135450832](http://ww2.sinaimg.cn/large/006tNc79ly1g4j52a8huej30hm011t8n.jpg)

![image-20190630134118004](http://ww3.sinaimg.cn/large/006tNc79ly1g4j4o6vcltj30xd0f5wj2.jpg)

至此，我们发现 `basePackage` 的值成功注入进去了！