# Spring之RestTemplate介绍

RestTemplate是Spring Web模块提供的一个基于Rest规范提供Http请求的工具。应用中如果需要访问第三方提供的Rest接口，使用RestTemplate操作将非常方便。RestTemplate中提供了一系列的getXXX、postXXX、putXXX、deleteXXX等方法，以供发起对应的Rest规范请求，以及更通用的exchange和execute系列方法。方法详情可以参考对应的API文档或源码。

#### 发起GET请求

通过`getForObject`可以发起GET请求，下面的代码中就展示了通过`getForObject`发起GET请求，第二个参数指定返回类型，指定String则表示以文本形式进行返回。

```java
RestTemplate restTemplate = new RestTemplate();
String response = restTemplate.getForObject("https://www.so.com/s?ie=utf-8&q=中国", String.class);
System.out.println(response);
```

#### 使用路径变量

URI中还可以使用路径变量。下面的代码中URI中就包含了一个路径变量`word`，路径变量可以作为最后一个参数传递，RestTemplate将自动进行替换。所以最终请求的URL是`https://www.so.com/s?ie=utf-8&q=ABC`。

```java
RestTemplate restTemplate = new RestTemplate();
String word = "ABC";
String response = restTemplate.getForObject("https://www.so.com/s?ie=utf-8&q={word}", String.class, word);
System.out.println(response);
```

当路径变量有多个时，可以按照顺序在最后依次传递。下面的代码中就定义了两个路径变量，charset和word，然后传递路径变量时，第一个参数值`utf-8`将传递给定义的第一个路径变量charset，第二个参数值`ABC`将传递给定义的第二个路径变量word。所以最终请求的URL将是`https://www.so.com/s?ie=utf-8&q=ABC`。

```java
RestTemplate restTemplate = new RestTemplate();
String uri = "https://www.so.com/s?ie={charset}&q={word}";
String response = restTemplate.getForObject(uri, String.class, "utf-8", "ABC");
System.out.println(response);
```

路径变量也可以通过Map传递，这样将更加精确，尤其是在有路径变量相同的情况下。下面的代码中就展示了如何通过Map传递路径变量。

```java
RestTemplate restTemplate = new RestTemplate();
String uri = "https://www.so.com/s?ie={charset}&q={word}";
Map<String, String> uriVariables = new HashMap<>();
uriVariables.put("charset", "utf-8");
uriVariables.put("word", "ABC");
String response = restTemplate.getForObject(uri, String.class, uriVariables);
System.out.println(response);
```

#### 响应内容使用ResponseEntity接收

响应内容也可以通过ResponseEntity接收，这样可以访问到响应的Http状态码和头信息。

```java
RestTemplate restTemplate = new RestTemplate();
ResponseEntity<String> responseEntity = restTemplate.getForEntity("https://www.so.com/s?ie=utf-8&q=ABC", String.class);
if (responseEntity.getStatusCode().equals(HttpStatus.OK)) {
    System.out.println("Get Success");
}

System.out.println("Headers: ");
HttpHeaders headers = responseEntity.getHeaders();
headers.forEach((name, values) -> {
    System.out.println(name + " = " + values);
});

if (responseEntity.hasBody()) {
    System.out.println("response: ");
    System.out.println(responseEntity.getBody());
}
```

#### 返回复杂对象

RestTemplate的请求响应对象不仅仅可以是String，也可以是一个复杂对象。发起请求的过程中的请求内容和响应内容都会经过`org.springframework.http.converter.HttpMessageConverter`进行处理，这跟Spring MVC中是一样的。

```java
RestTemplate restTemplate = new RestTemplate();
User user = restTemplate.getForObject("http://localhost:8081/user/1", User.class);
System.out.println(user);
```

RestTemplate中默认会添加如下HttpMessageConverter，其中if语句块包含的HttpMessageConverter将在Classpath下拥有指定的Class时才会添加。从下面的代码可以看出，如果需要请求的Object能够自动转换为JSON对象进行请求或者响应的JSON对象能够自动转换为Object，可以加入Jackson依赖，这样将自动加入MappingJackson2HttpMessageConverter。

```java
public RestTemplate() {
    this.messageConverters.add(new ByteArrayHttpMessageConverter());
    this.messageConverters.add(new StringHttpMessageConverter());
    this.messageConverters.add(new ResourceHttpMessageConverter(false));
    this.messageConverters.add(new SourceHttpMessageConverter<>());
    this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());

    if (romePresent) {
        this.messageConverters.add(new AtomFeedHttpMessageConverter());
        this.messageConverters.add(new RssChannelHttpMessageConverter());
    }

    if (jackson2XmlPresent) {
        this.messageConverters.add(new MappingJackson2XmlHttpMessageConverter());
    }
    else if (jaxb2Present) {
        this.messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
    }

    if (jackson2Present) {
        this.messageConverters.add(new MappingJackson2HttpMessageConverter());
    }
    else if (gsonPresent) {
        this.messageConverters.add(new GsonHttpMessageConverter());
    }
    else if (jsonbPresent) {
        this.messageConverters.add(new JsonbHttpMessageConverter());
    }

    if (jackson2SmilePresent) {
        this.messageConverters.add(new MappingJackson2SmileHttpMessageConverter());
    }
    if (jackson2CborPresent) {
        this.messageConverters.add(new MappingJackson2CborHttpMessageConverter());
    }
}
```

#### 添加自己的HttpMessageConverter

RestTemplate自带的HttpMessageConverter可能并不能完全满足你的需求。我们可以往HttpMessageConverter队列中添加自己的HttpMessageConverter。比如如果JSON处理你更喜欢使用Alibaba Fastjson，则可以添加FastJsonHttpMessageConverter到HttpMessageConverter列表，如果列表中已经拥有了可以处理JSON的HttpMessageConverter，则需要把它添加到原来处理JSON的HttpMessageConverter的前面。

```java
RestTemplate restTemplate = new RestTemplate();
restTemplate.getMessageConverters().add(0, new FastJsonHttpMessageConverter());
User user = restTemplate.getForObject("http://localhost:8081/user/1", User.class);
System.out.println(user);
```

> 添加HttpMessageConverter到HttpMessageConverter列表可以通过获取已有的列表再添加，也可以通过`setMessageConverters`方法指定，前者用于追加，后者将进行完整的替换。

已有的处理String的StringHttpMessageConverter默认使用的字符集是ISO-8859-1，这对使用中文的我们来说可能不合适，所以可以添加一个基于UTF-8字符集的StringHttpMessageConverter到现有的HttpMessageConverter列表中，然后需要保证在已有的StringHttpMessageConverter的前面。

```java
RestTemplate restTemplate = new RestTemplate();
restTemplate.getMessageConverters().add(0, new StringHttpMessageConverter(Charset.forName("UTF-8")));
String request = "你好";
Class<String> responseType = String.class;
String result = restTemplate.postForObject("http://localhost:8081/hello/string", request, responseType);
System.out.println(result);
```

#### 请求内容使用HttpEntity对象

在请求时如果需要指定Header信息，则可以将请求内容包装为一个HttpEntity对象。下面的代码中就传递了两个Header，但是没有传递其它body内容，如果需要传递body也是可以的，HttpBody拥有对应的重载的构造方法。

```java
RestTemplate restTemplate = new RestTemplate();
HttpHeaders headers = new HttpHeaders();
headers.add("abc", "123");//自定义Header
headers.add("Content-Type", "text/plain");//标准Header
HttpEntity<Void> httpEntity = new HttpEntity<>(headers);
Class<String> responseType = String.class;
String result = restTemplate.postForObject("http://localhost:8081/hello/header", httpEntity, responseType);
System.out.println(result);
```

RestTemplate的没有提供传递HttpEntity对象的getXXX方法，如果在进行GET请求时需要传递HttpEntity对象怎么办呢？这个时候可以使用万能的exchange方法。下面的代码使用了exchange方法进行了GET请求，同时通过HttpEntity定义了一些头信息。

```java
RestTemplate restTemplate = new RestTemplate();
HttpHeaders headers = new HttpHeaders();
headers.add("abc", "123");//自定义Header
headers.add("Content-Type", "text/plain");//标准Header
HttpEntity<Void> httpEntity = new HttpEntity<>(headers);
Class<String> responseType = String.class;

ResponseEntity<String> responseEntity = restTemplate.exchange("http://localhost:8081/hello/header", HttpMethod.GET, httpEntity, responseType);

System.out.println(responseEntity.getBody());
```

## 使用Apache HttpComponents实现

RestTemplate默认使用的是基于JDK的Http请求实现，RestTemplate中发起Http请求的操作被封装到了ClientHttpRequestFactory接口，默认使用的是`org.springframework.http.client.SimpleClientHttpRequestFactory`实现。Spring同样提供了基于Apache HttpComponents的实现类，`org.springframework.http.client.HttpComponentsClientHttpRequestFactory`。其它实现类可以参考API文档。使用Apache HttpComponents实现需要加入相应依赖。

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.6</version>
</dependency>
```

可以通过RestTemplate的`setRequestFactory()`指定需要使用的ClientHttpRequestFactory。使用Apache HttpComponents的实现的好处是可以利用其内部的连接池实现。下面的代码展示了如何使用基于Apache HttpComponents的实现进行编程。

```java
RequestConfig requestConfig = RequestConfig.custom().setConnectTimeout(10*1000).setConnectionRequestTimeout(30*1000).build();
HttpClient httpClient = HttpClientBuilder.create().setMaxConnPerRoute(20).setMaxConnTotal(50).setDefaultRequestConfig(requestConfig).build();
HttpComponentsClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory(httpClient);

RestTemplate restTemplate = new RestTemplate();
restTemplate.setRequestFactory(requestFactory);

ResponseEntity<String> responseEntity = restTemplate.getForEntity("http://localhost:8081/hello/json", String.class);

System.out.println(responseEntity);
```

> 如果需要在客户端进行异步的Http请求，还可以使用AsyncRestTemplate。不过它从Spring5开始已经废弃了。

（注：本文基于Spring4.1.0所写）

