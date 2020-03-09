---
title: java 泛型：PECS原则
author: 小飞
date: 2020-03-09 21:59:11
tags: [java, 泛型]
---
## PECS原则：
只读类型相当于生产者（product），只写类型相当于消费者（consumer），生产T，使用？ extends T， 消费T，则使用 ? super T.
<!-- more -->
例如：Collections.copy就使用了这个原则：
![](/images/pecs.png)    
Dest 只写，用 ? Super T， src 只读，用 ? Extends T。

## 泛型通配符使用建议:

* 只读类型使用上界通配符? extends T
* 只写类型使用下界通配符? super T
* 如果只读类型只用到 Object 的方法，即List<? extends Object>，可以用List<?>无界通配符
* 对于同时需要读取和写入的类型，不要使用通配符

上面原则不适用于方法返回值类型。避免在返回值中使用通配符，因为这样会强制要求调用者调用时处理通配符

## 类型擦除
List<Integer> 和 List<String> 的class 的类型时一致的。但是在编译时的类型是不一样的。java只提供了编译时的类型检查，并没有提供运行时的。     
Java 编译器在应用泛型类型擦除时有以下行为：
* 将泛型中所有参数化类型替换为泛型边界，如果参数化类型是无界的，则替换为 Object 类型。字节码中没有任何泛型的相关信息。
* 为了类型安全，在必要时插入类型转换代码。
* 生成桥接方法来保持泛型类型的多态性。

## 泛型限制

1. 不能使用基本类型实例化泛型；
2. 不能创建参数化类型的实例，但是可以使用反射创建
3. ![](/images/limit1.png)
4. 不能将静态属性声明为泛型类型
5. ![](/images/limit2.png)