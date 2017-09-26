# 编程式的创建Aop代理之ProxyFactory
Spring Aop是基于代理的，ProxyFactory是Spring Aop内部用来创建Proxy对象的一个工厂类。如果我们需要在程序运行时来动态的应用Spring Aop，则我们可以考虑使用ProxyFactory。使用ProxyFactory时，我们需要为它指定我们需要代理的目标对象、代理时我们需要使用的Advisor或Advice。如下示例就是一个简单的使用ProxyFactory创建MyService对象的代理，同时对其应用了一个MethodBeforeAdvice，即每次调用代理对象的方法时都将先调用MethodBeforeAdvice的before方法。  

```java
	@Test
	public void testProxyFactory() {
		MyService myService = new MyService();
		ProxyFactory proxyFactory = new ProxyFactory(myService);
		proxyFactory.addAdvice(new MethodBeforeAdvice() {

			@Override
			public void before(Method method, Object[] args, Object target) throws Throwable {
				System.out.println("执行目标方法调用之前的逻辑");
				//不需要手动去调用目标方法，Spring内置逻辑里面会调用目标方法
			}
			
		});;
		MyService proxy = (MyService) proxyFactory.getProxy();
		proxy.add();
	}
```
## 指定被代理对象
ProxyFactory有多个重载的构造函数，上面示例中笔者用的是指定被代理对象的构造函数，如果我们应用的是其它构造函数，则可以通过ProxyFactory的setTarget(Object)方法来指定被代理对象。如果我们没有指定被代理对象的Class，那么默认创建出来的代理对象是我们传递的被代理对象的类型，即获取的是targetObject.getClass()类型。如果我们的被代理对象的类型是包含多个接口实现或父类型的，而我们只希望代理其中的某一个类型时，我们可以通过ProxyFactory的setTargetClass(Class)来指定创建的代理对象是基于哪个Class的。默认情况下，ProxyFactory会根据实际情况选择创建的代理对象是基于JDK代理的还是基于CBLIB代理的，即目标对象拥有接口实现且没有设置`proxyTargetClass="true"`或者指定的targetClass是一个接口的时候将采用JDK代理，否则将采用CGLIB代理。也就是说即算是你通过ProxyFactory.setProxyTargetClass(true)指定了将会建立基于Class的CGLIB代理，最终也不一定是CGLIB代理，因为这种情况下如果targetClass是一个接口也将建立JDK代理。这块的逻辑是由DefaultAopProxyFactory的createAopProxy()方法实现的，其源码如下。  

```java
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface()) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```  
ProxyFactory底层在创建代理对象的时候实际上是会委托给AopProxyFactory对象的，AopProxyFactory是一个接口，其只定义了一个createAopProxy()方法，Spring提供了一个默认实现，DefaultAopProxyFactory。ProxyFactory中使用的就是DefaultAopProxyFactory，有兴趣的朋友可以参考一下ProxyFactory的源代码。   
## 指定Advisor
使用Aop时我们是需要对拦截的方法做一些处理的，对于Spring Aop来讲，需要对哪些方法调用做什么样的处理是通过Advisor来定义的，通常是一个PointcutAdvisor。PointcutAdvisor接口中包含主要有两个接口方法，一个用来获取Pointcut，一个用来获取Advice对象，它俩的组合就构成了需要在哪个Pointcut应用哪个Advice。所以有需要的时候我们也可以实现自己的Advisor实现。
```java
/**
 * 简单的实现自己的PointcutAdvisor
 * @author Elim 2017年5月9日
 */
public class MyAdvisor implements PointcutAdvisor {

	@Override
	public Advice getAdvice() {
		return new MethodBeforeAdvice() {

			@Override
			public void before(Method method, Object[] args, Object target) throws Throwable {
				System.out.println("BeforeAdvice实现，在目标方法被调用前调用，目标方法是：" + method.getDeclaringClass().getName() + "."
						+ method.getName());
			}
		};
	}

	@Override
	public boolean isPerInstance() {
		return true;
	}

	@Override
	public Pointcut getPointcut() {
		//匹配所有的方法调用
		return Pointcut.TRUE;
	}

}
```
```java
	@Test
	public void testProxyFactory2() {
		MyService myService = new MyService();
		ProxyFactory proxyFactory = new ProxyFactory(myService);
		proxyFactory.addAdvisor(new MyAdvisor());
		MyService proxy = (MyService) proxyFactory.getProxy();
		proxy.add();
	}
```
上述示例就是一个指定代理对象对应的Advisor的示例。其实一个代理对象可以同时绑定多个Advisor对象，ProxyFactory的addAdvisor()方法可多次被调用，且该方法还有一些重载的方法定义，可以参数Spring的API文档。  
## 指定Advice
我们的第一个示例指定的代理对象绑定的是一个Advice，而第二个示例指定的Advisor，对此你会不会有什么疑问呢？依据我们对Spring Aop的了解，Spring的Aop代理对象绑定的就一定是一个Advisor，而且通常是一个PointcutAdvisor，通过它我们可以知道我们的Advice究竟是要应用到哪个Pointcut（哪个方法调用）？当我们通过ProxyFactory在创建代理对象时绑定的是一个Advice对象时，实际上ProxyFactory内部还是为我们转换为了一个Advisor对象的，只是该Advisor对象对应的Pointcut是一个匹配所有方法调用的Pointcut实例。  
## 指定是否需要发布代理对象
在调用Aop代理对象的方法时，默认情况下我们是不能访问到当前的代理对象的，如果我们指定了创建的代理对象需要对外发布代理对象，那么在调用代理对象的方法时Spring会把当前的代理对象存入AopContext中，我们就可以在目标对象的方法中通过AopContext中获取到当前的代理对象了。这是通过exposeProxy属性来指定的，如果我们希望对外发布代理对象，我们可以通过exposeProxy的set方法来指定该属性的值为true。如：
```java
	@Test
	public void testProxyFactory2() {
		MyService myService = new MyService();
		ProxyFactory proxyFactory = new ProxyFactory(myService);
		proxyFactory.setExposeProxy(true);//指定对外发布代理对象，即在目标对象方法中可以通过AopContext.currentProxy()访问当前代理对象。
		proxyFactory.addAdvisor(new MyAdvisor());
		proxyFactory.addAdvisor(new MyAdvisor());//多次指定Advisor将同时应用多个Advisor
		MyService proxy = (MyService) proxyFactory.getProxy();
		proxy.add();
	}
```
  
除了上述配置信息以外，ProxyFactory其实还可以配置很多其它的信息，更多的配置信息项请参考ProxyFactory的源代码或参考Spring API文档。  
  
**参考文档**
* Spring4.1.0官方文档
* Spring源代码  

（本文是基于Spring4.1.0所写，Elim写于2017年5月9日）