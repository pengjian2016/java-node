| 序号 | 内容                            | 修改时间                |
|----|-------------------------------|---------------------|
| 1  | 增加classloader和class.forname区别 | 2020年09月16日18:25:05 |
<br>

#### ClassLoader 和 Class.forName() 他们都能加载类，有什么不同？

ClassLoader只是将class文件加载到内存中，不进行其他操作，即只执行了类的加载阶段

Class.forName() 除了将class文件加载到内存中，同时还会对类进行解释，执行类中的static块，并且进行静态变量的初始化。也就是说Class.forName() 执行了类的加载，验证，准备，解析和初始化的过程。

> 注：Class.forName重载的另外一个方法，可以指定是否进行初始化，如果指定为不初始化，则本质上与ClassLoader一样

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

发现最后都会调用forName0这个原生方法，内部具体如何实现的就不得而知了，其中initialize 参数应该就是控制是否只加载类还是会会执行后续的步骤


