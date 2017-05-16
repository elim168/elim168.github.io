# ProxyFactoryBean创建代理对象
`ProxyFactoryBean`实现了Spring的`FactoryBean`接口，所以它跟Spring中的其它`FactoryBean`一样，都是基于工厂模式来获取一个bean的。`ProxyFactoryBean`就是用来获取一个对象的代理对象的`FactoryBean`。它也是继承自`ProxyCreatorSupport`类的，所以它的功能基本跟`ProxyFactory`差不多，只是`ProxyFactory`是用于编程式的创建代理对象。而`ProxyFactoryBean`用于在Spring的bean容器中创建基于bean的代理对象。通常一个简单的`ProxyFactoryBean`配置大概会是如下这样。
```xml
	<bean id="proxyFactoryBeanTestService" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="target"><!-- 指定被代理的对象 -->
			<bean class="com.elim.learn.spring.aop.service.ProxyFactoryBeanTestService"/>
		</property>
		<property name="proxyTargetClass" value="true"/><!-- 指定启用基于Class的代理 -->
		<property name="interceptorNames"><!-- 指定生成的代理对象需要绑定的Advice或Advisor在bean容器中的名称 -->
			<list>
				<value>logAroundAdvice</value>
			</list>
		</property>
	</bean>
```
在上面的示例中我们被代理的对象对应的Class是`com.elim.learn.spring.aop.service.ProxyFactoryBeanTestService`，其是一个没有实现任何接口的Class，所以我们生成的代理对象最终会是基于CGLIB的代理。我们需要指定`proxyTargetClass="true"`，以表示我们是倾向于使用CGLIB代理的，对于上面的配置实际上就是告诉Spring我们要使用CGLIB代理。虽然这里我们不指定`proxyTargetClass="true"`时，Spring可能也会给我们使用CGLIB代理，为什么这里说是可能呢？因为`ProxyFactoryBean`默认生成的bean都是单例、且在生成bean时会自动检测被代理对象实现的接口，而且`proxyTargetClass`默认是false，这种情况下`ProxyFactoryBean`就会自动检测被代理对象的实现的接口。按理来说我们的bean是一个没有实现接口的bean，Spring给我们去找它实现的接口是找不出来的，但是我们知道Spring的Aop是会自动为我们的对象实现一些接口的。简单的说如果我们的bean容器中配置了其它的Advisor，那么我们指定的target对象有可能就不是一个原始的bean，而是一个已经被Aop代理过的bean对象，这种bean对象，Spring Aop默认会为其实现一个`Advised`接口。所以使用`ProxyFactoryBean`时，如果我们的代理对象类是没有实现接口的，或者我们期望生成代理对象时是基于Class的，而不是基于Interface的，我们最好明确的指定`proxyTargetClass="true"`，而不是寄希望于Spring的自动决定机制。  指定被代理对象时，除了可以直接指定target外，我们还可以通过targetName指定被代理对象在bean容器中的bean名称。
在上面的示例中我们通过interceptorNames属性指定了生成的代理对象需要应用的Advisor/Advice对应于bean容器中的bean的名称。跟ProxyFactory一样，如果我们指定的是Advice对象，则其会转换为一个匹配所有方法调用的Advisor与代理对象绑定。  在指定intercepterNames时我们也可以通过“\*”指定所有beanName以XX开始的Advisor/Advice，如我们的bean容器中同时拥有“abc1Advisor”、“abc2Advisor”两个Advisor，我们期望创建的ProxyFactoryBean同时应用这两个Advisor，那我们可以不用单独指定两次，而是一次性把interceptorNames指定的一个beanName为“abc\*”。需要注意的是“\*”只能定义在beanName的末端。
```xml
	<bean id="proxyFactoryBeanTestService" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="target"><!-- 指定被代理的对象 -->
			<bean class=" com.elim.learn.spring.aop.service.ProxyFactoryBeanTestService"/>
		</property>
		<property name="proxyTargetClass" value="true"/><!-- 指定启用基于Class的代理 -->
		<property name="interceptorNames"><!-- 指定生成的代理对象需要绑定的Advice或Advisor在bean容器中的名称 -->
			<list>
				<value>abc*</value>
			</list>
		</property>
	</bean>
```
  
**其它配置信息**  
- **exposeProxy**：属于从`ProxyCreatorSupport`继承过来的属性，用于定义是否需要在调用代理对象时把代理对象发布到`AopContext`，默认是`false`。
- **singleton**：用来指定`ProxyFactoryBean`生成的bean是否是单例的，默认是`true`。该值对应于`FactoryBean`的isSingleton()接口方法的返回值。
- **frozen**：属于从`ProxyCreatorSupport`继承过来的属性，用于指定代理对象被创建后是否还允许更改代理配置，通过`Advised`接口更改。`true`表示不允许，默认是false。
- **autodetectInterfaces**：表示是否在生成代理对象时需要启用自动检测被代理对象实现的接口，默认是`true`。
- **proxyInterfaces**：基于接口的代理时指定需要代理的接口。
- **interfaces**：基于接口的代理时指定需要代理的接口，属于从`ProxyCreatorSupport`继承过来的。
关于`ProxyFactoryBean`的更多配置项信息可以参考Spring的API文档或`ProxyFactoryBean`的源代码。  
（注：本文是基于Spring4.1.0所写，Elim写于2017年5月10日）