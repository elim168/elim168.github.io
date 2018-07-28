# Spring之WebClient介绍

WebClient是从Spring WebFlux 5.0版本开始提供的一个非阻塞的基于响应式编程的进行Http请求的客户端工具。它的响应式编程的基于Reactor的。WebClient中提供了标准Http请求方式对应的get、post、put、delete等方法，可以用来发起相应的请求。下面的代码是一个简单的WebClient请求示例。可以通过`WebClient.create()`创建一个WebClient的实例，之后可以通过`get()`、`post()`等选择调用方式，`uri()`指定需要请求的路径，`retrieve()`用来发起请求并获得响应，`bodyToMono(String.class)`用来指定请求结果需要处理为String，并包装为Reactor的Mono对象。

```java
WebClient webClient = WebClient.create();
Mono<String> mono = webClient.get().uri("https://www.baidu.com").retrieve().bodyToMono(String.class);
mono.subscribe(System.out::println);
```

### 响应结果解析为对象

当响应的结果是JSON时，也可以直接指定为一个Object，WebClient将接收到响应后把JSON字符串转换为对应的对象。比如下面这样。

```java
WebClient webClient = WebClient.create();
Mono<User> mono = webClient.get().uri("http://localhost:8081/user/1").retrieve().bodyToMono(User.class);
User user = mono.block();
```

如果响应的结果是一个集合，则不能继续使用`bodyToMono()`，应该改用`bodyToFlux()`，然后依次处理每一个元素，比如下面这样。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);
Flux<User> userFlux = webClient.get().uri("users").retrieve().bodyToFlux(User.class);
userFlux.subscribe(System.out::println);
```

如果需要通过Flux取到所有的元素构造为一个List，则可以通过如下的方式获取。

```java
List<User> users = userFlux.collectList().block();
```

### URL中使用路径变量

URL中也可以使用路径变量，路径变量的值可以通过uri方法的第2个参数指定。下面的代码中就定义了URL中拥有一个路径变量id，然后实际访问时该变量将取值1。

```java
webClient.get().uri("http://localhost:8081/user/{id}", 1);
```

URL中也可以使用多个路径变量，多个路径变量的赋值将依次使用uri方法的第2个、第3个、第N个参数。下面的代码中就定义了URL中拥有路径变量p1和p2，实际访问的时候将被替换为var1和var2。所以实际访问的URL是`http://localhost:8081/user/var1/var2`。

```
webClient.get().uri("http://localhost:8081/user/{p1}/{p2}", "var1", "var2");
```

使用的路径变量也可以通过Map进行赋值。面的代码中就定义了URL中拥有路径变量p1和p2，实际访问的时候会从uriVariables中获取值进行替换。所以实际访问的URL是`http://localhost:8081/user/var1/1`

```java
Map<String, Object> uriVariables = new HashMap<>();
uriVariables.put("p1", "var1");
uriVariables.put("p2", 1);
webClient.get().uri("http://localhost:8081/user/{p1}/{p2}", uriVariables);
```

### 指定baseUrl

在应用中使用WebClient时也许你要访问的URL都来自同一个应用，只是对应不同的URL地址，这个时候可以把公用的部分抽出来定义为baseUrl，然后在进行WebClient请求的时候只指定相对于baseUrl的URL部分即可。这样的好处是你的baseUrl需要变更的时候可以只要修改一处即可。下面的代码在创建WebClient时定义了baseUrl为`http://localhost:8081`，在发起Get请求时指定了URL为`/user/1`，而实际上访问的URL是`http://localhost:8081/user/1`。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);
Mono<User> mono = webClient.get().uri("user/{id}", 1).retrieve().bodyToMono(User.class);
```

### Form提交

当传递的请求体对象是一个MultiValueMap对象时，WebClient默认发起的是Form提交。下面的代码中就通过Form提交模拟了用户进行登录操作，给Form表单传递了参数username，值为u123，传递了参数password，值为p123。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);

MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
map.add("username", "u123");
map.add("password", "p123");

Mono<String> mono = webClient.post().uri("/login").syncBody(map).retrieve().bodyToMono(String.class);
```

### 请求JSON

假设现在拥有一个新增User的接口，按照接口定义客户端应该传递一个JSON对象，格式如下：

```json
{
    "name":"张三",
    "username":"zhangsan"
}
```

客户端可以建立一个满足需要的JSON格式的对象，然后直接把该对象作为请求体，WebClient会帮我们自动把它转换为JSON对象。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);

User user = new User();
user.setName("张三");
user.setUsername("zhangsan");

Mono<Void> mono = webClient.post().uri("user/add").syncBody(user).retrieve().bodyToMono(Void.class);
mono.block();
```

如果没有建立对应的对象，直接包装为一个Map对象也是可以的，比如下面这样。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);

Map<String, Object> user = new HashMap<>();
user.put("name", "张三");
user.put("username", "zhangsan");

Mono<Void> mono = webClient.post().uri("user/add").syncBody(user).retrieve().bodyToMono(Void.class);
mono.block();
```

直接传递一个JSON字符串也是可以的，但是此时需要指定contentType为`application/json`，也可以加上charset。默认情况下WebClient将根据传递的对象在进行解析处理后自动选择ContentType。直接传递字符串时默认使用的ContentType会是`text/plain`。其它情况下也可以主动指定ContentType。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);

String userJson = 
        "{" + 
        "    \"name\":\"张三\",\r\n" + 
        "    \"username\":\"zhangsan\"\r\n" + 
        "}";

Mono<Void> mono = webClient.post().uri("user/add").contentType(MediaType.APPLICATION_JSON_UTF8).syncBody(userJson).retrieve().bodyToMono(Void.class);
mono.block();
```

## exchange

前面介绍的示例都是直接获取到了响应的内容，可能你会想获取到响应的头信息、Cookie等。那就可以在通过WebClient请求时把调用`retrieve()`改为调用`exchange()`，这样可以访问到代表响应结果的`org.springframework.web.reactive.function.client.ClientResponse`对象，通过它可以获取响应的状态码、Cookie等。下面的代码先是模拟用户进行了一次表单的登录操作，通过ClientResponse获取到了登录成功后的写入Cookie的sessionId，然后继续请求了用户列表。在请求获取用户列表时传递了存储了sessionId的Cookie。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.create(baseUrl);

MultiValueMap<String, String> map = new LinkedMultiValueMap<>();
map.add("username", "u123");
map.add("password", "p123");

Mono<ClientResponse> mono = webClient.post().uri("login").syncBody(map).exchange();
ClientResponse response = mono.block();
if (response.statusCode() == HttpStatus.OK) {
    Mono<Result> resultMono = response.bodyToMono(Result.class);
    resultMono.subscribe(result -> {
        if (result.isSuccess()) {
            ResponseCookie sidCookie = response.cookies().getFirst("sid");
            Flux<User> userFlux = webClient.get().uri("users").cookie(sidCookie.getName(), sidCookie.getValue()).retrieve().bodyToFlux(User.class);
            userFlux.subscribe(System.out::println);
        }
    });
}
```

## WebClient.Builder

除了可以通过`WebClient.create()`创建WebClient对象外，还可以通过`WebClient.builder()`创建一个`WebClient.Builder`对象，再对Builder对象进行一些配置后调用其`build()`创建WebClient对象。下面的代码展示了其用法，配置了baseUrl和默认的cookie信息。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.builder().baseUrl(baseUrl).defaultCookie("cookieName", "cookieValue").build();
```

Builder还可以通过`clientConnector()`定义需要使用的ClientHttpConnector，默认将使用`org.springframework.http.client.reactive.ReactorClientHttpConnector`，其底层是基于netty的，如果你使用的是Maven，需要确保你的pom.xml中定义了如下依赖。

```xml
<dependency>
    <groupId>io.projectreactor.ipc</groupId>
    <artifactId>reactor-netty</artifactId>
    <version>0.7.8.RELEASE</version>
</dependency>
```

如果对默认的发送请求和处理响应结果的编解码不满意，还可以通过`exchangeStrategies()`定义使用的ExchangeStrategies。ExchangeStrategies中定义了用来编解码的对象，其也有对应的`build()`方法方便我们来创建ExchangeStrategies对象。

WebClient也提供了Filter，对应于`org.springframework.web.reactive.function.client.ExchangeFilterFunction`接口，其接口方法定义如下。

```java
Mono<ClientResponse> filter(ClientRequest request, ExchangeFunction next)
```

在进行拦截时可以拦截request，也可以拦截response。下面的代码定义的Filter就拦截了request，给每个request都添加了一个名为header1的header，值为value1。它也拦截了response，response中也是添加了一个新的header信息。拦截response时，如果新的ClientResponse对象是通过`ClientResponse.from(response)`创建的，新的response是不会包含旧的response的body的，如果需要可以通过`ClientResponse.Builder`的`body()`指定，其它诸如header、cookie、状态码是会包含的。

```java
String baseUrl = "http://localhost:8081";
WebClient webClient = WebClient.builder().baseUrl(baseUrl).filter((request, next) -> {
    ClientRequest newRequest = ClientRequest.from(request).header("header1", "value1").build();
    Mono<ClientResponse> responseMono = next.exchange(newRequest);
    return Mono.fromCallable(() -> {
        ClientResponse response = responseMono.block();
        ClientResponse newResponse = ClientResponse.from(response).header("responseHeader1", "Value1").build();
        return newResponse;
    });
}).build();
```

如果定义的Filter只期望对某个或某些request起作用，可以在Filter内部通过request的相关属性进行拦截，比如cookie信息、header信息、请求的方式或请求的URL等。也可以通过`ClientRequest.attribute(attrName)`获取某个特定的属性，该属性是在请求时通过`attribute("attrName", "attrValue")`指定的。这跟在HttpServletRequest中添加的属性的作用范围是类似的。

关于WebClient使用的更多信息请参考其API文档。

## 参考文档

[https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client)

（注：本文是基于Spring WebFlux 5.0.7所写）