# 编程式的创建Aop代理之AspectjProxyFactory
之前已经介绍了一款编程式的创建Aop代理的工厂——`ProxyFactory`，其实`ProxyFactory`拥有的功能`AspectjProxyFactory`都有。它们虽然没有直接的继承关系，但是它们都继承自`ProxyCreatorSupport`，而创建代理对象的核心逻辑都是在`ProxyCreatorSupport`中实现的。所以说`ProxyFactory`拥有的功能`AspectjProxyFactory`都有。那么`AspectjProxyFactory`与`ProxyFactory`相比有什么不同呢？  
`AspectjProxyFactory`的特殊之处就在于其可以直接指定需要创建的代理对象需要绑定的切面。在使用P`roxyFactory`时，我们能够绑定的是`Advisor`或`Advice`，但是如果我们的程序中已经有了现成的切面类定义且能够为我们新创建的代理类使用时，我们还要为了`ProxyFactory`建立代理对象的需要创建对应的`Advisor`类、`Advice`类和`Pointcut`类定义，这无疑是非常麻烦的。`AspectjProxyFactory`通常用于创建基于Aspectj风格的Aop代理对象。现假设我们有如下这样一个切面类定义。  
```java
@Aspect
public class MyAspect {

	@Pointcut("execution(* add(..))")
	private void beforeAdd() {}
	
	@Before("beforeAdd()")
	public void before1() {
		System.out.println("-----------before-----------");
	}
	
}
```
在上述切面类定义中我们定义了一个`Advisor`，其对应了一个`BeforeAdvice`，实际上是一个`AspectJMethodBeforeAdvice`，该`Advice`对应的是上面的`before1()`方法，还对应了一个`Pointcut`，实际上是一个`AspectJExpressionPointcut`。该`Advisor`的语义就是调用所有的方法名为“add”的方法时都将在调用前调用`MyAspect.before1()`方法。如果我们现在需要创建一个代理对象，其需要绑定的`Advisor`逻辑跟上面定义的切面类中定义的`Advisor`类似。则我们可以进行如下编程。  
```java
	@Test
	public void testAspectJProxyFactory() {
		MyService myService = new MyService();
		AspectJProxyFactory proxyFactory = new AspectJProxyFactory(myService);
		proxyFactory.addAspect(MyAspect.class);
		proxyFactory.setProxyTargetClass(true);//是否需要使用CGLIB代理
		MyService proxy = proxyFactory.getProxy();
		proxy.add();
	}
```
在上述代码中我们`AspectjProxyFactory`在创建代理对象时需要使用的切面类（<font color="red">其实addAspect还有一个重载的方法可以指定一个切面类的对象</font>），其实在`AspectjProxyFactory`内部还是解析了该切面类中包含的所有的`Advisor`，然后把能够匹配当前代理对象类的`Advisor`与创建的代理对象绑定了。有兴趣的读者可以查看一下`AspectjProxyFactory`的源码，以下是部分核心代码。
```java
	public void addAspect(Class<?> aspectClass) {
		String aspectName = aspectClass.getName();
		AspectMetadata am = createAspectMetadata(aspectClass, aspectName);
		MetadataAwareAspectInstanceFactory instanceFactory = createAspectInstanceFactory(am, aspectClass, aspectName);
		addAdvisorsFromAspectInstanceFactory(instanceFactory);
	}

	private void addAdvisorsFromAspectInstanceFactory(MetadataAwareAspectInstanceFactory instanceFactory) {
		List<Advisor> advisors = this.aspectFactory.getAdvisors(instanceFactory);
		advisors = AopUtils.findAdvisorsThatCanApply(advisors, getTargetClass());
		AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(advisors);
		OrderComparator.sort(advisors);
		addAdvisors(advisors);
	}
```


> 需要注意的是在使用`AspectjProxyFactory`基于切面类创建代理对象时，我们指定的切面类上必须包含@Aspect注解。  
> 另外需要注意的是虽然我们自己通过编程的方式可以通过`AspectjProxyFactory`创建基于`@Aspect`标注的切面类的代理，但是通过配置`<aop:aspectj-autoproxy/>`使用基于注解的Aspectj风格的Aop时，Spring内部不是通过`AspectjProxyFactory`创建的代理对象，而是通过`ProxyFactory`。有兴趣的朋友可以查看一下`AnnotationAwareAspectjAutoProxyCreator`的源代码。

（注：本文是基于Spring4.1.0所写，Elim写于2017年5月9日）