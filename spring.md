#	AOP 非入侵性
#	IOC	依赖倒转 里氏代换原则
#	面向接口编程
#	Adapter 适配器
#	filter 责任链
#  装饰器 Decorator
#	代理   Proxy cglib 动态代理
##	代理模式可以在不修改被代理对象的基础上,通过扩展代理类,进行一些功能的附加与增强。值得注意的是,
##	代理类和被代理类应该共同实现一个接口,或者是共同继承某个类。
	spring真正的执行方法并不是直接执行方法,而是执行代理方法。面向切面编程的原理也是如此。

#	spring BOOT 启动流程
##	new SpringApplication(primarySources)
	1、获取application类型
	2、getSpringFactoriesInstances 初始化容器资源
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		Enumeration<URL> urls = (classLoader != null ?classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
		createSpringFactoriesInstances
	3、赋值 setInitializers setListeners 
		mainApplicationClass = ***.InstantMessageApplication
		primarySource = ***.InstantMessageApplication
```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
	Assert.notNull(primarySources, "PrimarySources must not be null");
	this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
	//webApplicationType = SERVLET
	this.webApplicationType = deduceWebApplicationType();
	setInitializers((Collection) getSpringFactoriesInstances(
			ApplicationContextInitializer.class));
	setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
	this.mainApplicationClass = deduceMainApplicationClass();
}
```


##	springApplication.run()
	StopWatch 计时器 方便的对程序部分代码进行计时(ms级别)，适用于同步单线程代码块。

#	spring MVC 处理请求
##  RequestMappingHandlerMapping->afterPropertiesSet->initHandlerMethods->mappingRegistry.register

#	spring BEAN生命周期 
	* Instantiation 实例化
	* Populate 属性赋值
	* Initialization 初始化
	* Destruction 销毁

##	作用域
###  singleton
    是指在Spring IoC容器中仅存在一个Bean的示例,Bean以单实例的方式存在
###  prototype
    是指每次从容器中调用Bean时,都返回一个新的实例,即每次调用getBean()时,相当于执行new Bean()的操作。在默认情况下,Spring容器在启动时不实例化prototype的Bean。
###  request
###  session
###  global session
###  application

##	普通Java类获取spring 容器的bean的5种方法 全部直接或者间接继承自 BeanFactory
	ApplicationContext 继承自 BeanFactory
	applicationContext.getBean(beanName);
	beanFactory.getBean("taskTestService");
	ApplicationContext、BeanFactory 对象         				
	ApplicationObjectSupport 		
	WebApplicationObjectSupport		
	实现接口ApplicationContextAware				
									
##	Aware接口为了能够感知到自身的一些属性

#	EJB JavaBean POJO VO
##	EJB	企业级Java bean
###	VO POJO 简单Java Bean

#	注意、spring AOP、cache、asyn全部是依赖代理实现的。可以理解为spring所有增强、解耦和防止代码侵入性全部都是依赖代理实现的
#	Cache
##	@EnableCache
##	@CacheAble

#	Async
##	EnableAsync
##	@Async
