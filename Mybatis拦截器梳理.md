# Mybatis拦截器梳理
---
### Interceptor接口
```java
public interface Interceptor {
  Object intercept(Invocation invocation) throws Throwable;

  Object plugin(Object target);

  void setProperties(Properties properties);
}
```
### 实例
```java
@Intercepts(
  {
    @Signature(
      type = Executor.class,
      method = "update",
      args = {
        MappedStatement.class,Object.class
      }
    )
  }
)
public class PluginExample implements Interceptor {
  public Object intercept(Invocation invocation) throws Throwable {
    System.out.println("拦截");
    return invocation.proceed();
  }

  public Object plugin (Object target) {
    return Plugin.wrap(target, this);
  }

  public void setProperties(Properties properties) {

  }
}
```

### 1.插件放在哪里？
初始化的时候，插件都被放在interceptorChain的interceptors这个列表中。
### 2.哪里开始调用插件呢？
插件是对Executor，ParameterHandler，ResultSetHandler，StatementHandler有作用。因为在new这几个类的时候都调用了

executor = (Executor) interceptorChain.pluginAll(executor);

parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);

resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);

statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
### 3.pluginAll做了什么工作呢？
```java
public Object pluginAll(Object target) {
  for (Interceptor interceptor : interceptors) {
    target = interceptor.plugin(target);
  }
  return target;
}
```
就是遍历interceptors列表，调用各自的plugin方法。

### 4.plugin方法做了什么呢？
实现中执行如下操作
```java
return Plugin.wrap(target, this);
```

### 5.Plugin.wrap()做了什么？
Plugin类继承自InvocationHandler，主要有wrap方法和invoke方法。wrap方法获取target中是不是有SignatureMap中定义的需要拦截的接口，如果有就返回了一个代理类。这样在Executor，ParameterHandler，ResultSetHandler，StatementHandler执行某些方法时，会执行Plugin的invoke方法。

### 6.invoke方法中又做了什么？
判断需要拦截的方法中包不包含需要拦截的方法。如果包含，调用拦截器的intercept方法，如果不包含，只执行原方法内容。

### 7.拦截器的intercept方法中做了什么？
有自定义的内容以及invocation.proceed()方法。也就是执行原方法内容。
