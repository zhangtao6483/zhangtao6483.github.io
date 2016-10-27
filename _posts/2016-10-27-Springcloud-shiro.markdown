---
layout:     post
title:      "Spring Cloud与Shiro整合的一些坑"
subtitle:   " \"Can not understand\""
date:       2016-10-27
author:     "zhangtao"
header-img: "img/home-bg.jpg"
tags:
    - Spring
---

最近公司项目重构，需要使用Spring Cloud的技术栈，主要是看上了Hystrix断路器。但是呢，越是看似简单地东西，使用整合起来必定会遇到些坑。
今天来讲讲Spring Cloud与Shiro整合遇到的一些坑

# 1. ShiroProperties注入问题

为了更符合Spring Cloud的设计方式（Spring Cloud Config可以统一管理配置），以ShiroProperties类的方式加载Shiro配置ss

```java
@ConfigurationProperties(prefix = "shiro")
public class ShiroProperties {
	/**
	* Custom Realm
	*/
	private Class<? extends Realm> realmClass = null;

	/**
	* URL of login
	*/
	private String loginUrl = "/login";
	/**
	 * URL of success
	 */
	private String successUrl = "/index";
	/**
	 * URL of unauthorized
	 */
	private String unauthorizedUrl = "/unauthorized";
	/**
	 * filter chain
	 */
	private Map<String, String> filterChainDefinitions;

	/**
	 * Shiro managed filters
	 */
	private Map<String, Class<? extends Filter>> filters;

	/**
	*  get set ...
	*/
}
```

因为需要配置

```java
/**
 * 保证实现了Shiro内部lifecycle函数的bean执行
 */
@Bean(name = "lifecycleBeanPostProcessor")
@ConditionalOnMissingBean
public LifecycleBeanPostProcessor lifecycleBeanPostProcessor() {
	return new LifecycleBeanPostProcessor();
}
```

LifecycleBeanPostProcessor保证Shiro内部有自己的Bean执行顺序，实现了Initializable接口的Shiro bean初始化时调用Initializable接口回调，在实现了Destroyable接口的Shiro bean销毁时调用 Destroyable接口回调。
使用ShiroProperties这种将配置交给Spring管理的方式，在Autowired注入的方法会在postProcessBeforeInitialization之后实例化注入properties属性，所以会在方法中get得到的properties的属性会为null。
**解决办法是：**
将LifecycleBeanPostProcessor的加载过程放在ShiroConfiguration，而将ShiroProperties的配置注入ShiroAutoConfiguration，ShiroAutoConfiguration也需要Import(ShiroConfiguration)

```java
@Configuration
@ConditionalOnWebApplication
@Import(ShiroConfiguration.class)
@EnableConfigurationProperties(ShiroProperties.class)
public class ShiroAutoConfiguration {}
```

# 2. Realm 注入其他Service

在Realm需要注入Feign提供的service，然而使用@Autowired这种方式注入会报错：

```java
Error creating bean with name 'defaultServletHandlerMapping' defined in class path resource [org/springframework/boot/autoconfigure/web/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.springframework.web.servlet.HandlerMapping]: Factory method 'defaultServletHandlerMapping' threw exception; nested exception is java.lang.IllegalArgumentException: A ServletContext is required to configure default servlet handling
```

可以看到在初始化Spring Boot的default servlet之前发生的问题。
**解决的方式比较清奇：**
首先继承ApplicationContextInitializer，将Spring的ApplicationContext搞出来：

```java 
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextInitializer;
import org.springframework.context.ConfigurableApplicationContext;

public class ServletInitializer implements ApplicationContextInitializer {

	private static ApplicationContext applicationContext;

	@Override
	public void initialize(ConfigurableApplicationContext applicationContext) {
		this.applicationContext = applicationContext;
	}

	public static <T> T getBean(Class<T> clazz) {
		return applicationContext.getBean(clazz);
	}
}
```

然后再Main的Application中添加Initializer：

```java
public static void main(String[] args) {
	SpringApplication appliaction = new SpringApplication(Application.class);
	ServletInitializer servletInitializer = new ServletInitializer();
	appliaction.addInitializers(servletInitializer);
	appliaction.run();
}
```
最后再需要注入Service的地方这样得到：

```java
CustomerRemoteService customerRemoteService = (CustomerRemoteService) ServletInitializer
				.getBean(CustomerRemoteService.class);
```

这里还不能使用类名小写得到，Feign会将远程Service以全类名的方式注入。
看来需要抽空看下Spring Cloud/Spring Boot初始化源码了。

# 3. AbstractSessionDAO

以前的项目实现将Shiro共享缓存放入Redis中的时候RedisSessionDAO会继承AbstractSessionDAO，这样做在一次请求过来的时候回多次调用doCreate和doUpdate方法。
**解决方法：** 继承CachingSessionDAO
