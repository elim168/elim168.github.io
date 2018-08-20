# ImportSelector介绍

在`@Configuration`标注的Class上可以使用`@Import`引入其它的配置类，其实它还可以引入`org.springframework.context.annotation.ImportSelector`实现类。ImportSelector接口只定义了一个`selectImports()`，用于指定需要注册为bean的Class名称。当在`@Configuration`标注的Class上使用`@Import`引入了一个ImportSelector实现类后，会把实现类中返回的Class名称都定义为bean。来看一个简单的示例，假设现在有一个接口HelloService，需要把所有它的实现类都定义为bean，而且它的实现类上是没有加Spring默认会扫描的注解的，比如`@Component`、`@Service`等。

```java
public interface HelloService {

    void doSomething();
    
}

public class HelloServiceA implements HelloService {

    @Override
    public void doSomething() {
        System.out.println("Hello A");
    }

}

public class HelloServiceB implements HelloService {

    @Override
    public void doSomething() {
        System.out.println("Hello B");
    }

}
```

现定义了一个ImportSelector实现类HelloImportSelector，直接指定了需要把HelloService接口的实现类HelloServiceA和HelloServiceB定义为bean。

```java
public class HelloImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] {HelloServiceA.class.getName(), HelloServiceB.class.getName()};
    }

}
```

然后定义了`@Configuration`配置类HelloConfiguration，指定了`@Import`的是HelloImportSelector。

```java
@Configuration
@Import(HelloImportSelector.class)
public class HelloConfiguration {

}
```

这样当加载配置类HelloConfiguration的时候会一并把HelloServiceA和HelloServiceB注册为Spring bean。可以进行如下简单测试：

```java
@ContextConfiguration(classes=HelloConfiguration.class)
@RunWith(SpringRunner.class)
public class HelloImportSelectorTest {

    @Autowired
    private List<HelloService> helloServices;
    
    @Test
    public void test() {
        this.helloServices.forEach(HelloService::doSomething);
    }
    
}
```

看到这里可能你会觉得其实它也没什么用，因为整一个ImportSelector实现类那么麻烦，还不如直接在HelloConfiguration中定义bean或者import。在不引入ImportSelector的情况下，下面的两种方式都可以达到相同的效果。

```java
@Configuration
@Import({HelloServiceA.class, HelloServiceB.class})
public class HelloConfiguration {
    
}
```


```java
@Configuration
public class HelloConfiguration {

    @Bean
    public HelloServiceA helloServiceA() {
        return new HelloServiceA();
    }
    
    @Bean
    public HelloServiceB helloServiceB() {
        return new HelloServiceB();
    }
    
}
```

如果直接是固定的bean定义，那完全可以用上面的方式代替，但如果需要动态的带有逻辑性的定义bean，则使用ImportSelector还是很有用处的。因为在它的`selectImports()`你可以实现各种获取bean Class的逻辑，通过其参数`AnnotationMetadata importingClassMetadata`可以获取到`@Import`标注的Class的各种信息，包括其Class名称，实现的接口名称、父类名称、添加的其它注解等信息，通过这些额外的信息可以辅助我们选择需要定义为Spring bean的Class名称。现假设我们在HelloConfiguration上使用了`@ComponentScan`进行bean定义扫描，我们期望HelloImportSelector也可以扫描`@ComponentScan`指定的Package下HelloService实现类并把它们定义为bean，则HelloImportSelector和HelloConfiguration可以改为如下这样：

```java
@Configuration
@ComponentScan("com.elim.spring.core.importselector")
@Import(HelloImportSelector.class)
public class HelloConfiguration {

    
}
```


```java
public class HelloImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(ComponentScan.class.getName());
        String[] basePackages = (String[]) annotationAttributes.get("basePackages");
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        Set<String> classes = new HashSet<>();
        for (String basePackage : basePackages) {
            scanner.findCandidateComponents(basePackage).forEach(beanDefinition -> classes.add(beanDefinition.getBeanClassName()));
        }
        return classes.toArray(new String[classes.size()]);
    }

}
```

可以看到在HelloImportSelector的实现中获取了HelloConfiguration类上标注的`@ComponentScan`的basePackages属性值，并使用ClassPathScanningCandidateComponentProvider进行了扫描。可能有的时候你不希望依赖于配置类上的`@ComponentScan`，而期望直接扫描配置类所在的包。此时可以通过`importingClassMetadata.getClassName()`获取配置类的Class名称，进而获取其package名称。

```java
public class HelloImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        String packageName = null;
        try {
            packageName = Class.forName(importingClassMetadata.getClassName()).getPackage().getName();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        String[] basePackages = new String[] {packageName};
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        Set<String> classes = new HashSet<>();
        for (String basePackage : basePackages) {
            scanner.findCandidateComponents(basePackage).forEach(beanDefinition -> classes.add(beanDefinition.getBeanClassName()));
        }
        return classes.toArray(new String[classes.size()]);
    }

}
```

更通用一点的做法可能你还是期望扫描的package跟`@Configuration`上的`@ComponentScan`的basePackages保持一致或者在没有指定`@ComponentScan`时扫描配置类所在的package。`@ComponentScan`的basePackages如果没有指定，默认是把配置类当前所在的package当做basePackage。所以为了满足这些需求，我们的HelloImportSelector可以定义为如下这样：

```java
public class HelloImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        String[] basePackages = null;
        if (importingClassMetadata.hasAnnotation(ComponentScan.class.getName())) {
            Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(ComponentScan.class.getName());
            basePackages = (String[]) annotationAttributes.get("basePackages");
        }
        if (basePackages == null || basePackages.length == 0) {//ComponentScan的basePackages默认为空数组
            String basePackage = null;
            try {
                basePackage = Class.forName(importingClassMetadata.getClassName()).getPackage().getName();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            basePackages = new String[] {basePackage};
        }
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        Set<String> classes = new HashSet<>();
        for (String basePackage : basePackages) {
            scanner.findCandidateComponents(basePackage).forEach(beanDefinition -> classes.add(beanDefinition.getBeanClassName()));
        }
        return classes.toArray(new String[classes.size()]);
    }

}
```

## 为ImportSelector定义特定的注解

当我们觉得在`@Configuration`配置类上使用`@Import(HelloImportSelector.class)`太麻烦，或者是需要在ImportSelector实现类中使用一些特定的配置时就可以考虑为ImportSelector实现类定义一个特定的注解，在该注解上使用`@Import(HelloImportSelector.class)`。如下针对上面的HelloImportSelector定义了一个`@HelloServiceScan`注解，用于扫描HelloService实现类。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(HelloImportSelector.class)
public @interface HelloServiceScan {

}
```

此时，我们的HelloConfiguration类可以改为如下这样，效果跟之前的一样的。

```java
@Configuration
@HelloServiceScan
public class HelloConfiguration {

}
```

有了自定义注解后，就可以定义自定义注解的属性，以供在扫描bean时进行一些特殊的配置。比如可以把扫描的路径定义到自定义的注解中，而不必依赖于`@ComponentScan`。下面的代码中就为`@HelloServiceScan`自定义了属性basePackages和value，它俩互为别名。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(HelloImportSelector.class)
public @interface HelloServiceScan {

    @AliasFor("value")
    String[] basePackages() default {};
    
    @AliasFor("basePackages")
    String[] value() default {};
    
}
```

这样HelloImportSelector在进行bean扫描时可以通过`@HelloServiceScan`的basePackages属性获取需要扫描的basePackage。

```java
public class HelloImportSelector implements ImportSelector {

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        Map<String, Object> annotationAttributes = importingClassMetadata.getAnnotationAttributes(HelloServiceScan.class.getName());
        String[] basePackages = (String[]) annotationAttributes.get("basePackages");
        if (basePackages == null || basePackages.length == 0) {//HelloServiceScan的basePackages默认为空数组
            String basePackage = null;
            try {
                basePackage = Class.forName(importingClassMetadata.getClassName()).getPackage().getName();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            basePackages = new String[] {basePackage};
        }
        ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
        TypeFilter helloServiceFilter = new AssignableTypeFilter(HelloService.class);
        scanner.addIncludeFilter(helloServiceFilter);
        Set<String> classes = new HashSet<>();
        for (String basePackage : basePackages) {
            scanner.findCandidateComponents(basePackage).forEach(beanDefinition -> classes.add(beanDefinition.getBeanClassName()));
        }
        return classes.toArray(new String[classes.size()]);
    }

}
```

在`@Configuration`配置类上就可以为`@HelloServiceScan`指定额外的basePackages属性了。

```java
@Configuration
@HelloServiceScan("com.elim.spring.core.importselector")
public class HelloConfiguration {

}
```


（注：本文基于Spring 5.0.7所写）