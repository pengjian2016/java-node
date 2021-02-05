### 1.类文件结构

根据 Java 虚拟机规范，类文件由单个 ClassFile 结构组成：

```
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    field_info     fields[fields_count];//一个类会可以有多个字段
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
}
```
参考：[类文件结构](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/%E7%B1%BB%E6%96%87%E4%BB%B6%E7%BB%93%E6%9E%84) 更加详细的内容都在这里了。

在这里主要针对常量池做一下说明，在jvm内存结构那一章节中，在方法区的部分有提到常量池相关的内容，分为class常量池、运行时常量池、全局字符串池（全局常量池），它们之间到底是什么样的关系呢？

全局字符串池：

```
全局字符串池里的内容是在类加载完成，经过验证，准备阶段之后在堆中生成字符串对象实例，然后将该字符串对象实例的引用值存到string pool中（记住：string pool中存的是引用值而不是具体的实例对象，具体的实例对象是在堆中开辟的一块空间存放的。）。 在HotSpot VM里实现的string pool功能的是一个StringTable类，它是一个哈希表，里面存的是驻留字符串(也就是我们常说的用双引号括起来的)的引用（而不是驻留字符串实例本身），也就是说在堆中的某些字符串实例被这个StringTable引用之后就等同被赋予了”驻留字符串”的身份。这个StringTable在每个HotSpot VM的实例只有一份，被所有的类共享。

```

class文件常量池:
```

主要存放两大常量：字面量和符号引用。

字面量：如文本字符串、被声明为final的常量值等

符号引用： 是一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义地定位到目标即可。

```

![class文件常量池](https://images.gitee.com/uploads/images/2021/0205/104330_fc5e196c_8076629.png "class文件常量池.png")

运行时常量池:

```
当类加载到内存中后，jvm会将class常量池中的内容存放到运行时常量池中，而经过解析（resolve）之后，也就是把符号引用替换为直接引用，解析的过程会去查询全局字符串池，也就是上面所说的StringTable，以保证运行时常量池所引用的字符串与全局字符串池中所引用的是一致的。

```

![运行时常量池](https://images.gitee.com/uploads/images/2021/0205/111844_c81c03ac_8076629.png "运行时常量池.png")


### 2.类加载过程

![类加载过程](https://images.gitee.com/uploads/images/2021/0205/112028_e852edbe_8076629.png "类加载过程.png")

类的加载过程主要分为：加载、连接、初始化，其中连接又分为：验证、准备、解析

#### 2.1 加载

- 通过全类名获取定义此类的二进制字节流

- 将字节流所代表的静态存储结构转换为方法区的运行时数据结构

- 在内存中生成一个代表该类的 Class 对象,作为方法区这些数据的访问入口

加载.class文件的方式：从本地系统中直接加载、通过网络下载.class文件、从zip，jar等归档文件中加载.class文件、从专有数据库中提取.class文件、将Java源文件动态编译为.class文件等。

#### 2.2 验证


### 3.类加载器（双亲委派模型）

### 4.对象创建过程