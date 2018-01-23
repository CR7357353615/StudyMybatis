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
在了解方法(4)前，需要了解一下注解方式动态执行sql语句，下面是一个例子。
```java
public interface EmployeeMapper {
  // 动态查询
  @SelectProvider(type=EmployeeDynaSqlProvider.class,method="selectWhitParam")
  List<Employee> selectWhitParam(Map<String, Object> param);
}

public class EmployeeDynaSqlProvider {
  public String selectWhitParam(Map<String, Object> param){
    return new SQL(){
        {
            SELECT("*");
            FROM("tb_employee");
            if(param.get("id") != null){
                WHERE(" id = #{id} ");
            }
            if(param.get("loginname") != null){
                WHERE(" loginname = #{loginname} ");
            }
            if(param.get("password") != null){
                WHERE("password = #{password}");
            }
            if(param.get("name")!= null){
                WHERE("name = #{name}");
            }
            if(param.get("sex")!= null){
                WHERE("sex = #{sex}");
            }
            if(param.get("age")!= null){
                WHERE("age = #{age}");
            }
            if(param.get("phone")!= null){
                WHERE("phone = #{phone}");
            }
            if(param.get("sal")!= null){
                WHERE("sal = #{sal}");
            }
            if(param.get("state")!= null){
                WHERE("state = #{state}");
            }
        }
    }.toString();
  }    
}
```
上面这个例子就是在注解中指定实现类和method，根据传入的值动态执行。下面看一下源代码：
```java
public void resolve() {
  annotationBuilder.parseStatement(method);
}
```
```java
/**
 * 解析statement
 * @param method
 */
void parseStatement(Method method) {
	// 获取方法参数的类型
	Class<?> parameterTypeClass = getParameterType(method);
	// 获取language驱动
	LanguageDriver languageDriver = getLanguageDriver(method);
	// 从注解中获取sqlSource
	SqlSource sqlSource = getSqlSourceFromAnnotations(method, parameterTypeClass, languageDriver);
	// 如果sqlSoruce不为空
	if (sqlSource != null) {
		// 获取方法中的设置注解
		Options options = method.getAnnotation(Options.class);
		// mappedStatementId为接口名.方法名
		final String mappedStatementId = type.getName() + "." + method.getName();
		Integer fetchSize = null;
		Integer timeout = null;
		// statementType默认为Prepared
		StatementType statementType = StatementType.PREPARED;
		// resultSetType默认为foreard_only
		ResultSetType resultSetType = ResultSetType.FORWARD_ONLY;
		// 获取执行类型
		SqlCommandType sqlCommandType = getSqlCommandType(method);
		// 是否是select
		boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
		// 如果是select，不刷新缓存，否则刷新缓存
		boolean flushCache = !isSelect;
		// 如果是select，使用二级缓存，否则不使用
		boolean useCache = isSelect;

		KeyGenerator keyGenerator;
		String keyProperty = "id";
		String keyColumn = null;
		// 如果是Insert或是Update
		if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
			// first check for SelectKey annotation - that overrides
			// everything else
			// 获取SelectKey注解
			SelectKey selectKey = method.getAnnotation(SelectKey.class);
			// 如果不为空
			if (selectKey != null) {
				// 处理selectKey
				keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
				// 获取selectKey的keyProperty
				keyProperty = selectKey.keyProperty();
		    // 如果设置项为空
			} else if (options == null) {
				// 主键生成器
				keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
			} else {
				// 如果options不为空，则取options中的useGeneratedKeys进行判断
				keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
				keyProperty = options.keyProperty();
				keyColumn = options.keyColumn();
			}
		} else {
			keyGenerator = NoKeyGenerator.INSTANCE;
		}

		// 如果options不为空
		if (options != null) {
			// 判断options中是否要刷新缓存
			if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
				flushCache = true;
			} else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
				flushCache = false;
			}
			useCache = options.useCache();
			fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; // issue #348
			timeout = options.timeout() > -1 ? options.timeout() : null;
			statementType = options.statementType();
			resultSetType = options.resultSetType();
		}

		String resultMapId = null;
		// 获取方法的ResultMap注解
		ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
		// 如果注解不为空
		if (resultMapAnnotation != null) {
			String[] resultMaps = resultMapAnnotation.value();
			StringBuilder sb = new StringBuilder();
			for (String resultMap : resultMaps) {
				if (sb.length() > 0) {
					sb.append(",");
				}
				sb.append(resultMap);
			}
			resultMapId = sb.toString();
		}
		// 如果注解为空，并且是select
		else if (isSelect) {
			// 解析ResultMap
			resultMapId = parseResultMap(method);
		}

		assistant.addMappedStatement(mappedStatementId, sqlSource,
				statementType, sqlCommandType, fetchSize, timeout,
				// ParameterMapID
				null, parameterTypeClass, resultMapId,
				getReturnType(method), resultSetType, flushCache, useCache,
				// TODO gcode issue #577
				false, keyGenerator, keyProperty, keyColumn,
				// DatabaseID
				null, languageDriver,
				// ResultSets
				options != null ? nullOrEmpty(options.resultSets()) : null);
	}
}
```

步骤1是解析节点的步骤，内容比较多，因为有xml的方式还有Annotation的方式，主要来讲就是解析xml或Annotation中的各种配置项。
## 步骤2
再回到selectList方法，继续SelectOne的第二个步骤，执行器的查询过程
```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
	try {
		// 解析得到这个节点
		// MappedStatement ms = configuration.getMappedStatement(statement);
		// 执行器执行query操作
		return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
	// } catch (Exception e) {
		// throw ExceptionFactory.wrapException(
				// "Error querying database.  Cause: " + e, e);
	// } finally {
		// 异常上下文重置
		// ErrorContext.instance().reset();
	}
}
```
wrapCollection()方法作用是：
* 1.如果object是一个容器类，向map中放入collection-object，并且如果object是一个list，向map中放入list-object
* 2.如果object是一个数组类型，向map中放入array-object

下面是BaseExecutor的query方法：
```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameter);
  CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
  return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}
```
* 1.第一步获取boundSql
* 2.生成一级缓存key
* 3.查询

主要看一下query方法
```java
@SuppressWarnings("unchecked")
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  if (queryStack == 0 && ms.isFlushCacheRequired()) {
    clearLocalCache();
  }
  List<E> list;
  try {
    queryStack++;
//      localCache是一级缓存
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    //如果拿到一级缓存
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {	//没有拿到缓存，查询数据库
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
  if (queryStack == 0) {
    for (DeferredLoad deferredLoad : deferredLoads) {
      deferredLoad.load();
    }
    // issue #601
    deferredLoads.clear();
    if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
      // issue #482
      clearLocalCache();
    }
  }
  return list;
}
```
最终返回的List就是结果List
