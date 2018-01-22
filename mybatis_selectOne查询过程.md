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

## 源码分析
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

## 步骤1

### 方法调用流程图如下：</br>
selectList()  # 查询，返回list </br>
　　->configuration.getMappedStatement(statement) # 获取MappedStatement节点</br>
　　　　->buildAllStatements() #创建所有的属性</br>
　　　　　　(1)->ResultMapResolver resolve() # 添加ResultMap </br>
　　　　　　　　　(5)->MapperBuilderAssistant addResultMap()  #添加 resultMap</br>
　　　　　　　　　　　(6)->Configuration addResultMap() # 将resultMap加入configuration中</br>
　　　　　　(2)->CacheRefResolver resolveCacheRef() # 使用引用缓存 </br>
　　　　　　(3)->XMLStatementBuilder parseStatementNode() # 解析节点</br>
　　　　　　(4)->MethodResolver resolve() </br>
　　->executor.query() # 执行查询</br>
　　->ErrorContext.instance().reset() #异常上下文重置</br>

### 方法(1)
我们先讲解方法(1)ResultMapResolver的resolve()方法
```java
/**
  * @param id resultMap的id
  * @param type resultMap的类型
  * @param extend 父resultMap
  * @param discriminator resultMap的鉴别器
  * @param resultMappings 映射节点
  * @param autoMapping 是否自动映射
  * @return
*/
public ResultMap resolve() {
  return assistant.addResultMap(this.id, this.type, this.extend, this.discriminator, this.resultMappings, this.autoMapping);
}
```
这里的属性要关注一下extend和discriminator。

* extend是resultMap继承的父resultMap。
* discriminator是resultMap的鉴别器。

鉴别器介绍起来比较抽象，看了下面这段代码就能很好的理解：
```xml
<select id="getEmployee" parameterType="int" resultMap="employeeMap">  
    select id, emp_name as empName, sex from t_employee where id =#{id}  
</select>  

<!-- <resultMap id="employeeMap" type="com.ykzhen2015.csdn.pojo.Employee">  
  <id property="id" column="id" />  
  <result property="empName" column="emp_name" />  
  <result property="sex" column="sex" />  
  <association property="employeeCard" column="id"  
      select="com.ykzhen2015.csdn.mapper.EmployeeCardMapper.getEmployeeCardByEmpId" />  
  <collection property="projectList" column="id"  
      select="com.ykzhen2015.csdn.mapper.ProjectMapper.findProjectByEmpId" /> -->
  <discriminator javaType="int" column="sex">  
      <case value="1" resultMap="maleEmployeeMap" />  
      <case value="2" resultMap="femaleEmployeeMap" />  
  </discriminator>  
<!-- </resultMap> -->  

<resultMap id="maleEmployeeMap" type="com.ykzhen2015.csdn.pojo.MaleEmployee" extends="employeeMap">  
    <collection property="prostateList" select="com.ykzhen2015.csdn.mapper.MaleEmployeeMapper.findProstateList" column="id" />  
</resultMap>  

<resultMap id="femaleEmployeeMap" type="com.ykzhen2015.csdn.pojo.FemaleEmployee" extends="employeeMap">  
    <collection property="uterusList" select="com.ykzhen2015.csdn.mapper.FemaleEmployeeMapper.findUterusList" column="id" />  
</resultMap>
```
其实discriminator就是在resultMap中根据某属性的值动态选择另一个resultMap。

下面讲解一下方法(5) addResultMap():
```java
/**
 * 添加ResultMap
 * @param id namespace
 * @param type 类型
 * @param extend 继承自
 * @param discriminator 鉴别器
 * @param resultMappings 结果映射集
 * @param autoMapping 自动映射
 * @return
 */
public ResultMap addResultMap(String id, Class<?> type, String extend, Discriminator discriminator, List<ResultMapping> resultMappings, Boolean autoMapping) {
  // 获取当前命名空间
  // id = applyCurrentNamespace(id, false);

  // 获取继承的namespace
  // extend = applyCurrentNamespace(extend, true);

  // 如果resultMap有继承的resultMap
  if (extend != null) {
    // 如果继承的resultMap不存在，抛异常
    // if (!configuration.hasResultMap(extend)) {
      // throw new IncompleteElementException("Could not find a parent resultmap with id '" + extend + "'");
    // }

    // 获取父resultMap
    // ResultMap resultMap = configuration.getResultMap(extend);

    // 获取父resultMap的映射信息ResultMapping
    List<ResultMapping> extendedResultMappings = new ArrayList<ResultMapping>(resultMap.getResultMappings());
    // 将父resultMap中与子resultMap相同的resultMapping删除掉（为了实现子resultMap覆盖父resultMap）
    extendedResultMappings.removeAll(resultMappings);

    // Remove parent constructor if this resultMap declares a constructor.
    // 如果这个子resultMap中包含constructor，删除父resultMap的constructor
    boolean declaresConstructor = false;
    for (ResultMapping resultMapping : resultMappings) {
      if (resultMapping.getFlags().contains(ResultFlag.CONSTRUCTOR)) {
        declaresConstructor = true;
        break;
      }
    }
    if (declaresConstructor) {
      Iterator<ResultMapping> extendedResultMappingsIter = extendedResultMappings.iterator();
      while (extendedResultMappingsIter.hasNext()) {
        if (extendedResultMappingsIter.next().getFlags().contains(ResultFlag.CONSTRUCTOR)) {
          extendedResultMappingsIter.remove();
        }
      }
    }

    // 将父resultMap的resultMapping加入到子resultMap中
    resultMappings.addAll(extendedResultMappings);
  }

  // 如果没有父resultMap，
  ResultMap resultMap = new ResultMap.Builder(configuration, id, type, resultMappings, autoMapping)
      .discriminator(discriminator)
      .build();
  configuration.addResultMap(resultMap);
  return resultMap;
}
```
下面是方法(6)
```java
public void addResultMap(ResultMap rm) {
  resultMaps.put(rm.getId(), rm);
  // 从自身出发，判断该resultMap的discriminator是否嵌套ResultMap。
  checkLocallyForDiscriminatedNestedResultMaps(rm);
  // 从全局出发，判断系统中的resultMap的discriminator中是否包含自己。
  checkGloballyForDiscriminatedNestedResultMaps(rm);
}
```
这段代码主要做了以下几件事：
* 1.如果resultMap有父resultMap
* * 1.1.将父resultMap和本resultMap中相同的resultMappings删掉（为了用子resultMap中的resultMappings覆盖父resultMap）
* * 1.2.如果这个子resultMap中包含constructor，删除父resultMap的constructor
* * 1.3.将父resultMap的resultMapping加入到子resultMap中
* 2.如果resultMap没有父resultMap
* * 2.1.创建resultMap
* * 2.2.configuration加入resultMap
* * * 2.2.1.将resultMap放入configuration的resultMaps中
* * * 2.2.2.从自身出发，判断该resultMap的discriminator是否嵌套ResultMap。
* * * 2.2.3.从全局出发，判断系统中的resultMap的discriminator中是否包含自己。

### 方法(2)
下面看一下方法(2) CacheRefResolver的resolveCacheRef()

这个方法比较简单，思路就是从configuration中获取
```java
public Cache resolveCacheRef() {
  return assistant.useCacheRef(cacheRefNamespace);
}

public Cache useCacheRef(String namespace) {
	if (namespace == null) {
		throw new BuilderException(
				"cache-ref element requires a namespace attribute.");
	}
	try {
		// 未解决的缓存引用标志置为true
		unresolvedCacheRef = true;
		// 从configuration中根据namespace获取缓存
		Cache cache = configuration.getCache(namespace);
		if (cache == null) {
			throw new IncompleteElementException("No cache for namespace '"
					+ namespace + "' could be found.");
		}
		// 如果缓存引用不为空，赋给当前缓存
		currentCache = cache;
		// 未解决缓存引用标志置为false
		unresolvedCacheRef = false;
		return cache;
	} catch (IllegalArgumentException e) {
		throw new IncompleteElementException("No cache for namespace '"
				+ namespace + "' could be found.", e);
	}
}
```
### 方法(3)
解析了< select>中的所有属性  内容较多，难点在对< include>,< selectKey>两个节点的解析。
### 方法(4)
