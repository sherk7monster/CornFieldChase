---
title:  "Mybatis-数据库查询的记录是如何转变成Java对象的"
category: "Mybatis"
---

2021-02-04 记

<script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
<span id="busuanzi_container_page_pv">
  阅读量&nbsp;<span id="busuanzi_value_page_pv"></span>&nbsp;次，
</span>本文约 {{ content | strip_html | strip_newlines | split: "" | size }} 字

目录
* 目录
{:toc}

## 从 `Mybatis` 官网的入门 `Demo` 说起

#### 1、`Demo` 目录结构

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlExecuteDemo.jpg){: .align-center}

#### 2、测试类

```java
public class Demo {
    public static void main(String[] args) throws Exception {
        String resource = "mybatis-config.xml";
        InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        try (SqlSession session = sqlSessionFactory.openSession()) {
            BlogMapper mapper = session.getMapper(BlogMapper.class);
            Blog blog = mapper.selectBlog(101);
            System.out.println(blog);
        }
    }
}
```

#### 3、mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
  <mappers>
    <mapper resource="org/mybatis/example/BlogMapper.xml"/>
  </mappers>
</configuration>
```

#### 4、BlogMapper.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.mybatis.example.BlogMapper">
    <select id="selectBlog" resultType="org.mybatis.example.Blog">
        select * from Blog where id = #{id}
    </select>
</mapper>
```

#### 5、`Blog` 表结构

```text
DROP TABLE IF EXISTS `Blog`;
CREATE TABLE `Blog` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `blogName` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=100 DEFAULT CHARSET=utf8mb4; 
```

## 配置文件读取

代码：
```java
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
```

上述代码表达的内容很简单，就是读取 `mybatis-config.xml` 文件转成输入流，虽然代码简单，但确是 `Mybatis` 实现 `ORM` （对象关系映射）的基石。

#### `Resources#getResourceAsStream()` 调用流程

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoResource.jpg){: .align-center}

从调用链可以看出，最终还是调用的 `JDK` 的 `ClassLoader` 完成的文件读取。


## 构建 `SqlSessionFactory`

代码：
```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

看下 `SqlSessionFactoryBuilder` 是如何创建 `SqlSessionFactory` 的。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoSessionFactory.jpg){: .align-center}

由调用链可以看出，`SqlSessionFactoryBuilder` 通过 `XMLConfigBuilder` 解析了 `mybatis-config.xml`，
并将解析的内容封装成 `Configuration` 传给了 `DefaultSqlSessionFactory`，最终 `SqlSessionFactory` 的具体实例就是 `DefaultSqlSessionFactory`。 

## 创建 `SqlSession`

代码：
```java
SqlSession session = sqlSessionFactory.openSession();
```

`SqlSession` 顾名思义就是与数据库的会话，且看 `SqlSessionFactory` 是如何创建 `SqlSession` 的。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoSession.jpg){: .align-center}

可以看到，`DefaultSqlSessionFactory` 通过 `Configuration` 拿到了 `Environment` 和 `Executor`，并通过 `Environment` 获取了 `Transaction` 对象。
最终返回了 `DefaultSqlSession` 对象。


## 创建 `BlogMapper` 实例

代码：
```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
```

`BlogMapper` 在 `Demo` 中只定义为了一个接口，实际的对象创建是在 `SqlSession` 中。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoMapper.jpg){: .align-center}

从调用链看到，`DefaultSqlSession` 最终是通过 `MapperProxyFactory` 创建的 `BlogMapper` 的实例，并且这个实例是 `Proxy#newProxyInstance()` 创建的代理类。
先说 `MapperProxyFactory`，`MapperProxyFactory` 是在 `MapperRegistry` 中创建的，在 `MapperRegistry` 中注意到这样一段代码：

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```

上述代码中表明 `mapperProxyFactory` 是存在 `knownMappers` 中的，`knownMappers` 是一个 `HashMap`，`knownMappers` 是根据传入的 `BlogMapper` 的 `class` 类型对象获取的 `mapperProxyFactory`，
并且 `mapperProxyFactory` 不为空，那 `BlogMapper` 对应的 `MapperProxyFactory` 究竟是在哪里被创建的呢？

我们根据 `knownMappers` 在 `MapperRegistry` 中进行查找，发现了 `MapperRegistry` 的 `addMapper` 方法，如下：

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        //省略。。
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
}
```

在 `addMapper` 方法中，看到了 `knownMappers` 的 `put` 方法，在 `addMapper` 入口处打个断点，重跑下 `Demo`。断点处的方法调用栈如下：

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlXmlParserDebug.jpg){: .align-center}

一步步往回查看调用栈，定位到 `XMLConfigBuilder` 的 `parseConfiguration` 方法：

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlMapperXmlParserDebug.jpg){: .align-center}

是不是很熟悉，到这里就明白了，`BlogMapper` 对应的 `MapperProxyFactory` 对象，是在 `Mybatis` 解析 `mybatis-config.xml` 的 `mappers` 节点时被创建的。

回到 `BlogMapper` 的实例创建，注意到 `MapperProxy` 这个对象，回顾 `JDK` 创建代理类的方法，先实现一个 `InvocationHandler` 对象，
再调用 `Proxy#newProxyInstance()` 根据传入的 `InvocationHandler` 实现创建真正的代理对象。而 `MapperProxy` 正是实现了 `InvocationHandler` 接口。


## MapperProxy

继续看 `MapperProxy` 的 `invoke` 方法：

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else if (method.isDefault()) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
}
```

首先，判断被代理的方法是不是 `Object` 类声明的，或者是不是 `default` 方法，如果都不是，比如这里被代理的方法是：

```java
Blog blog = mapper.selectBlog(101);
```

程序接着往下走到 `mapperMethod#execute()`。

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoQueryReturn.jpg){: .align-center}

上面展示的就是 `Mybatis` 如何执行 `SQL` 查询，并封装返回对象 `Blog`。可以看到，主要流程集中在 `DefaultResultSetHandler` 中。
回过头再看一下 `BlogMapper` 的定义：

```java
public interface BlogMapper {
    Blog selectBlog(Integer id);
}
```

`BlogMapper` 是如何被创建的我们已经知道了，接下来重点分析一下其他几个问题：
* `id` 是如何赋值给 `SQL` 的
* 数据库返回记录行又是如何转化为 `Blog` 对象的

## `id` 是如何赋值给 `SQL` 的

#### 获取 `id` 值

我们注意到在 `MapperProxy#invoke` 方法中，`MapperMethod#execute` 执行之前，有这么一段代码：

```java
final MapperMethod mapperMethod = cachedMapperMethod(method);
return mapperMethod.execute(sqlSession, args);
```

进到 `cachedMapperMethod()` 去看 `MapperMethod` 是如何被创建的。我们注意到 `MapperMethod` 中有2个关键属性：
`SqlCommand` 和 `MethodSignature`。一路跟踪 `SqlCommand`，发现几个比较敏感的对象，其中之一是 `MappedStatement`。
这个是不是很像 `JDBC` 的 `Statement` 对象，`MappedStatement` 对象是根据 `statementId` 从 `configuration` 中拿到的，
`statementId` 在这里是 `org.mybatis.example.BlogMapper.selectBlog`。那么问题来了，`MappedStatement` 是什么时候被塞到 `configuration` 中的呢？
根据前面的经验，我们猜测还是与 `Mybatis` 解析 `mybatis-config.xml` 的 `mappers` 节点有关。经过断点调试，找到了以下2段相关代码。

代码1，`XMLMapperBuilder#configurationElement`:

```java
private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
}
```

代码2，`MapperBuilderAssistant#addMappedStatement`：

```java
public MappedStatement addMappedStatement() {//省略入参
    //省略部分代码。。
    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);
    return statement;
  }
```

到这里是不是就清晰了，代码1中解析的正是 `BlogMapper.xml` 的各个节点，包括 `select` 节点。
然后以 `BlogMapper` 类名加上点号，再拼上 `select` 节点的 `id` 属性值为 `key`，封装了 `MappedStatement` 为 `value` 对象，塞到了 `configuration` 中。

再来看 `MethodSignature` ，该类主要是封装了 `BlogMapper#selectBlog` 的方法签名，其中的 `ParamNameResolver` 则是将方法的参数名和参数所在下标位置进行了映射。

回到 `MapperMethod#execute`，继续往下跟，发现这样一行代码：

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    //省略部分代码。。
    Object param = method.convertArgsToSqlCommandParam(args);
    result = sqlSession.selectOne(command.getName(), param);
    //省略部分代码。。
}
```

其中 `method` 对象是 `MethodSignature`，刚才上面提到 `MethodSignature` 封装了被代理方法的参数名和参数所在下标的映射，
这里就根据传入的参数数组 `args` 获取了具体的参数值，如此，`id` 被获取出来传给了 `sqlSession`。

#### `id` 值绑定 `SQL`

继续向下走，追踪到了 `CachingExecutor#query`：

```java
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

这里的 `parameterObject` 就跟前面 `id` 值相关联，在这里 `parameterObject` 值就是 `101`。`BoundSql` 最终创建的地方是 `StaticSqlSource#getBoundSql`：

```java
public BoundSql getBoundSql(Object parameterObject) {
    return new BoundSql(configuration, sql, parameterMappings, parameterObject);
}
```

这里的 `sql` 就是 `BlogMapper.xml` 定义的 `select` 语句：

select * from Blog where id = ?
{: .notice--info}

到这里，`id` 值终于与 `sql` 相遇。

沿着调用链继续跟，到了 `SimpleExecutor#doQuery` 处：

```java
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
```

在这里，`Mybatis` 根据 `BoundSql` 和其他参数一起构建了 `Statement` 对象。


## 数据库返回记录行又是如何转化为 `Blog` 对象的

#### 数据库查询的执行

根据前面的调用链，一路跟到了 `PreparedStatementHandler#query`，就是在这里执行了数据库实际的查询：

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    return resultSetHandler.handleResultSets(ps);
}
```

#### `Blog` 对象的创建

还是看前面的调用链图，根据方法名，可以看到很多跟数据库查询结果处理的逻辑，基本都在 `DefaultResultSetHandler` 中，
继续跟踪代码到 `DefaultResultSetHandler#getRowValue` 方法：

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
    final ResultLoaderMap lazyLoader = new ResultLoaderMap();
    Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
    if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      boolean foundValues = this.useConstructorMappings;
      if (shouldApplyAutomaticMappings(resultMap, false)) {
        foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
      }
      foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
      foundValues = lazyLoader.size() > 0 || foundValues;
      rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
    }
    return rowValue;
}
```

从里面的方法名也基本可以看出，这里就是封装 `Blog` 对象以及设置对象属性大本营。首先看 `Blog` 对象的创建，一路跟踪 `createResultObject` 方法，最终定位到了这块：

```java
private Object createResultObject(ResultSetWrapper rsw, ResultMap resultMap, List<Class<?>> constructorArgTypes, List<Object> constructorArgs, String columnPrefix)
      throws SQLException {
    if (hasTypeHandlerForResultObject(rsw, resultType)) {
      //省略部分代码
    } else if (resultType.isInterface() || metaType.hasDefaultConstructor()) {
      return objectFactory.create(resultType);
    } 
    //省略部分代码
  }
```

`Blog` 对象的创建主要就是由 `objectFactory.create(resultType)` 执行的，那 `objectFactory` 是何方神圣？通过断点调试知道，
`objectFactory` 其实是 `DefaultObjectFactory` 实例。一路跟踪下去，直到 `DefaultObjectFactory#instantiateClass`：

```java
private  <T> T instantiateClass(Class<T> type, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
    try {
      Constructor<T> constructor;
      if (constructorArgTypes == null || constructorArgs == null) {
        constructor = type.getDeclaredConstructor();
        try {
          return constructor.newInstance();
        } catch (IllegalAccessException e) {
          if (Reflector.canControlMemberAccessible()) {
            constructor.setAccessible(true);
            return constructor.newInstance();
          } else {
            throw e;
          }
        }
      }
  }//省略部分代码
```

到这里就清楚了，这里的 `type` 就是 `Blog.class`，在这里获取 `Blog` 的构造器，反射创建了 `Blog` 对象。

#### `Blog` 对象属性的赋值

接着往下看 `applyAutomaticMappings()`，又是一路跟踪调试，最终定位到了 `BeanWrapper#setBeanProperty`：

```java
private void setBeanProperty(PropertyTokenizer prop, Object object, Object value) {
    try {
      Invoker method = metaClass.getSetInvoker(prop.getName());
      Object[] params = {value};
      try {
        method.invoke(object, params);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    } catch (Throwable t) {
      throw new ReflectionException("Could not set property '" + prop.getName() + "' of '" + object.getClass() + "' with value '" + value + "' Cause: " + t.toString(), t);
    }
}
```

这里的 `object` 对象就是前面反射创建的 `Blog` 对象实例，`value` 就是数据查询出的 `Blog` 表 `id` 为 `101` 的记录所有字段的值。
`prop` 里面封装着 `Blog` 对应的各属性字段名。而 `Invoker` 这个对象实际得到的是 `MethodInvoker` 对象实例，看一下当时的断点快照：

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/mybatis/sqlDemoPropSet.jpg){: .align-center}

从图中可以看到，`MethodInvoker` 中也有一个 `method` 属性，这个 `method` 就是 `java.lang.reflect.Method`，
而这个 `Method` 对应的正是 `Blog` 对象属性的 `set` 方法。然后程序在 `setBeanProperty` 中调用 `method.invoke(object, params)`。
最终调用的就是 `Blog` 的 `set*` 方法，至此，`Blog` 对象被赋予了属性值，之后一路 `return`。

## 结语

到这里，`Mybatis` 从如何执行数据库查询，到将查询结果封装为一个 `Java` 对象的整个过程就从源码的角度分析完了。
当然了，并不是所有细节都进行了分析，后续会针对一些细节，再从 `Mybatis` 源码角度进行多次分析。

