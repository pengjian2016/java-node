| 序号 | 内容           | 最后修改时间              |
|----|--------------|---------------------|
| 1  | 增加java 8 新特性 | 2020年09月17日21:50:33 |

<br>

### java 8 新特性

当面试官让我说几个java 8 的新特性，我巴拉巴拉把知道的都说了，然而，面试官接着问，stream里面如果按照分类过滤怎么做呢？map()是什么？
”嘀，扫码成功，哎呀，地铁里面的空调真不错啊，真不错“。

果然，只做到了解是不行的啊，其实这些都不是难点，关键是不使用的话很难记住这些东西，像我们这样的小公司大多数都是靠数据库操作逻辑的人，显然这些新东西没有用武之地，虽然我也在尽量使用Optional，foreach这些，但还是太简单了，大公司大多都注重代码简洁，人家一行代码就搞定的东西，你搞个几十行，这就是差距啊。


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

#### 2.lambda表达式
它允许把函数当做参数来使用，是面向函数式编程的思想，一定程度上可以使代码看起来更加简洁

```
 public interface TestForLambda{
   void sendMessage();
}

public interface TestForLambda2{
    String getMessage(String msg);
}

public static void main(String[] args){
    TestForLambda testForLambda = ()-> {
        System.out.println("简单实现了接口方法");
        System.out.println("do something");
    };
    testForLambda.sendMessage();

    TestForLambda2 testForLambda2 = (a)->"hello "+a;
    System.out.println(testForLambda2.getMessage("javajov"));
}

测试结果:

简单实现了接口方法
do something
hello java jov

```

#### 3.Optional

用来处理空值的类，如果你使用过springboot的jpa 那么你肯定对Optional不陌生，通过ID从数据库查询记录的时候返回的就是Optional<Entity>。如果你项目中使用过kotlin，我相信各位对它在空值变量的限制上一定映像深刻。可见现在对null的处理开始重视起来了。

测试代码
```
public static void main(String[] args) {
    System.out.println("=======测试有值的情况begin========");
    Optional<String> ret = Optional.of("javajov");
    ret.ifPresent(data -> System.out.println("有值的时候做点事情：" + data));
    System.out.println("获取值1：" + ret.orElse("如果为空则返回else提供的内容"));
    System.out.println("获取值2：" + ret.get());
    System.out.println("=======测试有值的情况end========");

    System.out.println("=======测试NULL的情况begin========");
    String bean = null;
    Optional<String> ret2 = Optional.ofNullable(bean);
    ret2.ifPresent(data -> System.out.println("有值的时候做点事情：" + data));

    System.out.println("获取值1：" + ret2.orElse("如果为空则返回else提供的内容"));
    System.out.println("获取值2：" + ret2.get());
    System.out.println("=======测试NULL的情况end========");
}

输出
=======测试有值的情况begin========
有值的时候做点事情：javajov
获取值1：javajov
获取值2：javajov
=======测试有值的情况end========
=======测试NULL的情况begin========
获取值1：如果为空则返回else提供的内容
Exception in thread "main" java.util.NoSuchElementException: No value present

```

#### 4.stream

它可以指定你希望对集合进行的操作，可以执行非常复杂的查找、过滤和映射数据等操作。


```
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("cc");
    list.add("bb");
    list.add("dd");
    list.add("aa");
    list.add("111");
    list.add("222");
    list.add("333");
    //filter过滤，假如list里面存放的是实体类，使用filter可以按照你想要的方式进行过滤
    List<String> newlist = list.stream().filter(bean -> bean.length() == 3).collect(Collectors.toList());
    System.out.println("filter过滤之后:");
    newlist.forEach(bean -> System.out.print(bean + ","));
    System.out.println();
    System.out.println("list 原来的内容并不会变化如下：");
    list.forEach(bean -> System.out.print(bean + ","));
    System.out.println();

    //使用sorted进行排序
    List<String> sortedList = list.stream().sorted((a, b) -> a.compareTo(b)).collect(Collectors.toList());
    System.out.println("sorted排序之后:");
    sortedList.forEach(bean -> System.out.print(bean + ","));
    System.out.println();
    System.out.println("list 原来的内容并不会变化如下：");
    list.forEach(bean -> System.out.print(bean + ","));
    System.out.println();

    //使用map进行转换
    List<String> mapList = list.stream().map(bean -> bean.toUpperCase()+"hello").collect(Collectors.toList());
    System.out.println("map转换之后：");
    mapList.forEach(bean -> System.out.print(bean + ","));
    System.out.println();
    System.out.println("list 原来的内容并不会变化如下：");
    list.forEach(bean -> System.out.print(bean + ","));
    System.out.println();
}

输出：

filter过滤之后:
111,222,333,
list 原来的内容并不会变化如下：
cc,bb,dd,aa,111,222,333,
sorted排序之后:
111,222,333,aa,bb,cc,dd,
list 原来的内容并不会变化如下：
cc,bb,dd,aa,111,222,333,
map转换之后：
CChello,BBhello,DDhello,AAhello,111hello,222hello,333hello,
list 原来的内容并不会变化如下：
cc,bb,dd,aa,111,222,333,
```


其他还有很多方法，大家可以去看一下Stream这个类，一定要把各个方法都尝试一遍，我这里面只是把面试中常问到的几个列了一下，但是并不代表其他不会被问到，作为一个以高级工程师、架构师为目标的面试者，这些东西都掌握不是应该的吗。
