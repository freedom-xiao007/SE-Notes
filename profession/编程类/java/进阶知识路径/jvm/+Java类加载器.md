# Java类加载器
***
## 简介
&ensp;&ensp;&ensp;&ensp;介绍类加载器的分类和特性

## 类加载器
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3d2936144d04af09f6e59a25a545ef7~tplv-k3u1fbpfcp-watermark.image)

### 加载器类型
&ensp;&ensp;&ensp;&ensp;如上图所示，类加载器大致可分为四大类，如上他们是有父子关系的，启动类加载器最先开始（用C++写的），后面加载上子加载器

- 1.启动类加载器
- 2.扩展类加载器
- 3.应用类加载器
- 4.自定义类加载器

&ensp;&ensp;&ensp;&ensp;在Java9之前，大致类加载器功能如下：

&ensp;&ensp;&ensp;&ensp;启动类加载器：负责加载最为基础的类、最为重要的类，比如存放在JRE的lib目录下的jar包中的类（以及由虚拟机参数 -Xbootclasspath 指定的类）

&ensp;&ensp;&ensp;&ensp;扩展类加载器：负责加载相对次要、但又通用的类，比如存放在JRE的lib/ext目录的jar包重点类（以及由系统变量java.ext.dirs指定的类）

&ensp;&ensp;&ensp;&ensp;应用类加载器：负责加载应用程序路径下的类（应用程序路径指虚拟机参数 -cp/-classpath、系统变量 java.class.path 或环境变量 CLASSPATH所指定的路径）

&ensp;&ensp;&ensp;&ensp;自定义类加载器：负责加载自定义的特殊的类。比如对class文件进行加密后，利用自定义的类加载器进行解密加载。

&ensp;&ensp;&ensp;&ensp;自定义加载器还有一个骚操作，功能描述大致如下：

> 在 Java 虚拟机中，类的唯一性是由类加载器实例以及类的全名一同确定的。即便是同一串字节流，经由不同的类加载器加载，也会得到两个不同的类。在大型应用中，我们往往借助这一特性，来运行同一个类的不同版本。

### 双亲委托机制
&ensp;&ensp;&ensp;&ensp;类加载器有个非常重要的机制：双亲委托机制

> 当类加载器要加载一个类时，如果自己曾经没有加载过这个类，则层层向上委托给父加载器尝试加载。如果父加载器都没有，才自己进行加载

&ensp;&ensp;&ensp;&ensp;为什么需要双亲委托机制呢：因为用它可以保证类的唯一性

### 添加引用类的几种方式
&ensp;&ensp;&ensp;&ensp;通过的前面的介绍，我们可以得到下面几种引用类的添加方式

- 1.放到JDK的 lib/ext 下，或者 -Djava.ext.dirs
- 2.java -cp/classpath 或者 class 文件放到当前目录
- 3.自定义 ClassLoader 加载
- 4.拿到当前执行类的 ClassLoader，反射调用 addURL 方法添加 Jar 或者 路径(JDK9无效)

### 自定义加载器的一些玩法
#### 加载两个同名类
&ensp;&ensp;&ensp;&ensp;虽然双亲委托机制保证了类的唯一性，但是只是父子关系才行，如果是兄弟关系，那就可以加载同一个类

&ensp;&ensp;&ensp;&ensp;在前面说过，类的唯一性是由类加载器+类名进行确定的,如同下面的代码，虽然都是同一个类，但通过不同的类加载器进行加载，被视为不同的类

```java
import java.lang.reflect.Method;
public class Test {
    public static void main(String[] args) {
        CustomClassLoader classLoader1 = new CustomClassLoader("/Users/yu/Desktop/lib");
        CustomClassLoader classLoader2 = new CustomClassLoader("/Users/yu/Desktop/lib");
        try {
            Class c1 = classLoader1.findClass("Person");
            Object instance1 = c1.newInstance();

            Class c2 = classLoader2.findClass("Person");
            Object instance2 = c2.newInstance();

            Method method = c1.getDeclaredMethod("setPerson", Object.class);
            method.invoke(instance1, instance2);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 参考资料
- [探秘类加载器和类加载机制](https://juejin.cn/post/6844903780031397902#heading-10)
- [03 | Java虚拟机是如何加载Java类的?](https://time.geekbang.org/column/article/11523)
- [第24讲 | 有哪些方法可以在运行时动态生成一个Java类？](https://time.geekbang.org/column/article/10076)