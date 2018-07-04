---
layout:     post
title:      "Java中的函数式 - 3 Lambda表达式"
date:       2018-07-01
author:     "zhangtao"
header-img: "img/post-bg-2018-1.jpg"
catalog: true
tags:
    - 函数式
---

# 1. Lambda表达式

[wikipedia](https://zh.wikipedia.org/wiki/%CE%9B%E6%BC%94%E7%AE%97)

Lambda演算包括了一条变换规则（变量替换）和一条将函数抽象化定义的方式。<br>
Lambda演算包括了建构lambda项，和对lambda项运行归约的操作。最基本的Lambda演算规则：

| 语法 | 名称 | 描述 |
|-----|-----|-----|
| x | 变量 | 表示参数或数学/逻辑值的字符或字符串 |
| (λx.M) | 抽象化	| 函数定义（M是一个lambda项）。变量x在表达式中已被绑定。|
| (M N) | 应用 | 将函数应用于参数。 M 和 N 是lambda项。|


## 1.1 非形式化的直觉描述

eg: 函数f(x)= x + 2可以用lambda演算表示为λx.x + 2<br>
而f(3)的值可以写作(λx.x + 2) 3

### 演化过程

两数的平方和函数
![](/img/in-post/function_3_2.svg)

可以用匿名的形式重新写为：
![](/img/in-post/function_3_3.svg)

进一步，可以使用柯里化的方法，将多参数的函数转换成为多个中介函数的复合链，每个中介函数都只接受一个参数。
![](/img/in-post/function_3_4.svg)


## 1.2 归约

### α-替换

α 替换(α-conversion)所指的是在不出现命名冲突的前提下可以将函数的参数重新命名。比如，下面等号两边的定义是等价的:

<img src="/img/in-post/function_3_5.png" width="55%" height="55%">

### β-归约

β 归约(β-reduction)则指的是参数到函数体的替换。应用参数 N 于函
数 λx → M，相当于在不出现命名冲突的前提下，把 M 中出现的 x 替换为 N。

<img src="/img/in-post/function_3_6.png" width="57%" height="57%">

eg: (λx → x + 2) y → β y + 2

### η-归约

η 归约(η-reduction)可以用来消去那些冗余的 λ 表达。在定义函数时，
可以将一个参数传给函数 M，进行计算的函数和 M 本身是同一个函数:

<img src="/img/in-post/function_3_7.png" width="40%" height="40%">

eg: (λy → λx → x + y) → (+) x y


# 2. in Java

Lambda表达式的主要应用有两个：

1. 构造一个匿名函数（anonymous function）
2. 对于对于柯里化函数，在不给定前一个参数的前提下给定后一个

## 2.1 基本语法

Lambda表达式有三个部分：

![](/img/in-post/function_3_1.png)

- 参数列表：这里使用了Comparator中compare方法的参数
- 箭头：箭头->把参数列表与Lambda主体分隔开
- Lambda主体：比较两个Apple的重量

Lambda的基本语法是

``` java
	(parameters) -> expression
 或(请注意语句的花括号)
	(parameters) -> { statements; }
```
 
之前

``` java
Comparator<Apple> byWeight = new Comparator<Apple>() {
    public int compare(Apple a1, Apple a2){
return a1.getWeight().compareTo(a2.getWeight());
};
```

使用了Lambda表达式后：

``` java
Comparator<Apple> byWeight =
	(Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight());
```

## 2.2 使用示例

| 使用案例            | Lambda示例 | 对应函数式接口 |
|--------------------|-----------|--------------|
| 布尔表达式          |(List<String> list) -> list.isEmpty()| Predicate<List<String>> |
| 创建对象            |() -> new Apple(10)| Supplier<Apple> |
| 消费一个对象         |(Apple a) ->  {<br>System.out.println(a.getWeight());<br>} | Consumer<Apple> |
| 从一个对象中选择/抽取  |(String s) -> s.length()| Function<String, Integer>或<br>ToIntFunction<String> |
| 组合两个值           |(int a, int b) -> a * b| IntBinaryOperator |
| 比较两个对象          |(Apple a1, Apple a2) -><br>a1.getWeight().compareTo(a2.getWeight())| Comparator<Apple>或 <br>BiFunction<Apple, Apple, Integer> 或 <br>ToIntBiFunction<Apple, Apple> |



