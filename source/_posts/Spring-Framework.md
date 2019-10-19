---
title: Spring Framework
date: 2019-07-19 21:48:15
tags:
- Java
- Spring
categories: Spring
---

# Spring Components

* Spring Core
  * provide core utilities and commons stuff
* Spring Context
  * provides application context, IoC
* Spring Web
* Spring MVC
  * spring mvc depends on spring web.
* Spring DAO
* Spring ORM
* Spring AOP

# 配置文件封装

Spring的配置文件由ClassPathResource进行封装

```java
public interface InputStreamSource {

	/**
	 * Return an {@link InputStream} for the content of an underlying resource.
	 * <p>It is expected that each call creates a <i>fresh</i> stream.
	 * <p>This requirement is particularly important when you consider an API such
	 * as JavaMail, which needs to be able to read the stream multiple times when
	 * creating mail attachments. For such a use case, it is <i>required</i>
	 * that each {@code getInputStream()} call returns a fresh stream.
	 * @return the input stream for the underlying resource (must not be {@code null})
	 * @throws java.io.FileNotFoundException if the underlying resource doesn't exist
	 * @throws IOException if the content stream could not be opened
	 */
	InputStream getInputStream() throws IOException;

}
```


```java

/**
 * Interface for a resource descriptor that abstracts from the actual
 * type of underlying resource, such as a file or class path resource.
 */
public interface Resource extends InputStreamSource {
	boolean exists();
	...
}
```

Different source has different implementation to Resource. FileSystemResource and ClassPathResource.



# Register bean into IoC

## Use xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="person" class="com.app.spring.Person">
        <property name="name" value="Name" />
        <property name="age" value="12" />
    </bean>
</beans>
```
```java
public class MainTest1 {
    public static void main(String[] args) {
        ApplicationContext ac = new ClassPathXmlApplicationContext("beans.xml");
        Person person = (Person) ac.getBean("person");
        System.out.println(person);
    }
}
```
<!--more-->
## Use annotation

### @Bean
```java
public class MainTest2 {
    public static void main(String[] args) {
        ApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
        Person person = (Person) ac.getBean("person01");
        System.out.println(person);


        String[] namesForBean = ac.getBeanNamesForType(Person.class);
        for(String name: namesForBean) {
            System.out.println(name);
        }
    }
}
```

```java
@Configuration
public class MainConfig {
    @Bean("beanName")
    public Person person01() {
        return new Person("Name", 14);
    }
}
```

### @ComponentScan

Just find Controller annotated bean.
```java
@Configuration
@ComponentScan(value = "com.app.spring2", includeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION, classes = {Controller.class})
}, useDefaultFilters = false)
public class MainConfig {
    // add this bean to IoC
   @Bean
   public Person person01() {
       return new Person("Name", 14);
   }
}
```

### @Import

```java
@Configuration
@Import(value = {Dog.class, Cat.class, TestImportSelector.class, TestImportBeanDefinitionRegister.class})
public class MainConfig {
    @Bean
    public Person person() {
        System.out.println("add person bean");
        return new Person("Name", 14);
    }
}

```
```java
public class TestImportSelector implements ImportSelector {
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"com.app.spring6.bean.Fish", "com.app.spring6.bean.Tiger"};
    }
}
```

Use BeanDefinitionRegistry to manually register a bean
```java
public class TestImportBeanDefinitionRegister implements ImportBeanDefinitionRegistrar {
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                        BeanDefinitionRegistry registry) {
        boolean dog = registry.containsBeanDefinition("com.app.spring6.bean.Dog");
        boolean cat = registry.containsBeanDefinition("com.app.spring6.bean.cat");
        if (dog && cat) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition("com.app.spring6.bean.Pig");
            registry.registerBeanDefinition("pig", beanDefinition);
        }
    }
}
```
### FactoryBean
A FactoryBean is defined in a bean style, but the object exposed for bean is always the object that it creates.

```java
public class TestFactoryBean implements FactoryBean<Monkey> {
    public Monkey getObject() throws Exception {
        return new Monkey();
    }

    public Class<?> getObjectType() {
        return null;
    }
}
```
In AbstractBeanFactory class:

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd == null) {
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

总结：
给容器注册组件：
- 包扫描 (@Controller, @Service, @Repository, @Component)
  - 一般针对我们自己写的累
- @Bean 导入第三方类或包的组件
- @Import 
- 使用FactoryBean

# Spring Bean lifecycle

- bean的生命周期是指：bean创建 -> 初始化 -> 销毁
  - 在@Bean中指定initMethod and destroyMethod
  - 让bean实现InitializingBean 和DisposableBean 接口
  - 使用@PostConstruct和@PreDestory annotation

- BeanPostProcessor

## Use initMethod and destoryMethod
```java
@Configuration
public class MainConfig {

    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Flower flower() {
        return new Flower();
    }
}
```

```java
public class Flower {

    public Flower() {
        System.out.println("Flower constructor...");
    }

    public void init() {
        System.out.println("Flower ...init...");
    }

    public void destroy() {
        System.out.println("Flower ...destroy...");
    }
}
```
```java
    @Test
    public void test01() {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(MainConfig.class);
        System.out.println("IoC is created");
        ac.close();
    }
```


Output:
```
Flower constructor...
Flower ...init...
IoC is created
Flower ...destroy...
```

In AbstractBeanFactory.java, doGetBean method:

```java
// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
```

createBean in AbstractAutowireCapableBeanFactory

## Use InitializingBean, DisposableBean interface
```java
@Component
public class Train implements InitializingBean, DisposableBean {
    //...
}
```

## Use BeanPostProcessor
```java
@Component
public class TestBeanPostProcessor implements BeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessBeforeInitialization...." + beanName + "bean");
        return bean;
    }


    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("postProcessAfterInitialization...." + beanName + "bean");
        return bean;
    }
}
```
PostProcessorBeforeInit -> initMethod -> PostProcessorAfterInit
initializeBean method:
```java
        Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}
```

Processors are also beans in IoC. They're created before business bean.

Spring底层对BeanPostProcessor的使用:

- ```ApplicationContextAwareProcessor```
- ```BeanValidationPostProcessor```
- ```InitDestroyAnnotationBeanPostProcessor```

# @Value


Use @Value to assign value to this variable. It supports String, StringEL and properties

```java
    @Value("Test")
    private String name;
    @Value("#{20-2}")
    private Integer age;
    @Value("${bird.color}")
    private String color;
```

```java
@Configuration
@PropertySource("classpath:/test.properties")
public class MainConfig {
```

#  Autowired, Qualifier, Primary


```java
@Autowired(required = false)
```


# Aware interface

The following method in `AbstractAutowireCapableBeanFactory`: Initialize the given bean instance, applying factory callbacks as well as init methods and bean post processors.

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

总结：把Spring底层的组件可以注入到自定义的bean中

# AOP
* @Before
* @After
* @AfterReturning
* @AfterThrowing
* @Around

* @EnableAspectJAutoProxy

```java
@Aspect
public class LogAspects {

    @Pointcut("execution(public int com.app.spring10.aop.Calculator.div(int, int))")
    public void pointCut() {

    }

    @Before("pointCut()")
    public void logStart(JoinPoint joinPoint) {
        System.out.println("div start @Before " + joinPoint.getSignature().getName()
        + " " + Arrays.toString(joinPoint.getArgs()));
    }

    @After("pointCut()")
    public void logEnd() {
        System.out.println("div end @After");
    }

    @AfterReturning(value = "pointCut()", returning = "result")
    public void logReturn(int result) {
        System.out.println("div return... result is @AfterReturning " + result);
    }

    @AfterThrowing(value = "pointCut()", throwing = "exception")
    public void logException(Exception exception) {
        System.out.println("div return throwing... exception is @AfterThrowing " + exception);
    }

    @Around("pointCut()")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        System.out.println("Before proceed @Around");
        Object obj = proceedingJoinPoint.proceed();
        System.out.println("After proceed @Around");
        return obj;
    }
}
```

```
Before proceed @Around
div start @Before
After proceed @Around
div end @After
div return... result is @AfterReturning 
```

AOP原理： 给容器注册了什么组件，这个组件什么时候工作，这个组件功能是什么

我们打开源码知道`class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar`，ImportBeanDefinitionRegistrar 是可以给容器自定义注册组件的，
- 它注册`AnnotationAwareAspectJAutoProxyCreator`组件
- 然后`AnnotationAwareAspectJAutoProxyCreator`是实现了一个postprocessor和Ordered和BeanFactoryAware
`class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar`

==> `AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);`

==> `registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source)`

==> `registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);` (AUTO_PROXY_CREATOR_BEAN_NAME = org.springframework.aop.config.internalAutoProxyCreator )

So the `AnnotationAwareAspectJAutoProxyCreator` bean definition is registered in IoC.

`public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator`. We can see it's `SmartInstantiationAwareBeanPostProcessor`, `Ordered` and `BeanFactoryAware`

`AbstractAutoProxyCreator` creates the proxy of the target class.


# Transactional

@EnableTransactionManagement

`public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement>`

it's a selector.

`TransactionInterceptor.class`




# BeanFactory的两个重要的后置处理器

- BeanFactoryPostProcessor 
  - 在BeanFactory标准初始化之后调用，来定制和修改BeanFactory的内容；
  - 所有的bean定义已经保存加载到beanFactory，但是bean的实例还未创建

- BeanDefinitionRegistryPostProcessor

# Spring Web MVC

Spring Web MVC is hte original web framework built on the Servlet API and has been included in the Spring Framework from the very beginning. In addition to Spring Web MVC, Spring Framework 5.0 introduced a reactive-stack web framework whose name is "Spring WebFlux"

## ServletContainerInitializer

web.xml three main components:
* Servlet
The Container runs multiple threads to process multiple requests to a single servlet.


* Filter
* Listener

SpringMVC uses ServletContainerInitializer to load IoC and to register servlet, filter and listener.
```java
 void onStartup(java.util.Set<java.lang.Class<?>> c, ServletContext ctx) {

     // register servlet
     javax.servlet.ServletRegistration.Dynamic servlet = ctx.addServlet("orderServlet", new OrderServlet());
     servlet.addMapping("/orderTest");

    // register listener
    ctx.addListener(OrderListener.class);

    // register filter
    javax.servlet.FilterRegistration.Dynamic filter = ctx.addFilter("orderFilter", OrderFilter.class);
    // add mapping to specify which servlet to intercept
    filter.addMappingForUrlPatterns();
 }
```

## SpringServletContainerInitializer

```java
@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}
```

Let's see what impl of webAppInitializerClasses are:

{% asset_img webappInitializer webappInitializer %}


First one is `AbstractContextLoaerInitializer`. In its onStartup method, it registers context loader listener.
```java
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}

	/**
	 * Register a {@link ContextLoaderListener} against the given servlet context. The
	 * {@code ContextLoaderListener} is initialized with the application context returned
	 * from the {@link #createRootApplicationContext()} template method.
	 * @param servletContext the servlet context to register the listener against
	 */
	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}
```
It creates a root application context firstly. It uses template pattern here. The actual creation is done by its child class.

Second one is `AbstractDispatcherServletInitializer`. As name implies, it initializes dispatcherServlet.

```java
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);
		registerDispatcherServlet(servletContext);
	}

	/**
	 * Register a {@link DispatcherServlet} against the given servlet context.
	 * <p>This method will create a {@code DispatcherServlet} with the name returned by
	 * {@link #getServletName()}, initializing it with the application context returned
	 * from {@link #createServletApplicationContext()}, and mapping it to the patterns
	 * returned from {@link #getServletMappings()}.
	 * <p>Further customization can be achieved by overriding {@link
	 * #customizeRegistration(ServletRegistration.Dynamic)} or
	 * {@link #createDispatcherServlet(WebApplicationContext)}.
	 * @param servletContext the context to register the servlet against
	 */
	protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();
		Assert.hasLength(servletName, "getServletName() must not return null or empty");

		WebApplicationContext servletAppContext = createServletApplicationContext();
		Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
		Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
		dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
		if (registration == null) {
			throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
					"Check if there is another servlet registered under the same name.");
		}

		registration.setLoadOnStartup(1);
		registration.addMapping(getServletMappings());
		registration.setAsyncSupported(isAsyncSupported());

		Filter[] filters = getServletFilters();
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);
			}
		}

		customizeRegistration(registration);
	}
```

It creates servlet application context. Same here, the actual implementation of `createServletApplicationContext` is done by its child class.

Now, The third impl is `AbstractAnnotationConfigDispatcherServletInitializer`;

```java
    @Override
	@Nullable
	protected WebApplicationContext createRootApplicationContext() {
		Class<?>[] configClasses = getRootConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
			context.register(configClasses);
			return context;
		}
		else {
			return null;
		}
	}

	@Override
	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		Class<?>[] configClasses = getServletConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			context.register(configClasses);
		}
		return context;
	}
```

`https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#spring-web`

DispatcherServlet is a HttpServlet. It prvoides a shared algorithm for request processing, while actual work is performed by configurable delegate components. The DispatcherServlet uses Spring configuration to discover the delegate components it needs for request mapping, view resolution, exception handling, and more.


## Asynchronous Requests

In Servlet 3.0, you can obtain AsyncContext object by startAsync method of ServletRequest.  
request receiver thread and request processing thread are separate. It doesn't block the Tomcat HTTP thread.


### DeferredResult

DeferredResult, available from Spring 3.2 onwards, assists in offloading a long-running computation from an http-worker thread to a separate thread. It can retrieve result from different thread. 

```java
@RestController
public class BlockingRESTController {

 @GetMapping("/blocking-request")
 public ResponseEntity < ? > blockHttpRequest() throws InterruptedException {

  Thread.sleep(4000);
  return ResponseEntity.ok("OK");
 }
}
```

=>

```java
@RestController
public class DeferredResultController {

 private static final Logger LOGGER = LoggerFactory.getLogger(DeferredResultController.class);

 @GetMapping("/asynchronous-request")
 public DeferredResult < ResponseEntity < ? >> asynchronousRequestProcessing(final Model model) {

  LOGGER.info("Started processing asynchronous request");
  final DeferredResult < ResponseEntity < ? >> deferredResult = new DeferredResult < > ();

  /**
   * Section to simulate slow running thread blocking process
   */
  ForkJoinPool forkJoinPool = new ForkJoinPool();
  forkJoinPool.submit(() -> {
   LOGGER.info("Processing request in new thread");
   try {
    Thread.sleep(4000);

   } catch (InterruptedException e) {
    LOGGER.error("InterruptedException while executing the thread {}", e.fillInStackTrace());
   }
   deferredResult.setResult(ResponseEntity.ok("OK"));
  });

  return deferredResult;
 }

}
```

output:
```
... INFO 11879 --- [nio-8080-exec-1] c.j.d.DeferredResultController           : Started processing asynchronous request
... INFO 11879 --- [nio-8080-exec-1] c.j.d.DeferredResultController           : HTTP Wroker thread is relased.
...  INFO 11879 --- [Pool-1-worker-1] c.j.d.DeferredResultController           : Processing request in new thread
```

*Callable

```java
@PostMapping
public Callable<String> processUpload(final MultipartFile file) {

    return new Callable<String>() {
        public String call() throws Exception {
            // ...
            return "someView";
        }
    };

}
```

## ContextLoaderListener
实现ServletContextListener接口，向ServletContext中添加任意的对象。

ServletContextListener中的核心逻辑便是初始化WebApplicationContext实例并存放至ServletContext中。

- 初始化
  - Servlet container load servlet class
  - Servlet container creates a ServletConfig对象
  - Servlet container creates a servlet 对象
  - Servlet cotnainer call init method

- 运行阶段

  - servlet container会创建ServletRequest和ServletResponse对象，然后调用service方法。

- 销毁阶段
  - servlet container会invoke servlet对象的destroy方法。然后再销毁servlet对象。destroy方法里可以关闭数据库链接等等


## Handler Methods
`https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-ann-methods`

@RequestMapping handler methods have a flexible signature and can choose from a range of supported controller method arguments and return values.

	
For access to the HTTP request body. Body content is converted to the declared method argument type by using HttpMessageConverter implementations. See @RequestBody.

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) { 
    //...
}
```

```
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```

```java
@GetMapping("/demo")
public void handle(
        @RequestHeader("Accept-Encoding") String encoding, 
        @RequestHeader("Keep-Alive") long keepAlive) { 
    //...
}
```
When an @RequestHeader annotation is used on a Map<String, String>, MultiValueMap<String, String>, or HttpHeaders argument, the map is populated with all header values.

```java
@PostMapping(path = "/pets", consumes = "application/json") 
public void addPet(@RequestBody Pet pet) {
    // ...
}
```

