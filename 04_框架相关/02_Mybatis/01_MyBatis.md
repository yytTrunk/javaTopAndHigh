# MyBatis

### 一 MyBatis是什么？

1. MyBatis是一个半自动ORM(Object-Relationl Mapping，对象关系映射)框架，能够将数据库与程序中对象进行相互映射。MyBatis通过配置xml文件或者注解方式，进行配置和映射对象信息，使得能够像操作对象一样，从数据库中获取更新数据。
2. MyBatis内部封装了JDBC，开发过程只需重点关注SQL语句，不需要去处理驱动加载，创建连接

### 二 myBatis-config.xml可配置参数

| 名称                                 | 作用                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| 属性（properties）                   | 系统属性通过配置文件配置                                     |
| 设置（settings）                     | 改变MyBatis 的运行时行为                                     |
| 类型别名（typeAliases）              | 为type配置别名，使显示更加简洁                               |
| 类型处理器（typeHandlers）           | 将从预处理语句或者结果集中参数，以合适的方式转换成 Java 类型 |
| 对象工厂（objectFactory）            | 提供默认构造器或者执行构造参数初始化目标类型对象             |
| 插件（plugins）                      | 在映射语句执行过程中的某一点进行拦截调用                     |
| 环境配置（environments）             | 配置多种数据库环境                                           |
| 数据库厂商标识（databaseIdProvider） | 根据不同的数据库厂商执行不同的语句                           |
| 映射器（mappers）                    | 定义 SQL 映射文件                                            |

### 三 MyBatis主要结构

| 名称              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| Configuration     | 对应mysql-config.xml的全局配置关系类                         |
| SqlSessionFactory | 管理session的工厂接口                                        |
| Session           | SqlSession与数据库建立连接的接口。包含了操作数据库的方法     |
| Executor          | 执行器接口，负责Sql语句的生成和查询缓存维护                  |
| MappedStatement   | 底层封装对象，对操作数据库的语句存储封装，对应映射文件里select\|update\|delete\|insert标签里对应内容 |
| StatementHandler  | 封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数等 |
| ResultSetHandler  | 处理操作数据库后返回的结果集handler接口                      |

![MyBatis结构图](.\img\01_03.jpg)

### 三 MyBatis代码中主要处理流程？

1. 读取配置文件mysql-config.xml，对配置文件内容，进行解析，存入对应类为Configureation中
2. 将解析后mysql-config.xml中的配置信息，作为参数传入DefaultSqlSessionFactory中，构造管理session的工厂SqlSessionFactory
3. 通过SqlSessionFactory中创建一个与数据库的连接sqlSession，包含事务、一级缓存、拦截器，都在此时进行初始化
4. SqlSession接口主要定义了操作数据库（查询、删除、更新等）方法，调用这些方法执行相应sql语句
5. 调用SqlSession中方法后，最终都是通过执行器executor执行
6. 执行前，通过MapperStatement对执行Sql语句，参数进行封装
7. 封装完成，通过newStatementHandler去执行sql
8. 通过handlerResultSets对返回结果进行处理

![MyBatis处理流程](./img/01_02.PNG)

### 四 MyBatis如何将XML映射文件转化为MyBatis中内部文件？

对于mybatis-config.xml配置文件

读取mybatis-config.xml文件后，会进行XML文件解析转为对应Document 对象，进而进行MyBatis配置参数的解析，全部转化放入Configureation对象中。然后将Configureation传入，构建SqlSession工厂DefaultSqlSessionFactory。

对于Mapper.xml映射文件

- 对于查询结果resultMap、变量parameterMap，都会被解析为对应对象

- 对于<select> <insert> <update>  <delete> 标签，会被转化为MappedStatement 对象，标签内Sql转化为Boundle对象。因此在执行Sql时，会去找对应MappedStatement 

```java
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    // 解析<select> <insert> <update>  <delete>表签
    parsePendingStatements();
  }
```

### 五 MyBatis中执行器分类？

三种基本的执行器，BatchExecutor、ReuseExecutor、SimpleExecutor

**SimpleExecutor**

默认执行器，每次执行一次update或者select就会开启一个Statement，执行完成后就会关闭Statement

**BatchExecutor**

批量执行器，执行update

**ReuseExecutor**

不同于SimpleExecutor执行器，每执行完一次就关闭Statement，而是能够重复使用statement



### 六 MyBatis中$和#号区别？

$和#都是参数标记，在动态解析时有差别

1. 使用#{}，在执行到预编译时，会将#{}替换为？占位符，执行时替换为实际值，然后加上单引号，从而防止sql注入。

   ```java
   PreparedStatement ps = conn.prepareStatement(sql);
   ps.setInt(1,id);
   // 初始Mapper中定义SQL为  Select * from tb_user where name = #{userName}
   // 执行前boundSql中变为：Select * from tb_user where name = ？
   ```

2. 使用${}，会将传入的数据直接显示生成在sql中，只是做简单的字符串替换，不会添加单引号

   ```java
   Statement st = conn.createStatement();
   ResultSet rs = st.executeQuery(sql);
   // sql为 Select * from user where name = ${userName}
   // 传入参数为
   // 解析后执行的SQL：Select * from user where name = xiaoming
   ```

**区别**

- $方式无法防止Sql注入， #方式能够很大程度防止sql注入
- $方式一般用于传入数据库对象，如表名
- 通常多用#{}



### 七 通常一个Xml映射文件，都会写一个Dao接口与之对应，请问，这个Dao接口的工作原理是什么？Dao接口里的方法，参数不同时，方法能重载吗？

Dao接口即指mapper接口，一个xml文件会对应于一个Dao。

接口的全限名，就是映射文件中的namespace的值，接口的方法名，就是映射文件中MappedStatement的id值，接口方法内的参数，就是传递给sql的参数。

举例来说：cn.mybatis.mappers.StudentDao.findStudentById，可以唯一找到namespace为 com.mybatis.mappers.StudentDao下面 id 为 findStudentById 的 MapperStatement。



不能重载，因为当调用接口方法时，接口全限名+方法名拼接字符串作为key值，可唯一定位一个MappedStatement。

Mapper 接口的工作原理是JDK动态代理，Mybatis运行时会使用JDK动态代理为Mapper接口生成代理对象proxy，代理对象会拦截接口方法，转而执行MapperStatement所代表的sql，然后将sql执行结果返回。

```xml
<mapper namespace="userMapper">
    <select id="selectUser" resultType="com.ttsummer.pojo.User">
      select * from tb_user where id = #{id}
    </select>
</mapper>
```



### 八 MyBatis中缓存？

通过缓存，能够提高查询效率，MyBatis中包含一级缓存，二级缓存。

#### 一级缓存

- 默认开启。
- 基于一次会话连接（SqlSession）,建立缓存，当下次查询的时候，先判断缓存中是否完全一样的查询条件，若有，会从缓存中直接取结果，减少一次数据库查询。
- 当SqlSession调用了close()方法，会释放掉一级缓存PerpetualCache对象，一级缓存将不可用。
- 调用clearCache()方法，能够手动清除缓存
- 当执行了update操作，也会清空缓存对象

**一级缓存执行流程**

1. 对于某个查询，根据statementId,params,rowBounds来构建一个key值，根据这个key值去缓存Cache中取出对应的key值存储的缓存结果

2. 判断从Cache中根据特定的key值取的数据数据是否为空，即是否命中；

3. 如果命中，则直接将缓存结果返回；

4. 如果没命中：

   1) 去数据库中查询数据，得到查询结果；

   2) 将key和查询到的结果分别作为key,value对存储到Cache中；

   3) 将查询结果返回；

```java
// 测试
User user = userMapper.queryUserById(1);
System.out.println(user.toString());
user = userMapper.queryUserById(1);
System.out.println(user.toString());
// 返回结果
[DEBUG] ==>  Preparing: select * from tb_user where id = ? 
[DEBUG] ==> Parameters: 1(Integer)
[DEBUG] <==      Total: 1
birthday='2019-02-20', created='2019-02-27 00:00:00', updated='2019-02-28 00:50:41',
birthday='2019-02-20', created='2019-02-27 00:00:00', updated='2019-02-28 00:50:41',
```

**一级缓存存在问题**

一级缓存不能跨Session，当其它Session在update了数据后，本Session会查询到脏数据，不同的会话之间对于相同的数据可能有不一样的缓存。

一级缓存无法关闭，但可以将一级缓存的级别设为 statement 级别的，这样每次查询结束都会清掉一级缓存。

#### 二级缓存

二级缓存能够解决一级缓存不能跨会话共享的问题的，范围是namespace级别的，存在多个Mapper实例中，可以被多个SqlSession 共享（只要是同一个接口里面的相同方法，都可以共享）。

其他的session执行 update， 整个缓存都会刷新

事务不提交，二级缓存不存在

**适用** 由于更新时会刷新缓存，适用于查询频率很高， 更新频率低时；同时，建议操作单表时使用，避免一个namespace中刷新了缓存，另一个namespace 中没有刷新，会出现读到脏数据

可参考[MyBatis缓存机制](https://www.cnblogs.com/wuzhenzhao/p/11103043.html)



### 九 MyBatis中插件的使用？

其实主要实现拦截器的功能。允许在已映射语句执行过程中的某一点进行拦截调用。

仅可编写

```java
// 拦截执行器的方法，mapper语句执行时拦截
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed) 
// 拦截参数的处理，设置参数时进行拦截
ParameterHandler (getParameterObject, setParameters) 
// 拦截结果集的处理，可以对查询返回结果做处理
ResultSetHandler (handleResultSets, handleOutputParameters) 
// 拦截Sql语法构建的处理
StatementHandler (prepare, parameterize, batch, update, query) 
```

MyBatis使用JDK动态代理，为需要拦截的接口，生成代理对象以实现接口方法的拦截。每当执行到4种接口对象的方法时，就会进入拦截方法。

Mybatis采用责任链模式，通过动态代理组织多个拦截器（插件）。

```java
// 实现Interceptor接口
@Intercepts({@Signature(type= Executor.class, method = "query", args = {
        MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class})})
public class SqlInterceptor implements Interceptor {

    // 进行拦截后要执行的方法
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("method intercept " + invocation.getMethod().getName());
        // 正常执行方法
        Object result = invocation.proceed();
        return result;
    }

    /**
     * plugin方法是拦截器用于封装目标对象的，通过该方法我们可以返回目标对象本身，也可以返回一个它的代理
     *
     * 当返回的是代理的时候我们可以对其中的方法进行拦截来调用intercept方法 -- Plugin.wrap(target, this)
     * 当返回的是当前对象的时候 就不会调用intercept方法，相当于当前拦截器无效
     */
    public Object plugin(Object target) {
        System.out.println("method plugin");
        return Plugin.wrap(target, this);
    }

    // 拦截拦截器设置参数时
    public void setProperties(Properties properties) {
    }
}

//全局xml配置：
<plugins>
    <plugin interceptor="org.format.mybatis.cache.interceptor.ExamplePlugin"></plugin>
</plugins>
```

这个拦截器拦截Executor接口的update方法（新增，删除，修改操作），所有执行executor的update方法都会被该拦截器拦截到。









## 参考

- JavaGuide中MyBatis章节













