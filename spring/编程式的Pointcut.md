#编程式的Pointcut	
除了可以通过注解和Xml配置定义Pointcut之外，其实我们还可以通过程序来定义`Pointcut`。Spring Aop的切入点（Pointcut）对应于它的一个`Pointcut`接口，全称是`org.springframework.aop.Pointcut`。该接口的定义如下：  
```java
public interface Pointcut {

	ClassFilter getClassFilter();

	MethodMatcher getMethodMatcher();

	Pointcut TRUE = TruePointcut.INSTANCE;

}
```  
  
该接口一共定义了两个核心方法，一个用于获取该Pointcut对应的过滤Class的ClassFilter对象，一个用于获取过滤Method的MethodMatcher对象。  
ClassFilter接口的定义如下：  
```java
public interface ClassFilter {

	boolean matches(Class<?> clazz);

	ClassFilter TRUE = TrueClassFilter.INSTANCE;

}
```    
该接口只定义了一个matches方法，用于判断指定的Class对象是否匹配当前的过滤规则。     
MethodMatcher接口定义如下：
```java
public interface MethodMatcher {

	boolean matches(Method method, Class<?> targetClass);

	boolean isRuntime();

	boolean matches(Method method, Class<?> targetClass, Object[] args);

	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}

```	  
   
该接口中一共定义了三个方法，两个matches方法，一个包含方法参数一个不包含。不包含方法参数的matches方法用于判断非运行时的方法匹配，比如只需要匹配方法名、方法参数定义的；包含方法参数值的matches方法用于运行时判断方法是否匹配，应用于需要根据方法传参来判断是否匹配的情况，但是该方法一般会在不包含方法参数的matches方法返回true和isRuntime()方法true的情形下才会调用。isRuntime()方法用于指定该Pointcut是否需要在运行时才能判断对应的方法是否匹配。  
	
##自定义Pointcut
以下是一个自定义Pointcut的代码，其将匹配所有的名称Service结尾的Class对应的名称以find开始的方法调用：  
```java
import java.lang.reflect.Method;

import org.springframework.aop.ClassFilter;
import org.springframework.aop.MethodMatcher;
import org.springframework.aop.Pointcut;

/**
 * 自定义Pointcut
 * @author Elim
 * 2017年5月8日
 */
public class MyCustomPointcut implements Pointcut {

	@Override
	public ClassFilter getClassFilter() {
		return new MyCustomClassFilter();
	}

	@Override
	public MethodMatcher getMethodMatcher() {
		return new MyCustomMethodMatcher();
	}
	
	private static class MyCustomClassFilter implements ClassFilter {

		@Override
		public boolean matches(Class<?> clazz) {
			//实现自己的判断逻辑，这里简单的判断对应Class的名称是以Service结尾的就表示匹配
			return clazz.getName().endsWith("Service");
		}
		
	}
	
	private static class MyCustomMethodMatcher implements MethodMatcher {

		@Override
		public boolean matches(Method method, Class<?> targetClass) {
			//实现方法匹配逻辑
			return method.getName().startsWith("find");
		}

		@Override
		public boolean isRuntime() {
			return false;
		}

		@Override
		public boolean matches(Method method, Class<?> targetClass, Object[] args) {
			return false;
		}
		
	}

}
```   
	然后我们可以定义该自定义Pointcut对应的bean，再定义一个Advisor将使用该Pointcut。如下示例中我们就指定了将在MyCustomPointcut对应的切入点处采用LogAroundAdvice。  
```xml
 	<aop:config>
 		<aop:advisor advice-ref="logAroundAdvice" pointcut-ref="myCustomPointcut"/>
 	</aop:config>
	<bean id="logAroundAdvice" class="com.elim.learn.spring.aop.advice.LogAroundAdvice"/>
	<bean id="myCustomPointcut" class="com.elim.learn.spring.aop.pointcut.MyCustomPointcut"/>
```	   
##继承自现有的Pointcut
除了可以完全实现Pointcut接口外，我们还可以直接使用Spring自带的Pointcut。比如基于固定方法的StaticMethodMatcherPointcut。该Pointcut是一个抽象类，在使用该Pointcut时只需要实现一个抽象方法matches(Method method, Class<?> targetClass)，以下是一个继承自StaticMethodMatcherPointcut的示例类定义，该Pointcut将匹配所有Class中定义的方法名以find开头的方法。  
```java
public class FindMethodMatcherPointcut extends StaticMethodMatcherPointcut {

	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		return method.getName().startsWith("find");
	}

}
```
关于更多Spring官方已经提供的其它Pointcut定义请参考Spring的API文档。
>（注：本文是基于Spring4.1.0所写，Elim写于2017年5月8日）