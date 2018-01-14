# Mybatis解析Xml文件
---
## Mybatis主要的类
类名|职责
--|--
Configuration|MyBatis所有的配置信息都维持在Configuration对象之中。
SqlSession|作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能
Executor|MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护
StatementHandler|封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。
ParameterHandler|负责对用户传递的参数转换成JDBC Statement 所需要的参数
ResultSetHandler|负责将JDBC返回的ResultSet结果集对象转换成List类型的集合
TypeHandler|负责java数据类型和jdbc数据类型之间的映射和转换
MappedStatement|MappedStatement维护了一条<select,update,delete,insert>节点的封装
SqlSource|负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回
BoundSql|表示动态生成的SQL语句以及相应的参数信息

## 解析xml配置文件主流程
#### 1.Spring中的配置项
```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.sqlSessionFactoryBean">
    <propert name="dataSource" ref="dataSource"/>
    <propert name="configLocation" value="classpath:configuration.xml"/>
    <propert name="mapperLocations" value="classpath:com/xxx/mybatis/mapper/*.xml"/>
    <propert name="typeAliasesPackage" value="com.xx.xx.xx.model"/>
</bean>
```
#### 2.sqlSessionFactory会根据mapperLocations这个位置读取下面所有的xml文件，代码如下：
#### 3.在sqlSessionFactoryBean类中的buildSqlSessionFactory方法中
#### 4.使用XMLMapperBuilder类的实例解析mapper配置文件
```java
if (!isEmpty(this.mapperLocations)) {
    // for (Resource mapperLocation : this.mapperLocations) {
    //     if (mapperLocation == null) {  
    //       continue;  
    //     }  
    //
    //     try {  
            XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),  
            configuration, mapperLocation.toString(), configuration.getSqlFragments());  
            xmlMapperBuilder.parse();  
    //     } catch (Exception e) {  
    //         throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);  
    //     } finally {  
    //         ErrorContext.instance().reset();  
    //     }  
    //
    //     if (logger.isDebugEnabled()) {  
    //         logger.debug("Parsed mapper file: '" + mapperLocation + "'");  
    //     }  
    // }  
}  
```
#### 5.接着系统调用xmlMapperBuilder的parse方法解析mapper
#### 6.parse方法中调用configurationElement方法解析mapper
##### 这里几乎解析了mapper节点下的所有子节点。至此，mybatis已经解析了mapper中的所有节点，并且将其加到了configuration对象中，提供给sqlSessionFactory随时使用。下面是详细内容





## 1.XMLMapperBuilder类
### parse()方法
```java
public void parse() {
    // if (!configuration.isResourceLoaded(resource)) {
      //根据mapper节点解析
      configurationElement(parser.evalNode("/mapper"));
    //   configuration.addLoadedResource(resource);
    //   bindMapperForNamespace();
    // }
    //
    // parsePendingResultMaps();
    // parsePendingCacheRefs();
    // parsePendingStatements();
}
```
```java
private void configurationElement(XNode context) {
    // try {
    //   String namespace = context.getStringAttribute("namespace");
    //   if (namespace == null || namespace.equals("")) {
    //     throw new BuilderException("Mapper's namespace cannot be empty");
    //   }
    //   builderAssistant.setCurrentNamespace(namespace);
      //处理缓存
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      //处理参数
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      //处理返回值
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      //处理sql
      sqlElement(context.evalNodes("/mapper/sql"));
      //处理增删改查
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    // } catch (Exception e) {
    //   throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    // }
  }
```
##### 看一下增删改查的方法
```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    for (XNode context : list) {
      //对于每个节点，使用XMLStatmentBuilder解析
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      // try {
        statementParser.parseStatementNode();
      // } catch (IncompleteElementException e) {
        // configuration.addIncompleteStatement(statementParser);
      // }
    }
}
```
## 2.XMLStatementBuilder类
```java
public void parseStatementNode() {
    String id = context.getStringAttribute("id");
    String databaseId = context.getStringAttribute("databaseId");

    if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
      return;
    }

    // Integer fetchSize = context.getIntAttribute("fetchSize");
    // Integer timeout = context.getIntAttribute("timeout");
    // String parameterMap = context.getStringAttribute("parameterMap");
    // String parameterType = context.getStringAttribute("parameterType");
    // Class<?> parameterTypeClass = resolveClass(parameterType);
    // String resultMap = context.getStringAttribute("resultMap");
    // String resultType = context.getStringAttribute("resultType");
    // String lang = context.getStringAttribute("lang");
    //使用LanguageDriver解析sql，并得到SqlSource
    LanguageDriver langDriver = getLanguageDriver(lang);

    // Class<?> resultTypeClass = resolveClass(resultType);
    // String resultSetType = context.getStringAttribute("resultSetType");
    // StatementType statementType = StatementType.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
    // ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
    //
    // String nodeName = context.getNode().getNodeName();
    // SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
    // boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
    // boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
    // boolean useCache = context.getBooleanAttribute("useCache", isSelect);
    // boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);
    //
    // // Include Fragments before parsing
    // XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
    // includeParser.applyIncludes(context.getNode());
    //
    // // Parse selectKey after includes and remove them.
    // processSelectKeyNodes(id, parameterTypeClass, langDriver);

    // Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
    //生成SqlSource
    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
    // String resultSets = context.getStringAttribute("resultSets");
    // String keyProperty = context.getStringAttribute("keyProperty");
    // String keyColumn = context.getStringAttribute("keyColumn");
    // KeyGenerator keyGenerator;
    // String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
    // keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
    // if (configuration.hasKeyGenerator(keyStatementId)) {
    //   keyGenerator = configuration.getKeyGenerator(keyStatementId);
    // } else {
    //   keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
    //       configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
    //       ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
    // }
    //调用助手类
    builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType,
        fetchSize, timeout, parameterMap, parameterTypeClass, resultMap, resultTypeClass,
        resultSetTypeEnum, flushCache, useCache, resultOrdered,
        keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
  }
```
<br>
## 3.XMLLanguageDriver类
##### 默认使用XMLLanguageDriver的createSqlSource去生成SqlSource

```java
@Override
public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
  // XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
  return builder.parseScriptNode();
}
```

## 4.XMLScriptBuilder类

```java
public SqlSource parseScriptNode() {
    //解析节点以及其子节点（<set><where>等等）
    List<SqlNode> contents = parseDynamicTags(context);
    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
    SqlSource sqlSource = null;
    if (isDynamic) {
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
}
```
```java
List<SqlNode> parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    //获得子节点
    NodeList children = node.getNode().getChildNodes();
    //循环遍历子节点
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        TextSqlNode textSqlNode = new TextSqlNode(data);
        if (textSqlNode.isDynamic()) {
          contents.add(textSqlNode);
          isDynamic = true;
        } else {
          contents.add(new StaticTextSqlNode(data));
        }
      } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        NodeHandler handler = nodeHandlers(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return contents;
}
```

## trim标签是如何实现的
## TrimSqlNode类
```java
@Override
public boolean apply(DynamicContext context) {
  // FilteredDynamicContext filteredDynamicContext = new FilteredDynamicContext(context);
  // boolean result = contents.apply(filteredDynamicContext);
  filteredDynamicContext.applyAll();
  // return result;
}
```
```java
public void applyAll() {
    // sqlBuffer = new StringBuilder(sqlBuffer.toString().trim());
    // String trimmedUppercaseSql = sqlBuffer.toString().toUpperCase(Locale.ENGLISH);
    // if (trimmedUppercaseSql.length() > 0) {
     applyPrefix(sqlBuffer, trimmedUppercaseSql);
     applySuffix(sqlBuffer, trimmedUppercaseSql);
    // }
    // delegate.appendSql(sqlBuffer.toString());
}
```
```java
private void applyPrefix(StringBuilder sql, String trimmedUppercaseSql) {
    if (!prefixApplied) {
        prefixApplied = true;
        //如果需要去除的前缀（AND OR等）不为空
        if (prefixesToOverride != null) {
            for (String toRemove : prefixesToOverride) {
                //如果这段sql中是以这些前缀开头的，需要删除
                if (trimmedUppercaseSql.startsWith(toRemove)) {
                    sql.delete(0, toRemove.trim().length());
                    break;
                }
            }
        }
        //如果前缀字符串不为空
        if (prefix != null) {
            //需要加上前缀
            sql.insert(0, " ");
            sql.insert(0, prefix);
        }
    }
}
```
```java
//后缀的处理方式相同
private void applySuffix(StringBuilder sql, String trimmedUppercaseSql) {
    if (!suffixApplied) {
        suffixApplied = true;
        if (suffixesToOverride != null) {
            for (String toRemove : suffixesToOverride) {
                if (trimmedUppercaseSql.endsWith(toRemove) || trimmedUppercaseSql.endsWith(toRemove.trim())) {
                    int start = sql.length() - toRemove.trim().length();
                    int end = sql.length();
                    sql.delete(start, end);
                    break;
                }
            }
        }
        if (suffix != null) {
            sql.append(" ");
            sql.append(suffix);
        }
    }
}
```
