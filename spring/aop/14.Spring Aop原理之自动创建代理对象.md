# Aop自动创建代理对象的原理
我们在使用Spring Aop时，通常Spring会自动为我们创建目标bean的代理对象，以使用对应的`Advisor`。前提是我们在使用Spring Aop时是使用的`<aop:config/>`或`<aop:aspectj-autoproxy/>`，这是因为当我们在applicationContext.xml文件中通过`<aop:config/>`的形式定义需要使用Aop的场景时，Spring会自动为我们添加`AspectjAwareAdvisorAutoProxyCreator`类型的bean；而我们定义了`<aop:aspectj-autoproxy/>`时，Spring会默认为我们添加`AnnotationAwareAspectjAutoProxyCreator`类型的bean。Spring中在bean实例化后能够对bean对象进行包装的是`BeanPostProcessor`，`AspectjAwareAdvisorAutoProxyCreator`和`AnnotationAwareAspectjAutoProxyCreator`都是实现了`BeanPostProcessor`接口的。`AnnotationAwareAspectjAutoProxyCreator`的父类是`AspectjAwareAdvisorAutoProxyCreator`，而`AspectjAwareAdvisorAutoProxyCreator`的父类是`AbstractAdvisorAutoProxyCreator`，`AbstractAdvisorAutoProxyCreator`的父类是实现了`BeanPostProcessor`接口的`AbstractAutoProxyCreator`。它们的核心逻辑都是在bean初始化后找出bean容器中所有的能够匹配当前bean的`Advisor`，找到了则将找到的`Advisor`通过`ProxyFactory`创建该bean的代理对象返回。`AspectjAwareAdvisorAutoProxyCreator`在寻找候选的`Advisor`时会找到bean容器中所有的实现了`Advisor`接口的bean，而`AnnotationAwareAspectjAutoProxyCreator`则在`AspectjAwareAdvisorAutoProxyCreator`的基础上增加了对标注了`@Aspect`的bean的处理，会附加上通过`@Aspect`标注的bean中隐式定义的`Advisor`。所以这也是为什么我们在使用`@Aspect`标注形式的Spring Aop时需要在applicationContext.xml文件中添加`<aop:aspectj-autoproxy/>`。既然`AspectjAwareAdvisorAutoProxyCreator`和`AnnotationAwareAspectjAutoProxyCreator`都会自动扫描bean容器中的`Advisor`，所以当我们使用了`<aop:config/>`或`<aop:aspectj-autoproxy/>`形式的Aop定义时，如果因为某些原因光通过配置满足不了你Aop的需求，而需要自己实现`Advisor`接口时（一般是实现`PointcutAdvisor接口`），那这时候你只需要把自己的`Advisor`实现类，定义为Spring的一个bean即可。如果你在applicationContext.xml中没有定义`<aop:config/>`或`<aop:aspectj-autoproxy/>`，那你也可以直接在applicationContext.xml中直接定义`AspectjAwareAdvisorAutoProxyCreator`或`AnnotationAwareAspectjAutoProxyCreator`类型的bean，效果也是一样的。其实为了能够在创建目标bean的时候能够自动创建基于我们自定义的`Advisor`实现类的代理对象，我们的bean容器中只要有`AbstractAutoProxyCreator`类型的bean定义即可，当然了你实现自己的`BeanPostProcessor`，在其`postProcessAfterInitialization`方法中创建自己的代理对象也是可以的。但是本着不重复发明轮子的原则，我们尽量使用官方已经提供好的实现即可。<font color="red">`AbstractAutoProxyCreator`是继承自`ProxyConfig`的，所以我们在定义`AbstractAutoProxyCreator`子类的bean时，我们也可以手动的定义一些`ProxyConfig`中包含的属性，比如`proxyTargetClass`、`exposeProxy`等。</font>`AbstractAutoProxyCreator`的子类除了`AspectjAwareAdvisorAutoProxyCreator`和`AnnotationAwareAspectjAutoProxyCreator`外，我们可用的还有`BeanNameAutoProxyCreator`和`DefaultAdvisorAutoProxyCreator`。  

# BeanNameAutoProxyCreator
`BeanNameAutoProxyCreator`可以用来定义哪些bean可与哪些`Advisor/Advice`绑定，以生成对应的代理对象。需要绑定的bean是通过`beanNames`属性来指定的，对应的是bean的名称，其中可以包含“\*”号，表示任意字符，比如“abc\*”则匹配任意名称以“abc”开始的bean；需要绑定的`Advisor/Advice`是通过`interceptorNames`来指定的，如果指定的是`Advisor`，那么是否可生成基于该`Advisor`的代理需要对应的bean能够匹配对应`Advisor`（`PointcutAdvisor`类型）的`Pointcut`；如果指定的是`Advice`，则该`Advice`会被包含在`Pointcut`恒匹配的`Advisor`中，即能够与所有的bean绑定生成对应的代理，且会对所有的方法调用起作用。  指定interceptorNames时是不能使用通配符的，只能精确的指定需要应用的`Advisor/Advice`对应的bean名称。
```xml
	<bean id="userService" class="com.elim.learn.spring.aop.service.UserServiceImpl"/>
	<bean id="myService" class="com.elim.learn.spring.aop.service.MyService"/>
	
	<bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
		<!-- 匹配userService和所有名称以my开头的bean -->
		<property name="beanNames" value="userService, my*"/>
		<!-- interceptorNames中不能使用通配符，只能是精确匹配，即精确指定Advisor/Advice的bean名称 -->
		<property name="interceptorNames" value="logBeforeAdvice, myAdvisor "/>
	</bean>
	
	<bean id="logBeforeAdvice" class="com.elim.learn.spring.aop.advice.LogBeforeAdvice" />
	 
	<bean id="myAdvisor" class="com.elim.learn.spring.aop.advisor.MyAdvisor"/>
```
如上就是一个使用`BeanNameAutoProxyCreator`建立指定的bean基于指定的`Advisor/Advice`的代理对象的示例。示例中我们指定interceptorNames时特意应用了一个`Advisor`实现和一个`Advice`实现。`Advice`会应用于所有绑定的bean的所有方法调用，而`Advisor`只会应用于其中的Pointcut能够匹配的方法调用。这里的源码我就不提供了，有兴趣的朋友可以自己试试。  

# DefaultAdvisorAutoProxyCreator
`DefaultAdvisorAutoProxyCreator`的父类也是`AbstractAdvisorAutoProxyCreator`。`DefaultAdvisorAutoProxyCreator`的作用是会默认将bean容器中所有的`Advisor`都取到，如果有能够匹配某一个bean的`Advisor`存在，则会基于能够匹配该bean的所有`Advisor`创建对应的代理对象。需要注意的是`DefaultAdvisorAutoProxyCreator`在创建bean的代理对象时是不会考虑`Advice`的，只是`Advisor`。如上面的示例中，如果我们希望所有的bean都能够自动的与匹配的`Advisor`进行绑定生成对应的代理对象，那么我们可以调整配置如下。
```xml
	<bean id="userService" class="com.elim.learn.spring.aop.service.UserServiceImpl"/>
	<bean id="myService" class="com.elim.learn.spring.aop.service.MyService"/>
	
	<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>
	
	<bean id="myAdvisor" class="com.elim.learn.spring.aop.advisor.MyAdvisor"/>
```
使用`DefaultAdvisorAutoProxyCreator`时可能我们并不希望为所有的bean定义都自动应用bean容器中的所有`Advisor`，而只是希望自动创建基于部分`Advisor`的代理对象。这个时候如果我们期望应用自动代理的`Advisor`的bean定义的名称都是拥有固定的前缀时，则我们可以应用`DefaultAdvisorAutoProxyCreator`的`setAdvisorBeanNamePrefix(String)`指定需要应用的`Advisor`的bean名称的前缀，同时需要通过`setUsePrefix(boolean)`指定需要应用这种前缀匹配机制。如我们的bean容器中有两个`Advisor`定义，一个bean名称是“myAdvisor”，一个bean名称是“youAdvisor”，如果只期望自动创建基于bean名称以“my”开始的Advisor的代理，则可以进行如下配置。
```xml
	<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
		<property name="usePrefix" value="true" />
		<!-- 匹配所有bean名称以my开始的Advisor -->
		<property name="advisorBeanNamePrefix" value="my" />
	</bean>
```

（注：本文是基于Spring4.1.0所写，Elim写于2017年5月12日）