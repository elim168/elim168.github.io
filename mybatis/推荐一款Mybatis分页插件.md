# 推荐一款Mybatis分页插件
以前也写过一篇博文介绍Mybatis的插件，以及如何通过Mybatis的插件功能实现一个自定义的分页插件，但是那个插件的侵入性是比较大的。前段时间遇到了一款开源的Mybatis分页插件，叫`PageHelper`，github地址是[https://github.com/pagehelper/Mybatis-PageHelper](https://github.com/pagehelper/Mybatis-PageHelper)，其原理是通过`ThreadLocal`来存放分页信息，从而可以做到在Service层实现无侵入性的Mybatis分页。笔者感觉还不错，所以特意发博文记录一下，并推荐给大家。  

## 简单示例
以下是使用`PageHelper`进行分页的一个简单的示例，更多详细的内容，请大家参数上面提供的[github地址](https://github.com/pagehelper/Mybatis-PageHelper)。  
### 添加依赖  
笔者使用的是Maven，添加依赖如下。  
```xml
<dependency>
	<groupId>com.github.pagehelper</groupId>
	<artifactId>pagehelper</artifactId>
	<version>4.1.6</version>
</dependency>
```

### 注册Mybatis Plugin
跟其它Mybatis Plugin一样，我们需要在Mybatis的配置文件中注册需要使用的Plugin，`PageHelper`中对应的Plugin实现类就是`com.github.pagehelper.PageHelper`自身。顺便说一句，Mybatis的Plugin我们说是Plugin，实际上对应的却是`org.apache.ibatis.plugin.Interceptor`接口，因为`Interceptor`的核心是其中的`plugin(Object target)`方法，而对于`plugin(Object target)`方法的实现，我们在需要对对应的对象进行拦截时会通过`org.apache.ibatis.plugin.Plugin`的静态方法`wrap(Object target, Interceptor interceptor)`返回一个代理对象，而方法入参就是当前的`Interceptor`实现类。  
```xml
<plugins>  
   <plugin interceptor="com.github.pagehelper.PageHelper"/>  
</plugins>
```

### 使用PageHelper
`PageHelper`拦截的是`org.apache.ibatis.executor.Executor`的`query`方法，其传参的核心原理是通过`ThreadLocal`进行的。当我们需要对某个查询进行分页查询时，我们可以在调用Mapper进行查询前调用一次`PageHelper.startPage(..)`，这样`PageHelper`会把分页信息存入一个`ThreadLocal`变量中。在拦截到`Executor`的`query`方法执行时会从对应的`ThreadLocal`中获取分页信息，获取到了，则进行分页处理，处理完了后又会把`ThreadLocal`中的分页信息清理掉，以便不影响下一次的查询操作。所以当我们使用了`PageHelper.startPage(..)`后，每次将对最近一次的查询进行分页查询，如果下一次查询还需要进行分页查询，需要重新进行一次`PageHelper.startPage(..)`。这样就做到了在引入了分页后可以对原来的查询代码没有任何的侵入性。此外，在进行分页查询时，我们的返回结果一般是一个`java.util.List`，`PageHelper`分页查询后的结果会变成`com.github.pagehelper.Page`类型，其继承了`java.util.ArrayList`，所以不会对我们的方法声明造成影响。`com.github.pagehelper.Page`中包含有返回结果的分页信息，包括总记录数，总的分页数等信息，所以一般我们需要把返回结果强转为`com.github.pagehelper.Page`类型。以下是一个简单的使用`PageHelper`进行分页查询的示例代码。  
```java
public class PageHelperTest {
	
	private static SqlSessionFactory sqlSessionFactory;
	private SqlSession session;
	
	@BeforeClass
	public static void beforeClass() throws IOException {
		InputStream is = Resources.getResourceAsStream("mybatis-config-single.xml");
		sqlSessionFactory = new SqlSessionFactoryBuilder().build(is);
	}
	
	@Before
	public void before() {
		this.session = sqlSessionFactory.openSession();
	}
	
	@After
	public void after() {
		this.session.close();
	}

	@Test
	public void test() {
		int pageNum = 2;//页码，从1开始
		int pageSize = 10;//每页记录数
		PageHelper.startPage(pageNum, pageSize);//指定开始分页
		UserMapper userMapper = this.session.getMapper(UserMapper.class);
		List<User> all = userMapper.findAll();
		Page<User> page = (Page<User>) all;
		System.out.println(page.getPages());
		System.out.println(page);
	}
	
}
```

以上是通过`PageHelper.startPage(..)`传递分页信息的示例，其实`PageHelper`还支持Mapper参数传递分页信息等其它用法。关于`PageHelper`的更多用法和配置信息等请参考该项目的GitHub[官方文档](https://github.com/pagehelper/Mybatis-PageHelper)。  

（本文由Elim写于2017年5月31日）