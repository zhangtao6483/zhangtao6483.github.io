---
layout:     post
title:      "Spring笔记（事件驱动模型）"
date:       2016-10-27
author:     "zhangtao"
header-img: "img/home-bg.jpg"
tags:
    - Spring
---

事件驱动模型，也可以叫做观察者模式，或者发布/订阅模式。在Spring中，
ApplicationContext继承了ApplicationEventPublisher实现事件机制

# 1. 观察者模式

>**观察者模式**定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新

发布消息的称为**主题**（Subject），订阅消息的被称为**观察者**（Observer）

![observer](/images/observer.png)

观察者模式的实现：

- 建立接口
```java
public interface Subject {
	public void registerObserver(Observer o);
	public void removeObserver(Observer o);
	public void notifyObserver();
}

public interface Observer {
	public void update(float temp);
}
```

- 实现主题接口
```java
public class ConcreteSubject implements Subject {

	private ArrayList<Observer> observers;
	private float temp;

	public ConcreteSubject() {
		observers = new ArrayList<Observer>();
	}

	public void registerObserver(Observer o) {
		observers.add(o);
	}

	public void removeObserver(Observer o) {
		int i = observers.indexOf(o);
		if (i >= 0) {
			observers.remove(i);
		}
	}

	public void notifyObserver() {
		for (int i = 0; i < observers.size(); i++) {
			Observer observer = observers.get(i);
			observer.update(temp);
		}
	}

	public void subjectChanged() {
		notifyObserver();
	}

	public void setTemp(float temp) {
		this.temp = temp;
		subjectChanged();
	}
}
```

- 观察者接口
```java
public class ConcreteObserver implements Observer {

	private float temp;
	private Subject subject;

	public ConcreteObserver(Subject subject) {
		this.subject = subject;
		subject.registerObserver(this);
	}

	public void update(float temp) {
		this.temp = temp;
		display();
	}

	public void display() {
		System.out.println("temp : " + temp);
	}
}
```

- 写一个测试程序
```java
public static void main(String[] args) {
	ConcreteSubject subject = new ConcreteSubject();
	ConcreteObserver observer = new ConcreteObserver(subject);
	subject.setTemp(1F);
	subject.setTemp(2F);
}
```
Java API中由内置的观察者模式。java.util包内包含最基本的Observer接口与Observable类，和Subject接口和Observer接口很相似，甚至可以使用推(push)或拉(pull)方式传送数据

# 2. Spring提供的事件驱动模型

![event](/images/spring_event.png)

事件驱动的模型如上图，其中：
- ApplicationEvent
	事件，继承java.util.EventObject，在EventObject中使用source得到事件源，ApplicationEvent抽象类，只有timestamp保存事件发生时间。
- ApplicationListener
	事件监听者，继承java.util.EventListener，只有一个onApplicationEvent方法。
- ApplicationEventMulticaster 
	事件广播器，将publish的事件广播给所有监听者


简单实现一个Spring事件

1. 实现自定义容器事件
```java
public class MessageEvent extends ApplicationEvent {

	private String message;

	public MessageEvent(Object source, String message) {
		super(source);
		this.message = message;
	}

	public MessageEvent(Object source) {
		super(source);
	}

	public String getMessage() {
		return message;
	}

	public void setText(String message) {
		this.message = message;
	}
}
```

2. 实现事件监听器
```java
public class MessageListener implements ApplicationListener {

	public void onApplicationEvent(ApplicationEvent event) {
		if (event instanceof MessageEvent) {
			MessageEvent messageEvent = (MessageEvent) event;
			System.out.println(messageEvent.getMessage());
		} else {
			System.out.println(event);
		}
	}
}
```

3. 将监听器写入配置文件
```xml
<beans>
	<bean class="com.event.MessageListener">
	</bean>
</beans>
```

4. 当系统创建Spring容器、加载Spring容器时会自动触发容器事件，容器事件监听器可以监听到这些事件。除此之外，程序也可以调用ApplicationContext的publishEvent()方法来主动触发一个容器事件，实现一个简单的例子：
```java
public static void main(String[] args) {
	ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
	MessageEvent event = new MessageEvent("hello");
	context.publishEvent(event);
}
```
如果Bean想发布事件，则Bean必须获得其容器的引用。如果程序中没有直接获取容器的引用，则应该让Bean实现ApplicationContextAware或者BeanFactoryAware接口，从而可以获得容器的引用。

# 3. 实现方式

## 3.1 广播器、监听者初始化

在AbstractApplication的refresh方法中定义了广播器初始化方法initApplicationEventMulticaster和监听者注册方法registerListeners

**事件广播器的初始化：**

```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isDebugEnabled()) {
			logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isDebugEnabled()) {
			logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
					APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
					"': using default [" + this.applicationEventMulticaster + "]");
		}
	}
}
```
用户可以在配置文件中为容器定义一个继承ApplicationEventMulticaster的自定义的事件广播器，默认使用SimpleApplicationEventMulticaster作为事件广播器

**注册事件监听器**

```java
protected void registerListeners() {
	// Register statically specified listeners first.
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// Publish early application events now that we finally have a multicaster...
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```

通过getBeanNamesForType方法得到继承ApplicationListener的Bean，将它们注册为容器的事件监听器，添加到事件广播器所提供的监听器注册表中。

**发布事件**
```java
// Multicast right now if possible - or lazily once the multicaster is initialized
if (this.earlyApplicationEvents != null) {
	this.earlyApplicationEvents.add(applicationEvent);
}
else {
	getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
}

// Publish event via parent context as well...
if (this.parent != null) {
	if (this.parent instanceof AbstractApplicationContext) {
		((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
	}
	else {
		this.parent.publishEvent(event);
	}
}
```

## 3.2 SimpleApplicationEventMulticaster

```java
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		Executor executor = getTaskExecutor();
		if (executor != null) {
			executor.execute(new Runnable() {
				@Override
				public void run() {
					invokeListener(listener, event);
				}
			});
		}
		else {
			invokeListener(listener, event);
		}
	}
}
```
事件发布方法，遍历注册的每个监听器，并启动来调用每个监听器的onApplicationEvent方法。
由于SimpleApplicationEventMulticaster的taskExecutor的实现类是SyncTaskExecutor，因此，事件监听器对事件的处理，是同步进行的。
从代码可以看出，applicationContext.publishEvent()方法，需要同步等待各个监听器处理完之后，才返回。
也就是说，Spring提供的事件机制，默认是同步的。如果想用异步的，可以自己实现ApplicationEventMulticaster。
