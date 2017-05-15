# Spring Aop原理之Advised接口
通过之前我们介绍的`ProxyFactory`我们知道，Spring Aop是通过`ProxyFactory`来创建代理对象的。`ProxyFactory`在创建代理对象时会委托给`DefaultAopProxyFactory.createAopProxy(AdvisedSupport config)`，`DefaultAopProxyFactory`内部会分情况返回基于JDK的`JdkDynamicAopProxy`或基于CGLIB的`ObjenesisCglibAopProxy`，它俩都实现了Spring的`AopProxy`接口。`AopProxy`接口中只定义了一个方法，`getProxy()`方法，Spring Aop创建的代理对象也就是该接口方法的返回结果。  

我们先来看一下基于JDK代理的`JdkDynamicAopProxy`的getProxy()的逻辑。
```java
	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	@Override
	public Object getProxy(ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}
```
我们可以看到它最终是通过JDK的`Proxy`来创建的代理，使用的`InvocationHandler`实现类是它本身，而使用的接口是`AopProxyUtils.completeProxiedInterfaces(this.advised)`的返回结果。而这个`this.advised`对象是`AdvisedSupport`类型，它是`ProxyFactory`的父类（间接通过`ProxyCreatorSupport`继承，`ProxyFactory`的直接父类是`ProxyCreatorSupport`，`ProxyCreatorSupport`的父类是`AdvisedSupport`），`AdvisedSupport`的父类是`ProxyConfig`，`ProxyConfig`中包含创建代理对象时的一些配置项信息。以下是`AopProxyUtils.completeProxiedInterfaces(this.advised)`的内部逻辑。  
```java
	public static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised) {
		Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
		if (specifiedInterfaces.length == 0) {
			// No user-specified interfaces: check whether target class is an interface.
			Class<?> targetClass = advised.getTargetClass();
			if (targetClass != null && targetClass.isInterface()) {
				specifiedInterfaces = new Class<?>[] {targetClass};
			}
		}
		boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
		boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
		int nonUserIfcCount = 0;
		if (addSpringProxy) {
			nonUserIfcCount++;
		}
		if (addAdvised) {
			nonUserIfcCount++;
		}
		Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
		System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
		if (addSpringProxy) {
			proxiedInterfaces[specifiedInterfaces.length] = SpringProxy.class;
		}
		if (addAdvised) {
			proxiedInterfaces[proxiedInterfaces.length - 1] = Advised.class;
		}
		return proxiedInterfaces;
	}
```
我们可以看到其会在`!advised.isOpaque() && !advised.isInterfaceProxied(Advised.class)`返回`true`的情况下加上本文的主角`Advised`接口。`isOpaque()`是`ProxyConfig`中的一个方法，对应的是`opaque`属性，表示是否禁止将代理对象转换为`Advised`对象，默认是`false`。`!advised.isInterfaceProxied(Advised.class)`表示将要代理的目标对象类没有实现`Advised`接口，对于我们自己应用的`Class`来说，一般都不会自己去实现`Advised`接口的，所以这个通常也是返回`true`，所以通常创建Aop代理对象时是会创建包含`Advised`接口的代理对象的，即上述的`proxiedInterfaces[proxiedInterfaces.length - 1] = Advised.class`会被执行。  
前面我们已经提到，`JdkDynamicAopProxy`创建代理对象应用的`InvocationHandler`是其自身，所以我们在调用`JdkDynamicAopProxy`创建的代理对象的任何方法时都将调用`JdkDynamicAopProxy`实现的`InvocationHandler`接口的`invoke(Object proxy, Method method, Object[] args)`方法。该方法实现如下：  
```java
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}

			// Get the interception chain for this method.
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// Check whether we have any advice. If we don't, we can fallback on direct
			// reflective invocation of the target, and avoid creating a MethodInvocation.
			if (chain.isEmpty()) {
				// We can skip creating a MethodInvocation: just invoke the target directly
				// Note that the final invoker must be an InvokerInterceptor so we know it does
				// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, args);
			} else {
				// We need to create a method invocation...
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			// Massage return value if necessary.
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				// Special case: it returned "this" and the return type of the method
				// is type-compatible. Note that we can't help if the target sets
				// a reference to itself in another returned object.
				retVal = proxy;
			} else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			if (target != null && !targetSource.isStatic()) {
				// Must have come from TargetSource.
				targetSource.releaseTarget(target);
			}
			if (setProxyContext) {
				// Restore old proxy.
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}
```
其中关于`Advised`接口方法调用最核心的一句是如下这句。我们可以看到，当我们调用的目标方法是定义自`Advised`接口时，对应方法的调用将委托给`AopUtils.invokeJoinpointUsingReflection(this.advised, method, args)`，`invokeJoinpointUsingReflection`方法的逻辑比较简单，是通过Java反射来调用目标方法。在这里`invokeJoinpointUsingReflection`传递的目标对象正是`AdvisedSupport`类型的`this.advised`对象。
```java
	if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
			method.getDeclaringClass().isAssignableFrom(Advised.class)) {
		// Service invocations on ProxyConfig with the proxy config...
		return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
	}
```
`AdvisedSupport`类是实现了`Advised`接口的，所以Spring Aop创建了基于`Advised`接口的代理对象后在调用`Advised`接口方法时可以把它委托给`AdvisedSupport`。而我们知道Spring Aop代理对象的创建正是基于`AdvisedSupport`的配置进行的（配置项主要都定义在`AdvisedSupport`的父类`ProxyConfig`类中）。创建代理对象时应用`AdvisedSupport`，调用`Advised`接口方法也用同一个实现了`Advised`接口的`AdvisedSupport`对象，所以这个过程在Spring Aop内部就可以很好的衔接。接着我们来看一下`Advised`接口的定义。  
```java
public interface Advised extends TargetClassAware {

	boolean isFrozen();

	boolean isProxyTargetClass();

	Class<?>[] getProxiedInterfaces();

	boolean isInterfaceProxied(Class<?> intf);

	void setTargetSource(TargetSource targetSource);

	TargetSource getTargetSource();

	void setExposeProxy(boolean exposeProxy);

	boolean isExposeProxy();

	void setPreFiltered(boolean preFiltered);

	boolean isPreFiltered();

	Advisor[] getAdvisors();

	void addAdvisor(Advisor advisor) throws AopConfigException;

	void addAdvisor(int pos, Advisor advisor) throws AopConfigException;

	boolean removeAdvisor(Advisor advisor);

	void removeAdvisor(int index) throws AopConfigException;

	int indexOf(Advisor advisor);

	boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;

	void addAdvice(Advice advice) throws AopConfigException;

	void addAdvice(int pos, Advice advice) throws AopConfigException;

	boolean removeAdvice(Advice advice);

	int indexOf(Advice advice);

	String toProxyConfigString();

}
```

`Advised`接口中定义的方法还是非常多的，通过它我们可以在运行时了解我们的代理对象是基于CGLIB的还是基于JDK代理的；可以了解我们的代理对应应用了哪些`Advisor`；也可以在运行时给我们的代理对象添加和删除`Advisor/Advise`。本文旨在描述Spring Aop在创建代理对象时是如何基于`Advised`接口创建代理的，以及我们能够应用`Advised`接口做哪些事。文中应用的是Spring创建基于JDK代理对象的过程为示例讲解的，其实基于CGLIB的代理也是一样的。关于CGLIB的代理过程、本文中描述的一些核心类以及本文的核心——`Advised`接口的接口方法说明等请有兴趣的朋友参考Spring的API文档和相关的源代码。  
（注：本文是基于Spring4.1.0所写，Elim写于2017年5月15日）