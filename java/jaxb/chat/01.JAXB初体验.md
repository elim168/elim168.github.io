# JAXB初体验

** 摘要 **

本文主要通过对比使用JAXB和非JAXB进行Java对象转XML和XML转Java对象的方式来介绍JAXB的基本功能，让大家对JAXB有一个初步的体会。


考虑下我们有如下这样两个Class，Person和Address，其中Person持有一个Address的引用。
```java
public class Person {
    private Integer id;
    private String name;
    private Integer age;
    private Address address;

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

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }

}

public class Address {
    private Integer id;
    private String province;
    private String city;
    private String area;
    private String other;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getProvince() {
        return province;
    }

    public void setProvince(String province) {
        this.province = province;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public String getArea() {
        return area;
    }

    public void setArea(String area) {
        this.area = area;
    }

    public String getOther() {
        return other;
    }

    public void setOther(String other) {
        this.other = other;
    }
}
```

## 1 Java对象转XML

假设我们现在需要把一个Person类型的对象生成如下格式的XML：
```xml
<person id="1" name="张三">
    <address id="1">
        <area>南山区</area>
        <city>深圳市</city>
        <other>其它</other>
        <province>广东省</province>
    </address>
    <age>30</age>
</person>
```

### 1.1 基于DOM实现
在没有使用JAXB之前，比如使用Dom生成XML的方式，我们得如下编程：
```java
@Test
public void geneByDom() throws Exception {
    Person person = this.buildPerson();
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();  
    DocumentBuilder db = factory.newDocumentBuilder();  
    Document document = db.newDocument();  
    Element personEle = document.createElement("person");  
    personEle.setAttribute("id", person.getId().toString());
    personEle.setAttribute("name", person.getName());
    Element ageEle = document.createElement("age");
    ageEle.setTextContent(person.getAge().toString());
    personEle.appendChild(ageEle);
    
    Address address = person.getAddress();
    Element addressEle = document.createElement("address");
    addressEle.setAttribute("id", address.getId().toString());
    Element province = document.createElement("province");
    province.setTextContent(address.getProvince());
    addressEle.appendChild(province);
    
    Element city = document.createElement("city");
    city.setTextContent(address.getCity());
    addressEle.appendChild(city);
    
    Element area = document.createElement("area");
    area.setTextContent(address.getArea());
    addressEle.appendChild(area);
    
    Element other = document.createElement("other");
    other.setTextContent(address.getOther());
    addressEle.appendChild(other);
    
    personEle.appendChild(addressEle);
    document.appendChild(personEle);  
    
    TransformerFactory transformerFactory = TransformerFactory.newInstance();  
    Transformer transformer = transformerFactory.newTransformer();  
    Source xmlSource = new DOMSource(document);  
    //把生成的XML输出到控制台 
    Result outputTarget = new StreamResult(System.out);  
    transformer.transform(xmlSource, outputTarget);  
}
```

### 1.2 基于JAXB实现

基于JAXB实现时我们需要通过JAXB提供的注解来控制它的一些行为，比如下面我们通过在Person类上指定@XmlRootElement指定它为一个根节点，对应的根节点名称将默认取类名称，然后首字母小写，在本示例中即取“person”。通过@XmlAttribute指定id和name作为节点person的XML属性，可以通过XmlAttribute的name属性指定生成的XML属性的名称，如getId()方法上的@XmlAttribute(name="id")；没有指定name属性时对应的XML属性名称将使用Java属性名称，如getName()方法上的@XmlAttribute。
```java
@XmlRootElement
public class Person {
    private Integer id;
    private String name;
    private Integer age;
    private Address address;

    @XmlAttribute(name = "id")
    public Integer getId() {
       return id;

    }

    public void setId(Integer id) {
        this.id = id;
    }

    @XmlAttribute
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }
    
}

public class Address {
    private Integer id;
    private String province;
    private String city;
    private String area;
    private String other;

    @XmlAttribute(name = "id")
    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getProvince() {
        return province;
    }

    public void setProvince(String province) {
        this.province = province;
    }

    public String getCity() {
        return city;
    }

    public void setCity(String city) {
        this.city = city;
    }

    public String getArea() {
        return area;
    }

    public void setArea(String area) {
        this.area = area;
    }

    public String getOther() {
        return other;
    }

    public void setOther(String other) {
        this.other = other;
    }
}
```

想要生成目标格式的XML，基于Class的配置就好了。接下来就是调用对应的API进行XML的转换了。通过如下几行代码就可以把一个Person对象生成我们需要的XML格式了。如果还有其它的类型的对象也需要生成XML我们只需要把下面代码中的对象和对应的类型更换一下即可，而同样的需求在基于Dom的转换中我们又得重新写一遍代码了，另外从代码层面我们可以明显的看出基于JAXB的方式比基于DOM的方式要节约很多代码，实现起来也要简单很多，它的XML绑定行为都是通过对应的注解来完成的，非常方便。

```java
@Test
public void testMarshal() throws JAXBException {
    JAXBContext context = JAXBContext.newInstance(Person.class);
    Marshaller marshaller = context.createMarshaller();
    marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true);
    StringWriter writer = new StringWriter();
    //构造Person对象，不是本文的重点
    Person person = this.buildPerson();

    marshaller.marshal(person, writer);
    System.out.println(writer.toString());
}
```

 
JAXB的全称是Java Architecture for XML Binding，是一项可以通过XML产生Java对象，也可以通过Java对象产生XML的技术。是JDK自带的功能。JDK中关于JAXB部分有几个比较重要的接口或类，如：
* JAXBContext：它是程序的入口类，提供了XML/Java绑定的操作，包括创建Marshaller和Unmarshaller等。
* Marshaller：它负责把Java对象序列化为对应的XML。
* Unmarshaller：它负责把XML反序列化为对应的Java对象。
 

使用JAXB进行对象转XML的基本操作步骤如下：
```java
//1、获取一个基于某个class的JAXBContext，即JAXB上下文
JAXBContext jaxbContext = JAXBContext.newInstance(obj.getClass());
//2、利用JAXBContext对象创建对应的Marshaller实例。
Marshaller marshaller = jaxbContext.createMarshaller();
//3、设置一些序列化时需要的指定的配置
marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, Boolean.TRUE);
marshaller.setProperty(Marshaller.JAXB_FRAGMENT, Boolean.TRUE);
StringWriter writer = new StringWriter();
//4、将对象进行序列化
marshaller.marshal(obj, writer);
```

* 1.创建一个JAXB上下文对象，可以传递序列化时需要用到的一些Class，以便JAXB在序列化对应的对象时能够识别其中关联的Class。
* 2.利用JAXB上下文对象创建对应的Marshaller对象。
* 3.指定序列化时的配置参数，具体可以设置的参数和对应的参数的含义可以参考API文档。
* 4.最后一步是将对应的对象序列化到一个Writer、OutputStream、File等输出对象中。


Marshaller接口中定义了很多marshal方法，可以满足我们不同的需要，具体方法定义如下，详情可参考对应的API文档。
```java
public void marshal(Object jaxbElement, javax.xml.transform.Result result) throws JAXBException;

public void marshal(Object jaxbElement, java.io.OutputStream os) throws JAXBException;

public void marshal(Object jaxbElement, File output) throws JAXBException;

public void marshal(Object jaxbElement, java.io.Writer writer) throws JAXBException;

public void marshal(Object jaxbElement, org.xml.sax.ContentHandler handler) throws JAXBException;

public void marshal(Object jaxbElement, org.w3c.dom.Node node) throws JAXBException;

public void marshal(Object jaxbElement, javax.xml.stream.XMLStreamWriter writer) throws JAXBException;

public void marshal(Object jaxbElement, javax.xml.stream.XMLEventWriter writer) throws JAXBException;
```

产生了Marshaller后，我们可以通过其`setProperty( String name, Object value )`方法设置一些属性，以控制XML的生成。常用的有如下属性：
* Marshaller.JAXB_ENCODING：指定生成的XML的字符集，并不是头信息上的那个encoding。
* Marshaller.JAXB\_FORMATTED\_OUTPUT：指定是否需要对生成的XML进行格式化，默认是不格式化的。
* Marshaller.JAXB_FRAGMENT：指定是否需要生成文档级别的事件，指定对应的属性值为true可以不生成XML的头信息。


使用JAXB进行对象的序列化时对应的对象类型必须是javax.xml.bind.JAXBElement（JAXBElement是JAXB用来表示XML元素的一个对象，其中包含了XML元素相关的一些信息）类型，或者是使用了javax.xml.bind.annotation.XmlRootElement注解标注的类型。XmlRootElement用于标注在class上面，表示把一个class映射为一个XML Element对象，而且通常是对应的XML的根元素。与之相配合使用的注解通常还有XmlElement和XmlAttribute等。XmlElement注解用于标注在class的属性上，用于把一个class的属性映射为一个XML Element对象。XmlAttribute注解用于标注在class的属性上，用于把一个class的属性映射为其class对应的XML Element上的一个属性。默认情况下，当我们的一个属性没有使用XmlElement标注时（更精确的说是没有使用JAXB注解标注时，除了XmlElement注解以外，JAXB还有很多其它的注解）也会被序列化为Xml元素的一个子元素，元素名会取java属性名，这也是为什么在上面的示例中我们不需要在一些java属性或get方法上加@XmlElement注解，它们也能转换为XML元素的原因。使用@XmlElement通常更多的需求是用于指定XML元素的名称，比如在上面的示例中如果我们的address属性对应的节点名称我们不希望取默认的"address"，而是希望取"addr"，那么我们就可以在对应的getAddress()方法上加上@XmlElement(name="addr")，即通过XmlElement的name属性来指定生成的XML元素的名称。
```java
@XmlRootElement
public class Person {
    private Integer id;
    private String name;
    private Integer age;
    private Address address;

    @XmlAttribute(name = "id")
    public Integer getId() {
       return id;

    }

    public void setId(Integer id) {
        this.id = id;
    }

    @XmlAttribute
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @XmlElement(name="addr")
    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }
    
}
```

生成的XML会是如下这样：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person id="1" name="张三">
    <addr id="1">
        <area>南山区</area>
        <city>深圳市</city>
        <other>其它</other>
        <province>广东省</province>
    </addr>
    <age>30</age>
</person>
```


既然默认情况下没有使用JAXB注解进行标注的属性（包括对应的get方法）会自动被映射为XML的一个元素，那么问题来了，当我们的Java类中有些属性是功能辅助性的，是在生成XML时不需要转换为对应的XML节点时怎么办呢？比如上面的Person在转换为XML时我们不希望age属性被转换为XML，那么默认情况下（随着后面内容的介绍你会发现还有其它方式也可以满足这种需求）我们可以在对应的属性的get方法上加上@XmlTransient注解，这样在转换XML时JAXB就会忽略该属性了。需要忽略age属性，加上@XmlTransient注解后，我们的Person类定义会是如下这个样子：
```java
@XmlRootElement
public class Person {
    private Integer id;
    private String name;
    private Integer age;
    private Address address;

    @XmlAttribute(name = "id")
    public Integer getId() {
       return id;

    }

    public void setId(Integer id) {
        this.id = id;
    }

    @XmlAttribute
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @XmlTransient
    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    public Address getAddress() {
        return address;
    }

    public void setAddress(Address address) {
        this.address = address;
    }
    
}
```

基于它转换出来的XML代码如下，我们可以看到与之前的XML相比确实是少了age节点。
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person id="1" name="张三">
    <address id="1">
        <area>南山区</area>
        <city>深圳市</city>
        <other>其它</other>
        <province>广东省</province>
    </address>
</person>
```


## 2 XML转Java对象

假设应用的类还是上面的Person和Address，XML还是那段XML。现需要基于XML转换为对应的Java对象，我们再来看看基于非JAXB的实现和基于JAXB的实现。

### 2.1 基于非JAXB实现

基于非JAXB的实现我们还是选择上面的DOM方式。代码大概会是如下这样，每个属性都得通过解析Element获得，如果对应的XML结构是动态变化的，那么解析过程还会更复杂。
```java
@Test
public void unmarshalByDom() throws Exception {
    
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();  
    //step2:获得DocumentBuilder  
    DocumentBuilder db = factory.newDocumentBuilder();  
    //step3:把需要解析的xml文件加载到一个document对象中  
    InputStream source = this.getClass().getClassLoader().getResourceAsStream("jaxb/person.xml");
    Document document = db.parse(source);  
    Element personEle = (Element) document.getChildNodes().item(0);
    
    Person person = new Person();
    String id = personEle.getAttribute("id");
    person.setId(Integer.parseInt(id));
    person.setName(personEle.getAttribute("name"));
    String age = personEle.getElementsByTagName("age").item(0).getTextContent();
    person.setAge(Integer.parseInt(age));
    
    Element addressEle = (Element) personEle.getElementsByTagName("address").item(0);
    Address address = new Address();
    person.setAddress(address);
    String addressId = addressEle.getAttribute("id");
    address.setId(Integer.parseInt(addressId));
    String province = addressEle.getElementsByTagName("province").item(0).getTextContent();
    address.setProvince(province);
    String city = addressEle.getElementsByTagName("city").item(0).getTextContent();
    address.setCity(city);
    String area = addressEle.getElementsByTagName("area").item(0).getTextContent();
    address.setArea(area);
    String other = addressEle.getElementsByTagName("other").item(0).getTextContent();
    address.setOther(other);
    System.out.println(person);
    
}
```


### 2.2 基于JAXB的实现

基于JAXB实现时，我们应用的Java类和对应的JAXB注解与Java转XML示例中的一致，基于这个配置我们只需要如下代码即可实现XML转Java对象。
```java
@Test
public void testUnmarshal() throws Exception {
    JAXBContext jaxbContext = JAXBContext.newInstance(Person.class);
    Unmarshaller unmarshaller = jaxbContext.createUnmarshaller();
    InputStream source = this.getClass().getClassLoader().getResourceAsStream("jaxb/person.xml");
    Person person = (Person) unmarshaller.unmarshal(source);
    System.out.println(person);
}
```

进行XML转Java的基本步骤如下：
```java
//1、创建一个指定class的JAXB上下文对象
JAXBContext context = JAXBContext.newInstance(Person.class);
//2、通过JAXBContext对象创建对应的Unmarshaller对象。
Unmarshaller unmarshaller = context.createUnmarshaller();
InputStream source = this.getClass().getClassLoader().getResourceAsStream("jaxb/person.xml");
//3、调用Unmarshaller对象的unmarshal方法进行反序列化，接收的参数可以是一个InputStream、Reader、File等
Person person = (Person) unmarshaller.unmarshal(source);
```
 
Unmarshaller对象中提供了一系列的unmarshal重载方法，对应的参数类型可以是File、InputStream、Reader等，以满足我们在不同情况下的需要，详情可参考对应的API文档。
```java
public Object unmarshal(java.io.File f) throws JAXBException;

public Object unmarshal(java.io.InputStream is) throws JAXBException;

public Object unmarshal(Reader reader) throws JAXBException;

public Object unmarshal(java.net.URL url) throws JAXBException;

public Object unmarshal(org.xml.sax.InputSource source) throws JAXBException;

public Object unmarshal(org.w3c.dom.Node node) throws JAXBException;

public <T> JAXBElement<T> unmarshal(org.w3c.dom.Node node, Class<T> declaredType) throws JAXBException;

public Object unmarshal(javax.xml.transform.Source source) throws JAXBException;

public <T> JAXBElement<T> unmarshal(javax.xml.transform.Source source, Class<T> declaredType)
        throws JAXBException;

public Object unmarshal(javax.xml.stream.XMLStreamReader reader) throws JAXBException;

public <T> JAXBElement<T> unmarshal(javax.xml.stream.XMLStreamReader reader, Class<T> declaredType)
        throws JAXBException;

public Object unmarshal(javax.xml.stream.XMLEventReader reader) throws JAXBException;

public <T> JAXBElement<T> unmarshal(javax.xml.stream.XMLEventReader reader, Class<T> declaredType)
        throws JAXBException;
```

 
### JAXB工具类
除了使用JAXBContext来创建Marshaller和Unmarshaller对象来实现Java对象和XML之间的互转外，Java还为我们提供了一个工具类JAXB。JAXB工具类提供了一系列的静态方法来简化了Java对象和XML之间的互转，具体的方法定义请参考对应的API文档。通过它我们在进行XML和Java互转时有时会非常方便，只需要简单的一行代码即可搞定。
```java
@Test
public void testMarshal1() {
    Person person = new Person();
    person.setId(1);
    person.setName("张三");
    person.setAge(30);
    Address address = new Address();
    address.setId(1);
    address.setProvince("广东省");
    address.setCity("深圳市");
    address.setArea("南山区");
    address.setOther("其它");
    person.setAddress(address);
    JAXB.marshal(person, System.out);
}

@Test
public void testUnmarshal1() {
    InputStream source = this.getClass().getClassLoader().getResourceAsStream("jaxb/person.xml");
    Person person = JAXB.unmarshal(source, Person.class);
    System.out.println(person);
}
```

以上就是本文的全部内容，是JAXB的入门介绍。
 
