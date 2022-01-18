### 介绍一下hibernate的一级缓存和二级缓存

一级缓存是session级别的（注：hibernate中的session是指服务器与数据库建立一个通信对象，非java servlet中认为的session），将某次查询操作的结果缓存在一级缓存中，如果短时间内同一个session又做了相同的查询操作，那么它将会直接从缓存中拿去结果，而不是查询数据库。

二级缓存是SessionFactory级别的缓存，如果同一个sessionFactory 创建的某个session执行了相同的操作，hibernate就会从二级缓存中拿结果，而不会再去连接数据库。
