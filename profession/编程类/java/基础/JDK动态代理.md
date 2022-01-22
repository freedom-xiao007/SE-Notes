# JDK 动态代理
***
这是我参与2022首次更文挑战的第1天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7052884569032392740)

## 简介
本篇文章将以两个JDK常见问题为引，探索介绍JDK动态代理的基础知识点

## 问题
在面试中有下面两个常见的问题，看看你是否能信心十足的回答上来，如果可以，这篇文章那不必花费时间去看

- 1.JDK动态代理为啥不能对类进行代理?
- 2.抽象类可不可以进行JDK动态代理？

本人也是看文章的时候遇到这两个问题，但没有信心十足回答上来，所以有了这篇文章

## JDK动态代理的使用示例
我们先写一个简单的JDK动态代理示例代码，看看生成的代理类源码

源码工程GitHub地址：https://github.com/lw1243925457/JAVA-000/tree/main/homework/mybatis/jdkProxy

```java
// 接口类，定义了两个，主要是想看看两个接口的代理类情况
public interface IExample {

    String hello();
}

public interface IExtend {

    String hi();
}

// 实现类
public class ExampleImpl implements IExample, IExtend {

    @Override
    public String hello() {
        return "ExampleImpl hello";
    }

    @Override
    public String hi() {
        return "hi";
    }
}

// 代理类
public class ProxyInvocationHandler implements InvocationHandler {

    private Object target;

    public ProxyInvocationHandler(final Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("enter proxy handler");
        return method.invoke(target, args);
    }
}
```

最后写一个测试类，测试输出，并保存代理类的源码，方便后面进行查看

```java
public class ProxyTest {

    @Test
    public void test() throws IOException {
        final IExample example = new ExampleImpl();
        final IExample exampleProxy = (IExample) Proxy.
                newProxyInstance(example.getClass().getClassLoader(),
                        example.getClass().getInterfaces(),
                        new ProxyInvocationHandler(example));
        System.out.println(exampleProxy.hello());

        byte[] bts = ProxyGenerator.generateProxyClass("$ExampleProxy", example.getClass().getInterfaces());
        FileOutputStream f = new FileOutputStream(new File("D:\\Code\\Java\\self\\JAVA-000\\homework\\mybatis\\jdkProxy\\src\\main\\resources\\$GameProxy.class"));
        f.write(bts);
        f.flush();
        f.close();
    }
}
```

上面的运行结果也很简单：

```text
enter proxy handler
ExampleImpl hello
```

## JDK代理类源码
下面我们来看看测试中，保存下来的代理类源码，如下：

```java
public final class $ExampleProxy extends Proxy implements IExample, IExtend {
    private static Method m1;
    private static Method m4;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $ExampleProxy(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final String hi() throws  {
        try {
            return (String)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String hello() throws  {
        try {
            return (String)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
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

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("IExtend").getMethod("hi");
            m3 = Class.forName("IExample").getMethod("hello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

可以看到，在代理类的内部其中保留被代理对象的很多引用，而在我们定义的代理类InvocationHandler中，保留了对原对象的引用

```java
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("IExtend").getMethod("hi");
            m3 = Class.forName("IExample").getMethod("hello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
```

对于文章开头的两个问题，答案就在类定义上面了，不知道你们看出来了没有

```java
public final class $ExampleProxy extends Proxy implements IExample, IExtend
```

可以看到生成的代理类，默认继承了类：Proxy，实现了我们定义的两个接口：IExample, IExtend

在Java中，是单继承，也就是代理类已经没有办法再继承其他的类，这就是上面两个问题的答案

## 总结
通过实现代理类，查看代理类的源码，我们得到了开头的两个问题的答案：

- 1.JDK动态代理为啥不能对类进行代理?
- 2.抽象类可不可以进行JDK动态代理？

代理对应是自动生成的，其中已固定继承了Proxy类，无法再对其他的类和抽象类进行继承

## 参考链接
- [MyBatis 动态代理详解](https://mp.weixin.qq.com/s/L2rYzOCl7pdEce1ztN0zDg)
- [Android：JAVA语言extends和implements用法的学习](https://blog.csdn.net/qq_37858386/article/details/79024000)