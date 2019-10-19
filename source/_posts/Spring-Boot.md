---
title: Spring Boot
date: 2019-07-26 00:02:37
tags:
- Java
- Web
categories: Spring
---

# 1 SpringApplication
## SpringApplication
```java
SpringApplication.run(XXX.class, args);
```

```java
new SpringApplicationBuilder(SpringBootPracticeApplication.class)
				.properties("server.port=0")  // bind to a random port
				.run(args);
```

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {  // introduced since spring framework 3.1
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM,
				classes = AutoConfigurationExcludeFilter.class) })
@ConfigurationPropertiesScan
public @interface SpringBootApplication {
    ...
}
```
<!--more-->
## `@Component`:

处理类 -> ConfigurationClassParser

扫描类 -> 
- ClassPathBeanDefinitionScanner
  - ClassPathScanningCandidateComponentProvider
    - 
    ```java
    protected void registerDefaultFilters() {
		this.includeFilters.add(new AnnotationTypeFilter(Component.class));
        ...
    }
    ```

- @Service
- @Repository
- @Controller
- @Configuration

## Stereotype Annotations


## SpringApplication 类型判断

`SpringApplication()` -> `WebApplicationType.deduceFromClasspath();`
- `WebApplicationType.NONE` 非Web类型
- `WebApplicationType.REACTIVE`：Spring WebFlux
  - `spring-boot-starter-webflux`
- `WebApplicationType.SERVLET`：Servlet
  - `spring-boot-starter-web`

## SpringBoot事件

```java
public class SpringEventListenerDemo {

	public static void main(String[] args) {
		GenericApplicationContext context = new GenericApplicationContext();

		// add listener
		context.addApplicationListener(new ApplicationListener<ApplicationEvent>() {
			@Override
			public void onApplicationEvent(ApplicationEvent event) {
				System.out.println("Listening Event: " + event);
			}
		});

		context.addApplicationListener(new ClosedListener());
		// start context
		// one event is org.springframework.context.event.ContextRefreshedEvent
		context.refresh();
		// one event is org.springframework.context.PayloadApplicationEvent
		context.publishEvent("HelloWorld");  // 发布一个HelloWorld内容的事件
		// one event is com.example.springbootpractice.SpringEventListenerDemo$MyEvent
		context.publishEvent(new MyEvent("HelloWorld 123"));
		// event is org.springframework.context.event.ContextClosedEvent
		context.close();

	}

	private static class MyEvent extends ApplicationEvent {
		public MyEvent(Object source) {
			super(source);
		}
	}

	private static class ClosedListener implements ApplicationListener<ContextClosedEvent> {
		@Override
		public void onApplicationEvent(ContextClosedEvent event) {
			System.out.println("Destory context " + event);
		}
	}
}
```
- Spring事件的类型`ApplicationEvent`  -> 消息
- Spring事件监听器`ApplicationListener`  -> 消息消费者
- Spring事件广播器`ApplicationEventMulticaster` -> 消息生产者
  - implementation: SimpleApplicationEventMulticaster();

```java
public class SpringEventMulticasterDemo {
	public static void main(String[] args) {
		ApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
		multicaster.addApplicationListener(event -> {
			System.out.println("Accepted event: " + event);
		});
		multicaster.multicastEvent(new PayloadApplicationEvent<>("src1", "Hello world"));
		multicaster.multicastEvent(new PayloadApplicationEvent<>("src2", "Hello world"));
	}
}
```

```java
@EnableAutoConfiguration
public class SpringBootEventDemo {
	public static void main(String[] args) {
		new SpringApplicationBuilder(SpringBootEventDemo.class)
				.listeners(event -> {
					System.err.println("get event: " + event.getClass().getSimpleName());
				}) // add listeners
				.run(args)
				.close();
	}
}
```

# 2. REST in Spring


RPC
* 语言相关
  * Java - RMI
  * .NET - COM+
* 语言无关
  * SOA
    * Web Services
      * SOAP (传输介质)
        * HTTP, SMTP(通讯协议)
  * 微服务
    * REST
      * HTML, XML, JSON
      * HTTP (通讯协议)
        * HTTP 1.1
          * 短链接
          * Keep-Alive
          * 连接池
          * Long Polling
        * WebSockets
        * HTTP/2
          * 长链接

      * 技术
        * Spring客户端： RestTemplate
        * Spring WebMVC： @RestController = @Controller + @ResponseBody + @RequestBody
        @ Spring Cloud： RestTemplate扩展 + @LoadBalanced

* HTTP状态码(org.springframework.http.HttpStatus)
  * 200
  * 304
  * 400
  * 404
  * 500

## Uniform interface 
- URI vs URL

```java
  	@Override
	protected void service(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
		if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
			processRequest(request, response);
		}
		else {
			super.service(request, response);
		}
	}
```

## Cacheability
`@ResponseBody` -> 响应体
- 响应 Response
  - 响应头（Headers）
    - 元信息（meta-data）
      - Accept-Language -> `Locale`
      - Connection -> Keep-Alive
    - 实现 
      - 多值map `MultiValueMap`
      - Name: Value => 1: N
      - 
      ```java
        public class HttpHeaders implements MultiValueMap<String, String>, Serializable {
        }
      ```
- 响应体
  - 业务信息
  - Body：Http实体，Rest
    - `@ResponseBody`
    - `HttpEntity.body`
  - Payload：消息JMS，事件，SOAP

```java
public class HttpEntity<T> {
	public static final HttpEntity<?> EMPTY = new HttpEntity<>();
	private final HttpHeaders headers;
	@Nullable
    private final T body;
}
```
  
`ResponseEntity` extends `HttpEntity`
`RequestEntity` extends `HttpEntity`

