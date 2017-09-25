# Java Webservice指定超时时间

使用JDK对Webservice的支持进行Webservice调用时通常的操作步骤如下：
```java
//1、创建一个javax.xml.ws.Service实例
javax.xml.ws.Service service = javax.xml.ws.Service.create(wsdl, serviceName);
//2、通过Service实例获取对应的服务接口的代理
HelloService helloService = service.getPort(portName, HelloService.class);
//3、通过获取到的Webservice服务接口的代理调用对应的服务方法
helloService.sayHello("Elim")
```

在上述的步骤一在构建Service实例的同时，在Service内部会构建一个ServiceDelegate类型的对象赋给属性delegate，内部持有。然后在第二步会利用delegate创建一个服务接口的代理对象，同时还会代理BindingProvider和Closeable接口。然后在第三步真正发起接口请求时，内部会发起一个HTTP请求，发起HTTP请求时会从BindingProvider的getRequestContext()返回结果中获取超时参数，分别对应com.sun.xml.internal.ws.connection.timeout和com.sun.xml.internal.ws.request.timeout参数，前者是建立连接的超时时间，后者是获取请求响应的超时时间，单位是毫秒。如果没有指定对应的超时时间或者指定的超时时间为0都表示永不超时。所以为了指定超时时间我们可以从BindingProvider下手。比如：
```java
public class Client {

	public static void main(String[] args) throws Exception {
		String targetNamespace = "http://test.elim.com/ws";
		QName serviceName = new QName(targetNamespace, "helloService");
		QName portName = new QName(targetNamespace, "helloService");
		URL wsdl = new URL("http://localhost:8888/hello");
		//内部会创建一个ServiceDelegate类型的对象赋给属性delegate
		Service service = Service.create(wsdl, serviceName);
		//会利用delegate创建一个服务接口的代理对象，同时还会代理BindingProvider和Closeable接口。
		HelloService helloService = service.getPort(portName, HelloService.class);
		
		
		BindingProvider bindingProvider = (BindingProvider) helloService;
		Map<String, Object> requestContext = bindingProvider.getRequestContext();
		requestContext.put("com.sun.xml.internal.ws.connection.timeout", 10 * 1000);//建立连接的超时时间为10秒
		requestContext.put("com.sun.xml.internal.ws.request.timeout", 15 * 1000);//指定请求的响应超时时间为15秒
		
		//在调用接口方法时，内部会发起一个HTTP请求，发起HTTP请求时会从BindingProvider的getRequestContext()返回结果中获取超时参数，
		//分别对应com.sun.xml.internal.ws.connection.timeout和com.sun.xml.internal.ws.request.timeout参数，
		//前者是建立连接的超时时间，后者是获取请求响应的超时时间，单位是毫秒。如果没有指定对应的超时时间或者指定的超时时间为0都表示永不超时。
		
		System.out.println(helloService.sayHello("Elim"));
	}
	
}
```

完整示例如下：  
服务接口：
```java
@WebService(portName="helloService", serviceName="helloService", targetNamespace="http://test.elim.com/ws")
public interface HelloService {

	String sayHello(String name);
	
}
```

服务接口实现：
```java
@WebService(portName="helloService", serviceName="helloService", targetNamespace="http://test.elim.com/ws")
public class HelloServiceImpl implements HelloService {

	private Random random = new Random();
	
	@Override
	public String sayHello(String name) {
		try {
			TimeUnit.SECONDS.sleep(5 + random.nextInt(21));//随机睡眠5-25秒
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		return "Hello " + name;
	}

}
```

服务端代码：
```java
public class Server {

	public static void main(String[] args) {
		Endpoint.publish("http://localhost:8888/hello", new HelloServiceImpl());
	}
	
}
```

在上述的服务端代码中随机睡眠了5-25秒，而客户端指定的超时时间是15秒，所以在测试的时候你会看到有时候服务调用会超时，有时会正常响应。  

（本文由Elim写于2017年9月25日）
