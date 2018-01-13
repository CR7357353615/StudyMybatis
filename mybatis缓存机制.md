# Mybatis缓存机制
---
## 一级缓存
######   在同个SqlSession中，查询语句相同的sql会被缓存，但是一旦执行新增或更新或删除操作，缓存就会被清除。

### 代码：
#### BaseExecutor（query）
##### 在BaseExecutor中，query方法：这里能看出有一级缓存，不去查询，没有拿到一级缓存，需要重新查询。
```java
@SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
      // ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
      // if (closed) {
      //   throw new ExecutorException("Executor was closed.");
      // }
      // if (queryStack == 0 && ms.isFlushCacheRequired()) {
      //   clearLocalCache();
      // }
      // List<E> list;
      // try {
      //   queryStack++;
        //localCache是一级缓存
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        //如果拿到一级缓存
        if (list != null) {
          handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {	//没有拿到缓存，查询数据库
          list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
      // } finally {
      //   queryStack--;
      // }
      // if (queryStack == 0) {
      //   for (DeferredLoad deferredLoad : deferredLoads) {
      //     deferredLoad.load();
      //   }
      //   // issue #601
      //   deferredLoads.clear();
      //   if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      //     // issue #482
      //     clearLocalCache();
      //   }
      // }
      // return list;
  }
```

#### BaseExecutor(update)
##### BaseExecutor类update方法：这里可以看出，新增或更新数据时，一级缓存会被清除。

```java
@Override
public int update(MappedStatement ms, Object parameter) throws SQLException {
    // ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    // if (closed) {
    //     throw new ExecutorException("Executor was closed.");
    // }
    //新增或更新后，清除一级缓存
    clearLocalCache();
    // return doUpdate(ms, parameter);
}

@Override
public void clearLocalCache() {
    // if (!closed) {
    //   //清除了一级缓存
      localCache.clear();
      // localOutputParameterCache.clear();
    // }
}
```
##### 可以看出，在update和insert后，一级缓存会被清除

## 二级缓存
### 使用规则
##### 二级缓存的作用域是全局的，二级缓存在SqlSession关闭或提交之后才会生效。
##### 二级缓存与一级缓存不同，一级缓存不需要配置任何东西，二级缓存需要配置。
### 配置
##### 主要配置以下三个部分：
##### 1.	mybatis全局配置文件中的setting中的cacheEnabled需要为true(默认为true，不设置也行)
##### 2.	mapper配置文件中需要加入<cache>节点
##### 3.	mapper配置文件中的select节点需要加上属性useCache需要为true(默认为true，不设置也行)
##### 所以，只需要加一个&lt;cache/&gt;就可以了。
##### 使用二级缓存之后：查询数据的话，先从二级缓存中拿数据，如果没有的话，去一级缓存中拿，一级缓存也没有的话再查询数据库。有了数据之后在丢到TransactionalCache这个对象的entriesToAddOnCommit属性中。

### 为什么commit或close后，二级缓存才会生效？
#### 1.	DefaultSqlSession中的commit()
```java
@Override
public void commit(boolean force) {
    // try {
      executor.commit(isCommitOrRollbackRequired(force));
    //   dirty = false;
    // } catch (Exception e) {
    //   throw ExceptionFactory.wrapException("Error committing transaction.  Cause: " + e, e);
    // } finally {
    //   ErrorContext.instance().reset();
    // }
}
```
#### 2.	CachingExecutor的commit()
```java
@Override
  public void commit(boolean required) throws SQLException {
    // delegate.commit(required);
    tcm.commit();
  }
```
#### 3.	TransactionalCacheManager的commit()
```java
public void commit() {
    // for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    // }
}
```
#### 4.	TransactionalCache的commit()
```java
public void commit() {
    // if (clearOnCommit) {
      // delegate.clear();
    // }
    flushPendingEntries();
    // reset();
}

private void flushPendingEntries() {
    // for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    // }
    // for (Object entry : entriesMissedInCache) {
      // if (!entriesToAddOnCommit.containsKey(entry)) {
        // delegate.putObject(entry, null);
      // }
    // }
}
```
##### 这里会将数据put到cache中，也就是二级缓存中。

##### 对于close方法
#### 1.BaseExecutoor中的close()
```java
@Override
  public void close(boolean forceRollback) {
    // try {
    //   //issues #499, #524 and #573
    //   if (forceRollback) {
    //     tcm.rollback();
    //   } else {
        tcm.commit();
  //     }
  //   } finally {
  //     delegate.close(forceRollback);
  //   }
  // }
```
##### 可以看出close方法中调用了commit方法，所以也会将二级缓存数据放入cache中
