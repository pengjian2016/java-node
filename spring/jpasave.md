# JPA 中只有save 方法，它是如何决定是insert 还是update的呢？

看一下SimpleJpaRepository中save方法实现的源码：

```
        @Transactional
	@Override
	public <S extends T> S save(S entity) {
                // 如果是新的记录，则persist，否则merge
		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}

```
源码中首先进行isNew判断，根据主键id是否为空，来判断是要调用persist方法(可理解为insert)，还是merge方法(可理解为update)

对于id为空时，可能使用了自增长的id或者uuid这种，那么jpa会直接进行insert 插入；

而对于id不为空的，不管是新的数据（主键自己设置的）还是更新数据，都会先执行select查询，然后如果没有记录则会insert 插入，否则的话会update


