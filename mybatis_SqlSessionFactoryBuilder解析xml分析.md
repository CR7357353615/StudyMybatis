# Mybatis SqlSessionFactoryBuilder解析xml分析
---
在不考虑与Spring集成的情况下,使用MyBatis执行数据库操作的代码如下：
```java
// String resource = "org/mybatis/example/mybatis-config.xml";
// InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
SqlSession session = sqlSessionFactory.openSession();
// try {
  // Blog blog = session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
// } finally {
  // session.close();
}
```
这里new了一个SqlSessionFactoryBuilder，并调用build方法。这是解析配置文件中的内容，解析为Configuration对象，并以SqlSessionFactory的形式返回。我们看一下build方法：
```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
		try {
			XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment,
					properties);
			//解析xml为Configuration对象，并且包装成SqlSessionFactory的形式返回
			return build(parser.parse());
		} catch (Exception e) {
			throw ExceptionFactory.wrapException("Error building SqlSession.", e);
		} finally {
			ErrorContext.instance().reset();
			try {
				reader.close();
			} catch (IOException e) {
				// Intentionally ignore. Prefer previous error.
			}
		}
}
```
这里先获得一个XMLConfigBuilder，再调用parse()方法解析xml，parse()方法如下：
```java
public Configuration parse() {
		// 如果已经解析过来，抛异常，不再解析
		if (parsed) {
			throw new BuilderException(
					"Each XMLConfigBuilder can only be used once.");
		}
		// 如果第一次解析，将标志置为true
		parsed = true;
		// parse.evalNode("/configuration")获得xml中的<configuration>结点
		parseConfiguration(parser.evalNode("/configuration"));
		return configuration;
}
```
通过parser.evalNode("/configuration")得到了<configuration>结点后调用parseConfiguration方法对configuration进行解析。下面是configuration结点的结构图：

![](imgs/configuration结点.png)
```java
private void parseConfiguration(XNode root) {
		try {
			// issue #117 read properties first
      //解析properties结点
			propertiesElement(root.evalNode("properties"));
      //解析settings结点
			Properties settings = settingsAsProperties(root.evalNode("settings"));
			loadCustomVfs(settings);
			typeAliasesElement(root.evalNode("typeAliases"));
			pluginElement(root.evalNode("plugins"));
			objectFactoryElement(root.evalNode("objectFactory"));
			objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
			reflectorFactoryElement(root.evalNode("reflectorFactory"));
			settingsElement(settings);
			// read it after objectFactory and objectWrapperFactory issue #631
			environmentsElement(root.evalNode("environments"));
			databaseIdProviderElement(root.evalNode("databaseIdProvider"));
			typeHandlerElement(root.evalNode("typeHandlers"));
			mapperElement(root.evalNode("mappers"));
		} catch (Exception e) {
			throw new BuilderException(
					"Error parsing SQL Mapper Configuration. Cause: " + e, e);
		}
}
```

这里看几个比较重要的解析方法

以下是properties结点的解析方法
```java
private void propertiesElement(XNode context) throws Exception {
		if (context != null) {
			// 获取properties结点下的所有子节点的属性名和属性值，并以Properties的形式返回
			Properties defaults = context.getChildrenAsProperties();
			// 获取外部文件
			String resource = context.getStringAttribute("resource");
			String url = context.getStringAttribute("url");
			if (resource != null && url != null) {
				throw new BuilderException(
						"The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
			}
			//将外部文件的属性名和属性值插入
			if (resource != null) {
				defaults.putAll(Resources.getResourceAsProperties(resource));
			} else if (url != null) {
				defaults.putAll(Resources.getUrlAsProperties(url));
			}
			// 找出已经存在的属性
			Properties vars = configuration.getVariables();
			if (vars != null) {
				defaults.putAll(vars);
			}
			// 设置属性
			parser.setVariables(defaults);
			configuration.setVariables(defaults);
		}
}
```

```java
/**
	 * 解析别名
	 * @param parent
	 */
	private void typeAliasesElement(XNode parent) {
		if (parent != null) {
			for (XNode child : parent.getChildren()) {
				// 如果是配置了一个包名，会扫描包下所有需要的Java Bean。如果这些类没有注解，会使用类的简称（首字母变小写），如果有@Alias注解，使用注解中的名称
				if ("package".equals(child.getName())) {
					String typeAliasPackage = child.getStringAttribute("name");
					configuration.getTypeAliasRegistry().registerAliases(
							typeAliasPackage);
				} else {
					String alias = child.getStringAttribute("alias");
					String type = child.getStringAttribute("type");
					try {
						Class<?> clazz = Resources.classForName(type);
						if (alias == null) {
							// 判断是否有注解，没有注解用类简称（首字母小写），有@Alias，用注解中的名称
							typeAliasRegistry.registerAlias(clazz);
						} else {
							// 设置
							typeAliasRegistry.registerAlias(alias, clazz);
						}
					} catch (ClassNotFoundException e) {
						throw new BuilderException(
								"Error registering typeAlias for '" + alias
										+ "'. Cause: " + e, e);
					}
				}
			}
		}
}
```
