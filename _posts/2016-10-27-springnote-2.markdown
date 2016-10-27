---
layout:     post
title:      "Spring笔记（AOP）"
date:       2016-10-27
author:     "zhangtao"
header-img: "img/home-bg.jpg"
catalog: true
tags:
    - Spring
---


# 1. 代理模式

>**代理模式：**为另一个对象提供一个替身或占位符以控制对这个对象的访问

使用代理模式创建代表对象（representative），让代表对象控制某个对象的访问，被代理的对象可以是远程的对象、创建开销的大对象或需要安全控制的对象。

![](/img/in-post/proxy.png)

Subject为RealSubject和Proxy提供了接口，通过实现同一接口，Proxy在RealSubject出现的地方取代它。
RealSubject是真正做事的对象，它是被Proxy代理和控制的对象

## 1.1 动态代理

java.lang.reflect包支持动态代理，动态代理是指运行时才将代理的类创建出来

![](/img/in-post/invoke.png)

Proxy是由Java产生的，而且实现了完整的Subject接口
实现InvocationHandler，Proxy上任何方法调用都会被传入此类。InvocationHandler控制对RealSubject方法的访问，在实现invoke()方法的时候加上我们的代理逻辑。

简单实现：

```java
public interface Subject {
	abstract public void request();
}
```

```java
public class RealSubject implements Subject {
	public RealSubject() {
	}

	public void request() {
		System.out.println("真实的请求.");
	}
}
```

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class DynamicSubject implements InvocationHandler {

	private Object sub;

	public DynamicSubject() {
	}

	public DynamicSubject(Object obj) {
		sub = obj;
	}

	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		method.invoke(sub, args);
		return null;
	}
}
```

```java
public static void main(String[] args) {
		RealSubject rs = new RealSubject();

		InvocationHandler handler = new DynamicSubject(rs);

		Class<?> clazz = rs.getClass();

		Subject subject = (Subject) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), handler);
		subject.request();
	}
```

使用jdk自身的反射实现动态代理存在限制：
1. 被代理的类要求至少实现了一个Interface
2. 被代理的类要求有public的构造函数(即没有显示的将其设置为private等)
3. 被代理的类要求不是final

## 1. 2 动态代理实现方式

- JDK动态代理
允许开发者在运行期创建接口的代理实例
JDK的动态代理主要涉及到java.lang.reflect包中的两个类：Proxy和InvocationHandler。其中InvocationHandler是一个接口，可以通过实现该接口定义横切逻辑，并通过反射机制调用目标类的代码，动态将横切逻辑和业务逻辑编织在一起。Proxy利用InvocationHandler动态创建一个符合某一接口的实例，生成目标类的代理对象

- AspectJ
定义了AOP语法，能够在编译器提供横切代码的织入，有一个专门的编译器用来生成遵守Java字节编码规范的Class文件

- Spring AOP
纯Java实现，它不需要专门的编译过程，不需要特殊的类装载器，它在运行期通过代理方式向目标类织入增强代码

- CGLib动态代理
使用JDK创建代理有一个限制，即它只能为接口创建代理实例，Proxy.newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h) ，第二个参数需要传入interfaces是需要代理实例实现的接口列表
CGLib采用非常底层的字节码技术，可以为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，并顺势织入横切逻辑

# 2. Spring AOP
# 3. Spring事务
