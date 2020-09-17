| 序号 | 内容           | 最后修改时间              |
|----|--------------|---------------------|
| 1  | 增加java 8 新特性 | 2020年09月17日15:50:33 |

<br>

### java 8 新特性

当面试官让我说几个java 8 的新特性，我巴拉巴拉把知道的都说了，然而，面试官接着问，stream里面如果按照分类过滤怎么做呢？map()结果是什么？
”嘀，扫码成功，哎呀，地铁里面的空调真不错啊，真不错啊“。

果然，只做到了解是不行的啊，其实东西都不是难点，关键是不使用的话很难记住这些东西，我们这样的小公司大多数都是靠数据库操作逻辑的人，显然这些新东西没有勇武之地，虽然我也在尽量使用Optional，foreach这些，但还是太简单了，大公司大多都注重代码简洁，人家一行代码就搞定的东西，你搞个几十行，这就是差距啊。


#### 1.接口默认方法
通过default 关键字可以为接口中的方法提供默认实现：

```
public interface MyJava8Service {
    default int getNewNum(int num) {
        return 2 * num;
    }
}
```
测试

```
public static void main(String[] args) {
    MyJava8Service java8Service = new MyJava8Service() {};
    System.out.println("新特性，接口默认方法："+java8Service.getNewNum(4));
}

输出结果：
新特性，接口默认方法：8

```
那么问题来了，default关键字是java 8 引入的吗？还好我立马想到了switch语句里面的default 所以显然不是新引入的，面试官给我爬！

#### 2.lambda表达式

