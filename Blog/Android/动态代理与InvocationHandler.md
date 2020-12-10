---
title: 动态代理与InvocationHandler
date: 2020-4-3
categories: Android
---

在说动态代理之前，我们先看一下静态代理。说起静态代理，很多人都会会心一样，因为它太简单了。

### 静态代理

看下面的一个例子：

```java
public interface Movie {
    void play();
}

public class RealMovie implements Movie {

    @Override
    public void play() {
        // TODO Auto-generated method stub
        System.out.println("您正在观看电影 《肖申克的救赎》");
    }

}


public class Cinema implements Movie {

    RealMovie movie;

    public Cinema(RealMovie movie) {
        this.movie = movie;
    }


    @Override
    public void play() {
        guanggao(true);
        movie.play();
        guanggao(false);
    }

    public void guanggao(boolean isStart){
        if (isStart) {
            System.out.println("电影马上开始了，爆米花、可乐、口香糖9.8折，快来买啊！");
        } else {
            System.out.println("电影马上结束了，爆米花、可乐、口香糖9.8折，买回家吃吧！");
        }
    }

}
```

上面的代码里面，Cinema就是一个代理类，它代理了 RealMovie 对象。一般来说，代理类与被代理类都会实现同一个接口，这样使用者就不用管它到底是不是代理类。使用者只需要知道它可以实现想要的功能就行了。

甚至有时候，我为了偷懒都不想去写接口，只是写一个包装类做些处理，然后将请求转发给被代理类。

所以，静态代理就是这样的一个东西，代理类可以预先处理一些东西，而被代理类是实现真正的功能。



### 动态代理

我们再来看看动态代理。Java 里面使用动态代理很简答，我们看代码：

```java
Proxy.newProxyInstance(targetObject.getClass().getClassLoader(), targetObject.getClass().getInterfaces(), DInvocationHandler());  
```

Proxy.newProxyInstance 就是为了生成一个代理类，上面说静态代理的时候已经说了啥是代理类。这里我们不去追踪源码研究这个类是怎么生成的，我们直接看看它生成的类是个什么样子！！！

这是一个接口：

```java
public interface ICook {
     void dealWithFood();

     void cook();
}
```

这里是生成的代理类：

```java
public final class $Proxy0 extends Proxy implements ICook {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m4;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void cook() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void dealWithFoot() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("com.company.ICook").getMethod("cook", new Class[0]);
            m4 = Class.forName("com.company.ICook").getMethod("dealWithFoot", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

是不是没什么惊喜！它和静态代理的代理类是一样的功能，将请求转发给被代理类。

只不过这里它多了一个东西，那就是 InvocationHandler。那么为什么要多一个 InvocationHandler 呢？

显然是因为这个类是动态生成的，我们无法修改它，那么我们想在代理类里面搞事的想法不就无法实现了吗！但是这难不倒Java设计人员，它搞了一个接口 InvocationHandler ，将每个请求的转发转给 InvocationHandler，然后由实现者自己再去将请求转发给代理类。

所以，动态代理的本质与静态代理是一样的，它只不过是将请求先转发到 InvocationHandler，然后由开发者自己再去将请求转发给被代理类。