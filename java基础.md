| 序号 | 内容                            | 修改时间                |
|----|-------------------------------|---------------------|
| 1  | 增加classloader和class.forname区别 | 2020年09月16日18:25:05 |



#### ClassLoader 和 Class.forName() 他们都能加载类，有什么不同？
ClassLoader只是将class文件加载到内存中，不进行其他操作
Class.forName() 除了将class文件加载到内存中，同时还会对类进行解释（这里待确认），执行类中的static块（静态变量是否会初始化？）



