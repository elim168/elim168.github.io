# ImportBeanDefinitionRegistrar介绍

在上一篇博文[http://elim.iteye.com/blog/2428994](http://elim.iteye.com/blog/2428994)中介绍了ImportSelector的作用及其用法。本文需要介绍的ImportBeanDefinitionRegistrar的用法和作用跟ImportSelector类似。唯一的不同点是ImportBeanDefinitionRegistrar的接口方法`void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry)`的返回类型是`void`，且多了一个BeanDefinitionRegistry类型的参数，它允许我们直接通过BeanDefinitionRegistry对象注册bean。承接上文内容，为了扫描并注册HelloService类型的bean，我们可以自定义如下ImportBeanDefinitionRegistrar实现类。在实现类中可以使用ClassPathBeanDefinitionScanner进行扫描并自动注册，它是ClassPathScanningCandidateComponentProvider的子类，所以还是可以添加相同的TypeFilter，然后通过`scanner.scan(basePackages)`扫描指定的basePackage下满足条件的Class并注册它们为bean。

```java
public class HelloImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(ComponentScan.class.getName());
        String[] basePackages = (String[]) annotationAttributes.get("basePackages");
        
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry, false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        scanner.scan(basePackages);
    }

}
```

此时我们的`@Configuration`配置类可以进行如下定义。

```java
@Configuration
@ComponentScan("com.elim.spring.core.importselector")
@Import(HelloImportBeanDefinitionRegistrar.class)
public class HelloConfiguration {

}
```

为它定义一个特定的注解也是可以的，比如下面代码为HelloImportBeanDefinitionRegistrar定义了`@HelloScan`，其value属性和basePackages属性互为别名，用于指定需要扫描的basePackage。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(HelloImportBeanDefinitionRegistrar.class)
public @interface HelloScan {

    @AliasFor("value")
    String[] basePackages() default {};
    
    @AliasFor("basePackages")
    String[] value() default {};
    
}
```

为了满足`@HelloScan`指定扫描的basePackage的需求，我们的HelloImportBeanDefinitionRegistrar需要改造为如下这样。

```java
public class HelloImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(HelloScan.class.getName());
        String[] basePackages = (String[]) annotationAttributes.get("basePackages");
        if (basePackages == null || basePackages.length == 0) {//HelloScan的basePackages默认为空数组
            String basePackage = null;
            try {
                basePackage = Class.forName(importingClassMetadata.getClassName()).getPackage().getName();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            basePackages = new String[] {basePackage};
        }
        
        ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(registry, false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        scanner.scan(basePackages);
    }

}
```

此时我们的HelloConfiguration可以定义为如下这样，它的效果和之前是一模一样的。

```java
@Configuration
@HelloScan("com.elim.spring.core.importselector")
public class HelloConfiguration {

}
```

（注：本文是基于Spring 5.0.7所写）