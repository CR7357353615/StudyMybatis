# Mybatis拦截器机制
---
## 总结
### 第一步：定义拦截器，实现Interceptor接口，xml配置文件中添加
```java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    System.out.println("拦截");
    return invocation.proceed();
  }
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
  public void setProperties(Properties properties) {
  }
}
```
```xml
<plugins>
    <plugin interceptor="org.format.mybatis.cache.interceptor.ExamplePlugin"></plugin>
</plugins>
```
### 第二步 创建执行器Executor时，调用pluginAll()方法
### 第三步 调用interceptor.plugin()方法。
### 第四步 调用Plugin.wrap()方法。
```java
public static Object wrap(Object target, Interceptor interceptor) {
    //获取需要拦截的类以及对应类需要拦截的方法
	Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    //根据目标实例target(这个target就是之前所说的MyBatis拦截器可以拦截的类，
    //Executor,ParameterHandler,ResultSetHandler,StatementHandler)和它的父类们，
    //返回signatureMap中含有target实现的接口数组。
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
}
```
##### 如果是需要拦截的类，则返回代理类，否则返回原类
### 第五步 如果是代理类，调用Plugin的invoke方法
##### 如果是需要拦截的类，调用Interceptor的interceptor()方法，如果不是，执行正常方法。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   try {
     Set<Method> methods = signatureMap.get(method.getDeclaringClass());
     if (methods != null && methods.contains(method)) {
       return interceptor.intercept(new Invocation(target, method, args));
     }
     return method.invoke(target, args);
   } catch (Exception e) {
     throw ExceptionUtil.unwrapThrowable(e);
   }
}
```
## 下面介绍拦截机制
##### Mybatis提供了一种叫插件（plugin）的功能，其实就是起到了拦截器的功能，允许在已经映射的语句执行过程中的某一点进行拦截调用。以下是允许使用插件拦截的方法：
类|方法
--|--
Executor|update(),query(),flushStatements(),commit(),rollback(),getTransaction(),close(),isClosed()
ParameterHandler|getParameterObject(),setParameters()
ResultSetHandler|handleResultSets(),handleOutputParameters
StatementHandler|prepare(),parameterize()batch(),update(),query()
#### 可以概括为：
##### 1.拦截执行器的方法
##### 2.拦截参数的处理
##### 3.拦截结果集的处理
##### 4.拦截sql语法构建的处理

##### Mybatis拦截器的接口定义
```java
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  Object plugin(Object target);

  void setProperties(Properties properties);

}
```
#### 拦截器的使用实例
##### 这个拦截器拦截的是Executor接口的update方法（也就是SqlSession的增，删，改操作）
```java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    return invocation.proceed();
  }
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
  public void setProperties(Properties properties) {
  }
}
```
```xml
<plugins>
    <plugin interceptor="org.format.mybatis.cache.interceptor.ExamplePlugin"></plugin>
</plugins>
```
### 源码分析
##### XMLConfigBuilder解析pluginElement方法：
```java
private void pluginElement(XNode parent) throws Exception {
    // if (parent != null) {
    //   for (XNode child : parent.getChildren()) {
    //     String interceptor = child.getStringAttribute("interceptor");
    //     Properties properties = child.getChildrenAsProperties();
    //     Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();
    //     interceptorInstance.setProperties(properties);
        configuration.addInterceptor(interceptorInstance);
    //   }
    // }
  }
```
##### Configuration的addInterceptor方法
```java
public void addInterceptor(Interceptor interceptor) {
    interceptorChain.addInterceptor(interceptor);
}
```
##### InterceptorChain类的addInterceptor方法
```java
private final List<Interceptor> interceptors = new ArrayList<Interceptor>();

public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
}
```
##### 可以看出所有的插件最终都会加入到interceptors这个List中
##### 继续看一下InterceptorChain这个类中的pluginAll方法,这个方法就是遍历interceptors中的每个interceptor执行其plugin方法。
```java
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
}
```

##### 下面就看一下为什么Executor，ParameterHandler，ResultSetHandler，StatementHandler会被拦截器拦截。
##### 以下是Configuration中的方法：
```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    // ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    // return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
      ResultHandler resultHandler, BoundSql boundSql) {
    // ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    // return resultSetHandler;
}

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    // return statementHandler;
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // executorType = executorType == null ? defaultExecutorType : executorType;
    // executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    // Executor executor;
    // if (ExecutorType.BATCH == executorType) {
    //   executor = new BatchExecutor(this, transaction);
    // } else if (ExecutorType.REUSE == executorType) {
    //   executor = new ReuseExecutor(this, transaction);
    // } else {
    //   executor = new SimpleExecutor(this, transaction);
    // }
    // if (cacheEnabled) {
    //   executor = new CachingExecutor(executor);
    // }
    executor = (Executor) interceptorChain.pluginAll(executor);
    // return executor;
}
```
##### 这些方法都是在生成handler或者Executor时就调用了intercrptorChain的pluginAll方法，调用了List中的interceptor的plugin方法。

### Executor，ParameterHandler，ResultSetHandler，StatementHandler的创建顺序。
##### 以上四个对象的创建顺序是Executor，ParameterHandler，ResultSetHandler，StatementHandler。我们可以看一段代码，这是BatchExecutor类中的doUpdate方法。
```java
@Override
public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    //获得Configuration
    final Configuration configuration = ms.getConfiguration();
    //创建StatementHandler。
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
  //   final BoundSql boundSql = handler.getBoundSql();
  //   final String sql = boundSql.getSql();
  //   final Statement stmt;
  //   if (sql.equals(currentSql) && ms.equals(currentStatement)) {
  //     int last = statementList.size() - 1;
  //     stmt = statementList.get(last);
  //     applyTransactionTimeout(stmt);
  //    handler.parameterize(stmt);//fix Issues 322
  //     BatchResult batchResult = batchResultList.get(last);
  //     batchResult.addParameterObject(parameterObject);
  //   } else {
  //     Connection connection = getConnection(ms.getStatementLog());
  //     stmt = handler.prepare(connection, transaction.getTimeout());
  //     handler.parameterize(stmt);    //fix Issues 322
  //     currentSql = sql;
  //     currentStatement = ms;
  //     statementList.add(stmt);
  //     batchResultList.add(new BatchResult(ms, sql, parameterObject));
  //   }
  // // handler.parameterize(stmt);
  //   handler.batch(stmt);
  //   return BATCH_UPDATE_RETURN_VALUE;
}
```
##### 从上文可以看出，在Executor的doUpdate方法中，创建了StatementHandler，那么为什么说ParameterHandler和ResultSetHandler比StatementHandler早创建呢？下面我们就看一下Configuration的newStatementHandler方法。
```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    // statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    // return statementHandler;
  }
```
##### 我们发现这不就是刚才看到的执行拦截器的那个方法么，但是这次我们的关注点放在RoutingStatementHandler这个方法上。我们看一下RoutingStatementHandler类的构造方法。
```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    // switch (ms.getStatementType()) {
      // case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        // break;
      // case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        // break;
      // case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        // break;
      // default:
        // throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

}
```
##### 可以看出有三种类型的StatementHandler，我们随意看一个，比如SimpleStatementHandler的构造方法。
```java
public SimpleStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    super(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
}
```
##### 可以发现SimpleStatementHandler，PreparedStatementHandler，CallableStatementHandler的构造方法都是调用了它们的父类BaseStatementHandler的构造方法，我们看一下这个构造方法：
```java
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // this.configuration = mappedStatement.getConfiguration();
    // this.executor = executor;
    // this.mappedStatement = mappedStatement;
    // this.rowBounds = rowBounds;
    //
    // this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    // this.objectFactory = configuration.getObjectFactory();
    //
    // if (boundSql == null) { // issue #435, get the key before calculating the statement
    //   generateKeys(parameterObject);
    //   boundSql = mappedStatement.getBoundSql(parameterObject);
    // }
    //
    // this.boundSql = boundSql;

    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}
```
##### 谜底终于揭开，就是在构建StatementHandler的时候创建了ParameterHandler和ResultSetHandler。

## mybatis拦截器源码解析
##### 要了解mybatis的拦截机制，需要了解Java反射和动态代理的知识。第一步要从Configuration的newExecutor方法开始讲起。
##### 示例：
```java
@Intercepts({@Signature(
  type= Executor.class,
  method = "update",
  args = {MappedStatement.class,Object.class})})
public class ExamplePlugin implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    System.out.println("拦截");
    return invocation.proceed();
  }
  public Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }
  public void setProperties(Properties properties) {
  }
}
```
```xml
<plugins>
    <plugin interceptor="org.format.mybatis.cache.interceptor.ExamplePlugin"></plugin>
</plugins>
```
### 1.创建执行器Executor
##### 前面说到了，在创建执行器时，会调用interceptorChain的pluginAll方法，其实就是调用了所有拦截器的plugin方法。
```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    // executorType = executorType == null ? defaultExecutorType : executorType;
    // executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    // Executor executor;
    // if (ExecutorType.BATCH == executorType) {
    //   executor = new BatchExecutor(this, transaction);
    // } else if (ExecutorType.REUSE == executorType) {
    //   executor = new ReuseExecutor(this, transaction);
    // } else {
    //   executor = new SimpleExecutor(this, transaction);
    // }
    // if (cacheEnabled) {
    //   executor = new CachingExecutor(executor);
    // }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```
##### plugin方法如下：
```java
public Object plugin(Object target) {
  return Plugin.wrap(target, this);
}
```
##### 下面看一下Plugin的wrap方法，先进入Plugin类
```java
public class Plugin implements InvocationHandler
```
##### 可以看出这个类实现了InvocationHandler，所以会有一个方法invoke
```java
@Override
 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   try {
     Set<Method> methods = signatureMap.get(method.getDeclaringClass());
     if (methods != null && methods.contains(method)) {
       return interceptor.intercept(new Invocation(target, method, args));
     }
     return method.invoke(target, args);
   } catch (Exception e) {
     throw ExceptionUtil.unwrapThrowable(e);
   }
 }
```
##### 那什么时候会执行这个invoke方法呢？就是代理对象执行方法时。我们现在看一下刚才调用的wrap方法。
```java
public static Object wrap(Object target, Interceptor interceptor) {
    //获取需要拦截的类以及对应类需要拦截的方法
	Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    //根据目标实例target(这个target就是之前所说的MyBatis拦截器可以拦截的类，
    //Executor,ParameterHandler,ResultSetHandler,StatementHandler)和它的父类们，
    //返回signatureMap中含有target实现的接口数组。
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```
##### 继续进入getAllInterfaces()方法
```java
private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    //获取注解
	Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    //获取签名数组
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
    	//反射获得需要拦截的方法
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
}
```
##### 这里就是获取注解中的需要拦截的类和方法。
##### 再看一下getAllInterfaces()方法,这个方法的作用是：根据目标实例target(这个target就是之前所说的MyBatis拦截器可以拦截的类，Executor,ParameterHandler,ResultSetHandler,StatementHandler)和它的父类们，返回signatureMap中含有target实现的接口数组。
```java
private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
}
```
##### 下面是wrap方法的片段
```java
if (interfaces.length > 0) {
  return Proxy.newProxyInstance(
      type.getClassLoader(),
      interfaces,
      new Plugin(target, interceptor, signatureMap));
}
return target;
```
##### 这里可以看出，如果是有被拦截接口的，那么，会返回一个代理类，这样Executor执行器在执行方法时会执行Plugin的invoke方法。再看一下invoke方法
```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   try {
     Set<Method> methods = signatureMap.get(method.getDeclaringClass());
     if (methods != null && methods.contains(method)) {
       return interceptor.intercept(new Invocation(target, method, args));
     }
     return method.invoke(target, args);
   } catch (Exception e) {
     throw ExceptionUtil.unwrapThrowable(e);
   }
}
```
##### 这里可以看出，当执行到需要拦截的方法时，会执行intercept方法，也就是我们自己写的这部分
```java
public Object intercept(Invocation invocation) throws Throwable {
  System.out.println("拦截");
  return invocation.proceed();
}
```
##### 可以看到除了执行我们自己添加的逻辑外，还执行了invocation的proceed方法，我们看一下Invocation类的proceed方法：
```java
public Object proceed() throws InvocationTargetException, IllegalAccessException {
    return method.invoke(target, args);
}
```
##### 其实这里还是执行原方法的内容。
##### 到这里，Mybatis的拦截原理解析完成了。
