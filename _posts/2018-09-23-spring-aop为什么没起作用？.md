---
title: spring aop为什么没起作用？
Category: 
 -spring
---

Spring Aop是Spring杀手锏功能之一，Aop可以在不改变原有代码的情况下，为方法添加诸如日志记录、事务保证、数据打点等增强功能。本文主要对笔者最近遇到的一个Aop失效问题进行总结，避免再次踩坑。

# 问题重现

首先是接口和实现类

```java
public interface CoreController {
    String invokeShow();
}
```

``` java
@Component
public class CoreControllerImpl implements CoreController{
    @Override
    public String invokeShow(){
        return this.show();
    }

    public String show(){
        return "Sing a song";
    }
}

```

接着我们希望在表演之前能有开场提示，提示使用aop实现，将show方法定义为切点，并为这个切点写一个前置方法

```java
@Aspect
@Component
public class MyAspect {
    @Pointcut("execution(public String com.gymn.aop.CoreControllerImpl.show())")
    public void pointCutShow() {}

    @Before("pointCutShow()")
    public void beforeAdvice(JoinPoint joinPoint) {
        System.out.println("表演即将开始："+joinPoint.getSignature().getName());
    }
}
```

最后在main方法中开始表演：

```java
public class AopMain {
    public static void main(String[] args) throws Exception{
        ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:config/spring.local/spring-*.xml");
        CoreController coreController = ctx.getBean(CoreController.class);

        String res = coreController.invokeShow();
        System.out.println(res);
    }
}
```

我们希望的是在歌手唱歌之前，主持人先念一下开场白，结果怎样呢？发现仅仅打印了“Sing a song”，aop前置方法并没有执行，这就尴尬了（不相信的小伙伴也可以自己debug证实一下）。

所以这个问题可以总结为：__对象内部调用切点方法不会触发aop__

# 原因分析

所以原理是什么？在spring中，aop是通过代理机制实现的,如果我们在其他service中调用show方法，那么方法最终实际上是在代理类中调用的，调用过程会像下面这样（仅做演示，实际实现细节并非这样）：

```java
public class CoreControllerProxy{
    CoreControllerImpl object;
    public Object invoke() {
        System.out.println("表演即将开始：");
        object.show();
    }
}
```

不幸的是，我们需要调用的是invokeShow()方法，于是情况变成了下面这样：

```java
public class CoreControllerProxy{
    CoreControllerImpl object;
    public Object invoke() {
        object.invokeShow();
    }
}
```

在调用invokeShow方法的时候，肯定已经存在CoreControllerImpl对象的，在invokeShow方法中再调用show就会直接调用已有对象中的方法，而不再走代理了。

# 解决方法

aop没有执行的原因是在类内部调用的时候没有走代理，所以解决方案就是让show()方法在调用的时候仍通过代理类来调用。你可以修改代码，避免同一个类中的方法调用，如果修改不便，也可以通过下面几个方法解决；

在配置文件中，配置：

```xml
<aop:aspectj-autoproxy proxy-target-class="true" expose-proxy="true"/>
```

invokeShow方法改成如下：

```java
@Override
    public String invokeShow(){
        return ((CoreControllerImpl) AopContext.currentProxy()).show();
    }
```

这种方法需要spring4.3以上版本

还有一种方式:

添加utill类：

```java
public class SpringContextUtil {
    private static ApplicationContext springContext;

    public static void init(ApplicationContext applicationContext) {
        springContext = applicationContext;
    }

    public static Object getBean(Class clz) {
        return springContext.getBean(clz);
    }
}
```

main方法中初始化一下：

```java
public class AopMain {
    public static void main(String[] args) throws Exception{
        ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:config/spring.local/spring-*.xml");
        SpringContextUtil.init(ctx);

        CoreController coreController = ctx.getBean(CoreController.class);

        String res = coreController.invokeShow();
        System.out.println(res);
    }
}
```

invokeMethod改写：

```java
@Override
public String invokeShow(){
    return ((CoreControllerImpl)SpringContextUtil.getBean(this.getClass())).show();
}
```

