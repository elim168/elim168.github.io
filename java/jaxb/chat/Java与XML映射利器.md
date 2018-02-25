# Java与XML映射利器

熟悉Hibernate的朋友都知道，它可以把Java类和数据库表进行映射，通过操作Java对象的方式可以对表记录进行更新。这可以大大增加我们的开发效率，免去自己直接通过JDBC操作数据库表的繁琐过程。其实Mybatis也是类似的，只不过它是半自动的，需要自己写SQL。在利用Java开发基于XML的操作时你会不会也想要一款可以直接基于Java类建立对应的XML的映射关系，然后可以直接通过Java对象转换为对应的XML，或者可以直接通过XML转换为对应的Java对象的工具呢？如果你正在寻找这样一款工具，那么JAXB可以满足你的需求。JAXB是Java提供的一款Java和XML绑定（映射）的工具，建立起了对应的关系后，就可以直接通过一个简单的API就可以实现Java对象和XML之间的相互转换。

假设现在有一个User类，其结构如下，需要把它的对象转换为根节点为user的XML，其中User对象的每一个属性需要映射为XML的user节点下的一个字节点。那么我们只需要简单的在User类上标注@XmlRootElement即可，这表示类User需要映射为XML的一个根节点，对应的节点名称是user（这是JAXB的默认策略，默认取类名称的首字母小写形式作为根节点名称），如果不希望使用默认名称，也可以通过name属性指定自己想要的名称，就像下面示例中那样。就加一个注解就搞定了，是不是非常简单呢？

```java
@XmlRootElement(name="user")
public class User {
    
    private Integer id;
    private String name;
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    
}
```

这时候我们使用下面的代码来进行测试，最核心的代码就是`JAXB.marshal(user, System.out)`，它表示需要把user对象转换为对应的XML形式，并把它输出到控制台（System.out），其中的JAXB类是Java自带的一个JAXB工具类。
```java
@Test
public void testXml() throws Exception {
    //构造User对象
    User user = new User();
    user.setId(1);
    user.setName("张三");
    
    JAXB.marshal(user, System.out);
    
}
```

应用上面的测试代码会在控制台输出如下XML内容。是不是非常简单的就把Java对象转换为XML了。如果我们是基于DOM或者dom4j等编程，则我们需要一步步的调用很多的API，相比而言，这种基于注解进行映射的方式明显简单很多。
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<user>
    <id>1</id>
    <name>张三</name>
</user>
```

如果我们的根节点还是user，但是User类的id属性我们不想把它作为user节点下面的一个字节点，而是作为user节点的一个属性，这种也是很简单的，直接在getId()方法上加上@XmlAttribute注解即可。
```java
@XmlRootElement(name="user")
public static class User {
    
    private Integer id;
    private String name;
    @XmlAttribute
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    
}
```

这个时候再运行上面的测试代码，生成的XML将是如下这样。
```xml
@XmlRootElement(name="user")
public static class User {
    
    private Integer id;
    private String name;
    @XmlAttribute
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    
}
```

如果User类的id属性希望作为user节点的一个属性，但是属性名称不希望使用默认的id，则可以通过@XmlAttribute的name属性指定一个我们想要的名称，比如下面指定的就是id1。
```java
@XmlRootElement(name="user")
public static class User {
    
    private Integer id;
    private String name;
    @XmlAttribute(name="id1")
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    
}
```

这时候生成的XML会是如下这样。
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<user id1="1">
    <name>张三</name>
</user>
```

同样的，如果name属性映射的节点名称不希望是name，希望是name1,则我们可以通过@XmlElement(name="name1")来指定。
```java
@XmlRootElement(name="user")
public static class User {
    
    private Integer id;
    private String name;
    @XmlAttribute(name="id1")
    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    @XmlElement(name="name1")
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    
}
```

这时候生成的XML会是如下这样。
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<user id1="1">
    <name1>张三</name1>
</user>
```

上面介绍的都是定义了Java类与XML的映射关系后，把Java对象转换为XML的示例。接下来我们来看如下示例中把XML转换为Java对象。
```java
@Test
public void testXml() throws Exception {
    String xml = "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>\r\n" + 
            "<user id1=\"1\">\r\n" + 
            "    <name1>张三</name1>\r\n" + 
            "</user>";
    User user = JAXB.unmarshal(new StringReader(xml), User.class);
    Assert.assertTrue(user.getId() == 1);
    Assert.assertEquals("张三", user.getName());
}
```

是不是非常简单呢？以上只是介绍了JAXB的一些最基本的操作，想要了解更多内容可以参考笔者的[JAXB开发详解](http://gitbook.cn/gitchat/column/5a210d8a39fa666a31fb0984)，可以让你收获更多。
