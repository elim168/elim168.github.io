# 基于正则表达式的Pointcut
## JdkRegexpMethodPointcut
Spring官方为我们提供了一个基于正则表达式来匹配方法名的Pointcut，`JdkRegexpMethodPointcut`。该Pointcut是继承自`StaticMethodMatcherPointcut`的。我们在定义`JdkRegexpMethodPointcut`时可以通过`patterns`和`excludedPatterns`来注入需要满足和排除的正则表达式，它们对应的都是一个`String[]`。比如我们想匹配所有的方法名以`find`开头的方法，我们可以如下定义：
```xml
	<bean id="regexPointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">
		<property name="patterns">
	        <list>
	            <value>find.*</value><!-- 所有方法名以find开始的方法 -->
	        </list>
	    </property>
	</bean>
```
如果我们需要匹配或需要排除的正则表达式只是单一的一个正则表达式，那么我们也可以通过`pattern`和`excludedPattern`来指定单一的需要匹配和排除的正则表达式。需要注意的是`patterns`和`pattern`不能同时使用，`excludedPattern`和`excludedPatterns`也是一样的。当我们同时指定了`patterns`和`excludedPatterns`时，该`Pointcut`将先匹配`patterns`，对于能够匹配`patterns`的将再判断其是否在`excludedPatterns`中，如果存在也将不匹配。以下是该匹配逻辑的核心代码。      
``` java
	protected boolean matchesPattern(String signatureString) {
		for (int i = 0; i < this.patterns.length; i++) {
			boolean matched = matches(signatureString, i);
			if (matched) {
				for (int j = 0; j < this.excludedPatterns.length; j++) {
					boolean excluded = matchesExclusion(signatureString, j);
					if (excluded) {
						return false;
					}
				}
				return true;
			}
		}
		return false;
	}
```  
>需要说明的是在上面的匹配逻辑中传递的参数signatureString是对应方法的全路径名称，即包含该方法的类的全路径及该方法的名称，如“org.springframework.aop.support.JdkRegexpMethodPointcut.matches”这种，所以如果我们需要在使用正则表达式定义Pointcut时，也可以匹配某某类的某某方法这种形式。  

## RegexpMethodPointcutAdvisor
使用了`JdkRegexpMethodPointcut`后，我们在使用的时候通常会进行如下配置。  
```xml
 	<aop:config>
  		<aop:advisor advice-ref="logBeforeAdvice" pointcut-ref="regexPointcut"/>
 	</aop:config>
	<bean id="logBeforeAdvice" class="com.elim.learn.spring.aop.advice.LogBeforeAdvice" />
	<bean id="regexPointcut" class="org.springframework.aop.support.JdkRegexpMethodPointcut">
		<property name="patterns">
	        <list>
	            <value>find.*</value><!-- 所有方法名以find开始的方法 -->
	        </list>
	    </property>
	</bean>
```
其实针对于`JdkRegexpMethodPointcut`，Spring为我们提供了一个简便的`Advisor`定义，可以让我们同时指定一个`JdkRegexpMethodPointcut`和其需要对应的`Advice`，那就是`RegexpMethodPointcutAdvisor`，我们可以给它注入一个Advice和对应需要匹配的正则表达式（pattern或patterns注入）。  
```xml
	<bean class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
		<property name="advice" ref="logBeforeAdvice"/>
		<property name="pattern" value="find.*"/>
	</bean>
```
>需要注意的是`RegexpMethodPointcutAdvisor`没有提供不匹配的正则表达式注入方法，即没有excludedPattern和excludedPatterns注入，如果需要该功能请还是使用`JdkRegexpMethodPointcut`。  

（本文是基于Spring4.1.0所写，Elim写于2017年5月8日）