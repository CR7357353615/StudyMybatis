# Mybatis_selectOne查询过程
---
## 背景
在不考虑与Spring集成的情况下,使用MyBatis执行数据库操作的代码如下：
```java
// String resource = "org/mybatis/example/mybatis-config.xml";
// InputStream inputStream = Resources.getResourceAsStream(resource);
// SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
// SqlSession session = sqlSessionFactory.openSession();
// try {
  Blog blog = session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
// } finally {
  // session.close();
}
```
这里已经获取到了SqlSessionFactory，并且生成了session，调用session的selectOne方法。本文就研究一下SelectOne方法的源码。

首先，SqlSession的默认实现是DefaultSqlSession，下面是DefaultSqlSession的selectOne()方法。
```java
@Override
	public <T> T selectOne(String statement, Object parameter) {
		// Popular vote was to return null on 0 results and throw exception on
		// too many.
		// 获取结果
		List<T> list = this.<T> selectList(statement, parameter);
		// 如果只查询出一个值，那么就返回这个值。
		if (list.size() == 1) {
			return list.get(0);
		// 如果返回了多个值，那么抛异常
		} else if (list.size() > 1) {
			// throw new TooManyResultsException(
					// "Expected one result (or null) to be returned by selectOne(), but found: "
							// + list.size());
		// } else {
			// return null;
		// }
	}
```
这里调用selectList方法，可以看出查询最终都调用的是selectList方法，只是selectOne方法在外层加了一层判断。下面是selectList方法的源码：
```java
@Override
	public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
		// try {
			// 得到这个xml中的节点
			MappedStatement ms = configuration.getMappedStatement(statement);
			// 执行器执行query操作
			return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
		// } catch (Exception e) {
			// throw ExceptionFactory.wrapException(
					// "Error querying database.  Cause: " + e, e);
		// } finally {
			// 异常上下文重置
			ErrorContext.instance().reset();
		}
}
```
这个方法中其实是执行了三个步骤：
* 1.通过statement从configuration中获取到相应的MappedStatement节点（对应的xml节点）
* 2.使用执行器executor执行sql查询内容
* 3.将异常上下文ErrorContext重置

方法调用流程图如下：
selectList()  # 查询，返回list </br>
　　->configuration.getMappedStatement(statement) # 获取MappedStatement节点</br>
　　->executor.query() # 执行查询</br>
　　->ErrorContext.instance().reset() #异常上下文重置
