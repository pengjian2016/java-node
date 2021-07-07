### 重点知识

1. 类的加载过程
   - 加载，链接（验证、准备、解析），初始化，使用，卸载，每一步做了什么都要熟悉，其他的如：类加载的时候将信息放在内存中哪个区（方法区，jdk8.0后称为元空间）；准备的时候干了什么（为类的static变量分配空间和初始化零值）等等

2. 双亲委派模型
   - 双亲委派模型的过程要知道，以及它的好处，不使用的话会有什么问题等，要打破双亲委派模型应该怎么做？（继承ClassLoader，重写loadClass方法），要自定义类加载器应该怎么做？（继承ClassLoader，重写findClass方法）

更多内容参考:[类的加载过程](https://gitee.com/javajov/java-senior-engineer-interview/blob/master/jvm/class.md)

这些重点知识，必须要非常熟悉，一般面试中也是先问这些东西，而后引出一堆其他问题。

我曾经面试的时候这两点也是回答的非常完美了，当我以为稳了的时候，面试官来了一句ClassLoader 和 Class.forName() 他们都能加载类，有什么不同？我又寂寞了，这个点我确实没有关注，所以当面试官把我能回答的‘他们都能加载类’这句话堵死了之后，我只能在心里说一句MMP了


### ClassLoader 和 Class.forName() 他们都能加载类，有什么不同？

ClassLoader（类加载器）只是将class加载到内存中，不进行其他操作，即只执行了类的加载阶段

Class.forName() 除了将class加载到内存中，同时还会对类进行后续的链接初始化过程。也就是说Class.forName() 执行了类的加载，验证，准备，解析和初始化的全过程。

> 注：Class.forName重载的另外一个方法，可以指定是否进行后续的链接、初始化过程，如果指定为false，则本质上与ClassLoader一样

测试类：

```
package com.test;

public class TestClassForName {
    private static int s = 1;
    static {
        System.out.println(s);
        System.out.println("static 代码块");
        s = 2;
        System.out.println(s);
    }
    private static int t = getANum();
    public TestClassForName() {
        System.out.println("调用了构造函数");
    }

    public static int getANum() {
        System.out.println("静态方法getANum执行了");
        return 2;
    }
}

```

测试方法：

```
public class MainTest {
    public static void main(String[] args) {
        System.out.println("-----开始forName-------");
        try {
            Class.forName("com.test.TestClassForName");
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("-----结束forName-------");

        System.out.println("-----开始forName init=false-------");
        try {
            Class.forName("com.test.TestClassForName",false,ClassLoader.getSystemClassLoader());
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("-----结束forName init=false-------");

        System.out.println("-----开始loadClass-------");
        try {
            ClassLoader.getSystemClassLoader().loadClass("com.test.TestClassForName");
        } catch (Exception e) {
            e.printStackTrace();
        }
        System.out.println("-----结束loadClass-------");
    }
}
```

测试结果：

```
-----开始forName-------
1
static 代码块
2
静态方法getANum执行了
-----结束forName-------
-----开始forName init=false-------
-----结束forName init=false-------
-----开始loadClass-------
-----结束loadClass-------
```

可以看到 当调用Class.forName 时静态代码块和类变量都已经执行并初始化了，而如果设置false时，则跟ClassLoader一样，没有执行任何东西

看一下Class.forName()方法的源码：


```
public static Class<?> forName(String className)  throws ClassNotFoundException {
     Class<?> caller = Reflection.getCallerClass();
     return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
}

public static Class<?> forName(String name, boolean initialize,
                                   ClassLoader loader) throws ClassNotFoundException
    {
        Class<?> caller = null;
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // Reflective call to get caller class is only needed if a security manager
            // is present.  Avoid the overhead of making this call otherwise.
            caller = Reflection.getCallerClass();
            if (sun.misc.VM.isSystemDomainLoader(loader)) {
                ClassLoader ccl = ClassLoader.getClassLoader(caller);
                if (!sun.misc.VM.isSystemDomainLoader(ccl)) {
                    sm.checkPermission(
                        SecurityConstants.GET_CLASSLOADER_PERMISSION);
                }
            }
        }
        return forName0(name, initialize, loader, caller);
    }

private static native Class<?> forName0(String name, boolean initialize,
                                            ClassLoader loader,
                                            Class<?> caller)  throws ClassNotFoundException;

```

发现最后都会调用forName0这个原生方法，关于该方法的说明，有兴趣的可以参考这篇文章：[Class.forName 发生了什么](https://zhuanlan.zhihu.com/p/205324628)


