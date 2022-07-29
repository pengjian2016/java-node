### 为什么要是用 MyBatis?

- MyBatis 是一个半自动的持久化框架。
- SQL与业务分离，功能边界清晰，当需要优化的时候，懂SQL的人比如DBA，直接看我们的SQL就够了，而不需要关注我们的业务代码。
- 因为是直接的SQL语句，不需要像Hibernate和JPA那样需要解析成数据库的SQL，性能也就相对较高。

###  #{} 和 ${} 的区别

- #{}：是以预编译的形式，将参数设置到 sql 语句中
- ${}：取出的值直接拼装在 sql 语句中；会有安全问题（SQL注入）

### MyBatis 中常用的标签有哪些？
- select 对应查询语句，可以包含属性parameterType（参数），resultType 或resultMap （返回值）
- insert 对应插入语句
- delete 对应删除语句
- update 对应更新语句
- if 通常用于WHERE语句、UPDATE语句、INSERT语句中，通过判断参数值来决定是否使用某个查询条件、判断是否更新某一个字段、判断是否插入某个字段的值
- foreach 主要用于构建in条件，可在sql中对集合进行迭代。也常用到批量删除、添加等操作中
- choose when 多条件判断 类似if else
- sql 当多种类型的查询语句的查询字段或者查询条件相同时，可以将其定义为常量，方便调用
- include 用于引用定义的常量

### 对于不支持主键自增的数据库（Oracle）如何在插入的时候获取主键值

oracle中没有自增长主键，而是sequence序列，插入的时候需要把sequence值放到主键中，这个时候可以使用selectKey标签获取sequence序列或者自定义主键：

```
<insert id="insertUser" parameterType="model.User">
		<selectKey keyProperty="id" resultType="int" order="BEFORE">
			select idseq.nextVal from dual
		</selectKey>
	insert into User
	(id, account,username, password) values
	(#{id}, #{account},#{username}, #{password})
</insert>

```

### MyBatis的工作原理是什么？

### MyBatis有哪些执行器

### MyBatis支持懒加载吗？原理是什么？



参考

https://www.zhihu.com/question/48910838
