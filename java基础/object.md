## 一个Java对象占用多大内存？

java中对象由对象头、实例数据、填充数据组成，虚拟机要求对象必须是8的倍数

对象头包括：markword和类型指针

以空对象Object为例

| 内容       | 32位    | 64位                  |
|----------|--------|----------------------|
| markword | 4byte   | 8byte                 |
| 类型指针     | 4byte   | 未开启压缩为8bit，开启压缩为4byte |
| 实例数据     | 0      | 0                    |
| 填充数据（将最后的大小填充为8的倍数）     | 0~7byte | 0~7byte               |
| 总计  |  8byte  |    开启压缩后，则8+4=12，对齐填充为8的倍数，最后为 16byte  |

实例数据部分，根据类中的成员变量类型不同而不同

| 类型      | 占用空间(byte) |
|---------|------------|
| boolean | 1          |
| byte    | 1          |
| short   | 2          |
| char    | 2          |
| int     | 4          |
| float |  4 |
| long  | 8 |
| double | 8 |

测试，工具org.openjdk.jol:jol-core:0.14
```
public static void main(String[] args){
    ClassLayout classLayout = ClassLayout.parseInstance(new Object());
    System.out.println(classLayout.toPrintable());
}

结果：

java.lang.Object object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           e5 01 00 20 (11100101 00000001 00000000 00100000) (536871397)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```
测试2
```
public class TestObject {
    private boolean flag = true;
    private int s1 = Integer.MAX_VALUE;
    private double s2 = 0.01;
    private Integer s3 = 9999;

    public static void main(String[] args){
        ClassLayout classLayout = ClassLayout.parseInstance(new TestObject());
        System.out.println(classLayout.toPrintable());
    }
}

结果
com.jov.TestObject object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0     4                     (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4                     (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4                     (object header)                           05 c1 00 20 (00000101 11000001 00000000 00100000) (536920325)
     12     4                 int TestObject.s1                             2147483647
     16     8              double TestObject.s2                             0.01
     24     1             boolean TestObject.flag                           true
     25     3                     (alignment/padding gap)                  
     28     4   java.lang.Integer TestObject.s3                             9999
Instance size: 32 bytes
Space losses: 3 bytes internal + 0 bytes external = 3 bytes total

```

参考
[一个Java对象占用多大内存](https://www.cnblogs.com/rickiyang/p/14206724.html)