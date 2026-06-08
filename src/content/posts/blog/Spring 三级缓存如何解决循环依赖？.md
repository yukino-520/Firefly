---
title: Spring 三级缓存如何解决循环依赖？
published: 2026-05-30
updated: 2026-06-08
description: spring三级缓存
tags:
  - 八股
  - Java
category: 后端开发
draft: false
author: yukino
---
在 Spring 中，循环依赖是一个非常经典的问题。

所谓循环依赖，就是两个或多个 Bean 之间相互依赖。最常见的情况是：A 依赖 B，B 又依赖 A。

比如下面这段代码：

```
@Component
public class A {
 
    @Autowired
    private B b;
}
```
```
@Component
public class B {
 
    @Autowired
    private A a;
}
```

从直觉上看，这段代码似乎会出问题。

Spring 创建 A 的时候，发现 A 需要 B，于是去创建 B。创建 B 的时候，又发现 B 需要 A，于是又回来创建 A。这样一来，A 等 B，B 又等 A，看起来就像陷入了一个死循环。

但实际开发中，在单例 Bean 的属性注入场景下，Spring 往往可以正常 解决 这种循环依赖。

原因就在于 Spring 的三级缓存机制。

## 一、什么是三级缓存

Spring 解决单例 Bean 循环依赖的核心逻辑，主要在 DefaultSingletonBeanRegistry 中。

里面有三个非常关键的缓存：

```java
// 一级缓存：保存已经完整创建好的单例 Bean

private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

 

// 二级缓存：保存提前暴露的 Bean 的早期引用

private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

 

// 三级缓存：保存 ObjectFactory，用来生成 Bean 的早期引用

private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
java运行
```

这三个缓存分别承担不同的职责。

一级缓存 singletonObjects，保存的是已经完整创建好的单例 Bean。

所谓完整创建好，意思是这个 Bean 已经经历了实例化、属性填充、初始化等完整 生命周期 。平时我们从 Spring 容器中获取 Bean，大多数情况下拿到的就是一级缓存里的对象。

二级缓存 earlySingletonObjects，保存的是提前暴露的 Bean 的早期引用。

这个对象已经被实例化出来了，但是还没有完成属性填充和初始化。因此可以把它理解成一个“半成品 Bean”。

三级缓存 singletonFactories，保存的是 ObjectFactory。

它不是直接保存 Bean 对象，而是保存一个工厂。只有当发生循环依赖，确实需要提前暴露 Bean 的时候，Spring 才会调用这个 ObjectFactory，生成对应的早期引用。

## 二、A 和 B 的循环依赖创建过程

接下来用 A 和 B 的例子，把整个流程串起来。

假设现在有两个 Bean：

```
@Component
public class A {
 
    @Autowired
    private B b;
}
```
```
@Component
public class B {
 
    @Autowired
    private A a;
}
```

Spring 首先开始创建 A。

创建 Bean 的第一步是实例化。也就是说，Spring 会先通过构造方法把 A 对象 new 出来。

这个时候，A 对象已经存在了，但是它里面的属性 b 还没有被赋值，初始化方法也还没有执行。

由于 A 是单例 Bean，并且 Spring 允许提前暴露单例对象，所以 Spring 会把 A 对应的 ObjectFactory 放入三级缓存 singletonFactories 中。

注意，此时放进去的不是 A 本身，而是一个可以生成 A 的早期引用的工厂。

接着，Spring 开始给 A 填充属性。

填充属性时，Spring 发现 A 依赖 B，于是开始创建 B。

创建 B 的过程和 A 类 似。Spring 先实例化 B，然后也会把 B 对应的 ObjectFactory 放入三级缓存。

之后 Spring 开始给 B 填充属性，发现 B 依赖 A。

这个时候，关键点来了。

Spring 会尝试从容器中获取 A。

首先查一级缓存 singletonObjects。

结果发现没有。因为 A 还没有创建完成，自然不会出现在一级缓存中。

然后查二级缓存 earlySingletonObjects。

结果还是没有。因为此时 A 的早期引用还没有真正生成出来。

最后查三级缓存 singletonFactories。

这一次找到了 A 对应的 ObjectFactory。于是 Spring 调用这个 ObjectFactory，拿到 A 的早期引用。

拿到之后，Spring 会把 A 的早期引用放入二级缓存 earlySingletonObjects，同时从三级缓存 singletonFactories 中移除 A 对应的 ObjectFactory。

这样，B 就可以拿到 A 的引用，完成自己的属性填充和初始化。

B 创建完成之后，会被放入一级缓存 singletonObjects。

接着，Spring 回到 A 的创建流程。

此时 A 需要的 B 已经创建完成，并且已经在一级缓存中了，所以 A 可以顺利完成属性填充、初始化，最后也进入一级缓存。

到这里，A 和 B 的循环依赖就被解决了。

整个过程可以简单理解成：

Spring 先把 A 实例化出来，并提前暴露一个可以获取 A 早期引用的工厂。等 B 创建过程中需要 A 时，就通过这个工厂拿到 A 的早期引用，让 B 先完成创建。B 创建完成后，A 再继续完成自己的创建流程。

## 三、为什么需要三级缓存

很多人看到这里会有一个疑问：

既然二级缓存已经可以保存早期引用了，为什么还需要三级缓存？为什么不在 A 实例化之后，直接把 A 放进二级缓存？

这个问题非常关键。

如果只是为了解决普通的属性注入循环依赖，二级缓存似乎已经够用了。

A 实例化之后，直接把 A 的早期引用放入二级缓存。B 需要 A 的时候，直接从二级缓存拿出来。流程看起来也能跑通。

但是 Spring 需要考虑一个更复杂的问题：AOP。

在 Spring 中，一个 Bean 最终暴露给外部使用的对象，可能不是原始对象，而是代理对象。

比如某个 Bean 上有事务注解：

```java
@Service

public class OrderService {

 

    @Transactional

    public void createOrder() {

        // 创建订单

    }

}
java运行
```

为了让事务生效，Spring 可能会为 OrderService 创建一个代理对象。外部调用的并不是原始的 OrderService 对象，而是它的代理对象。

如果 Spring 在实例化之后就直接把原始对象放入二级缓存，那么发生循环依赖时，其他 Bean 注入到的可能就是原始对象。

但是等这个 Bean 完成初始化之后，Spring 最终放入一级缓存的又可能是代理对象。

这样就会出现一个严重问题：

有些地方拿到的是原始对象，有些地方拿到的是代理对象。

同一个 Bean 在容器中出现了两个不同的引用，这会导致语义混乱。更严重的是，如果某个地方拿到的是原始对象，那么事务、切面等 AOP 功能可能就不会生效。

三级缓存的价值就在这里。

三级缓存里保存的是 ObjectFactory，而不是 Bean 本身。这个 ObjectFactory 的作用，是在真正需要提前暴露 Bean 的时候，再去生成早期引用。

在生成早期引用的过程中，Spring 会调用类似 getEarlyBeanReference 的逻辑。

这个过程会给 BeanPostProcessor 一个机会，让它判断当前 Bean 是否需要被代理。如果需要代理，那么返回的早期引用就可以是代理对象，而不是原始对象。

也就是说，三级缓存的核心意义是：

延迟生成早期引用，并且允许 AOP 在早期引用生成阶段介入。

这就是为什么 Spring 不直接把实例化后的 Bean 放入二级缓存，而是先放一个 ObjectFactory 到三级缓存。

## 四、三级缓存之间的数据流转

理解三级缓存，还要理解 Bean 在这三个缓存中的流转过程。

一个正常创建完成的单例 Bean，最终会进入一级缓存 singletonObjects。

如果创建过程中没有发生循环依赖，那么三级缓存中的 ObjectFactory 可能根本不会被调用。

如果发生循环依赖，Spring 才会从三级缓存中取出 ObjectFactory，调用它生成早期引用。

生成早期引用之后，这个对象会被放入二级缓存 earlySingletonObjects，同时对应的 ObjectFactory 会从三级缓存 singletonFactories 中移除。

等 Bean 最终创建完成之后，它会被放入一级缓存 singletonObjects，同时从二级缓存和三级缓存中清理掉相关记录。

所以三级缓存不是三个地方都长期保存 Bean，而是在 Bean 创建的不同阶段承担不同职责。

一级缓存保存成品。

二级缓存保存已经生成的早期引用。

三级缓存保存生成早期引用的工厂。

## 五、Spring 三级缓存能解决哪些循环依赖

Spring 三级缓存并不是万能的。

它主要解决的是单例 Bean 的属性注入循环依赖。

比如这种：

```
java运行
```
```
java运行
```

或者 setter 注入：

```java
@Component

public class A {

 

    private B b;

 

    @Autowired

    public void setB(B b) {

        this.b = b;

    }

}
java运行
```

类循环依赖有一个共同特点：

对象可以先被实例化出来，然后再填充属性。

只要对象能先实例化，Spring 就有机会提前暴露它的早期引用，从而打破循环依赖。

## 六、Spring 解决不了哪些循环依赖

第一种是构造器注入的循环依赖。

比如：

```java
@Component

public class A {

 

    private final B b;

 

    public A(B b) {

        this.b = b;

    }

}
java运行
```
```javascript
@Component

public class B {

 

    private final A a;

 

    public B(A a) {

        this.a = a;

    }

}
```

这种情况 Spring 无法通过三级缓存解决。

原因很简单：构造器注入要求对象在创建时就必须拿到依赖。

创建 A 必须先有 B，创建 B 又必须先有 A。此时 A 和 B 都没有完成实例化，Spring 没有办法提前暴露任何一个对象的引用。

三级缓存解决循环依赖的前提，是至少可以先把对象实例化出来。

构造器循环依赖的问题就在于，对象连实例化这一步都过不去。

第二种是 prototype Bean 的循环依赖。

Spring 的三级缓存机制主要针对单例 Bean。

prototype Bean 每次获取都会创建一个新的对象，不会像单例 Bean 那样放入 singletonObjects 这类缓存中。

所以 prototype Bean 的循环依赖，不能依赖三级缓存解决。

第三种情况是 Spring Boot 2.6 之后默认关闭循环依赖。

从 Spring Boot 2.6 开始，默认配置是：

> spring.main.allow-circular-references=false

也就是说，即使是单例 Bean 的属性注入循环依赖，也可能直接启动失败。

如果确实需要开启，可以配置：

> spring.main.allow-circular-references=true

不过从设计角度来说，更推荐消除循环依赖，而不是依赖这个配置。

## 七、为什么不建议依赖循环依赖

虽然 Spring 可以在某些情况下解决循环依赖，但这并不意味着循环依赖是一种好的设计。

循环依赖通常说明两个类之间的职责边界不够清晰。

A 需要 B，B 又需要 A，意味着它们之间互相知道太多，耦合程度过高。

短期看，Spring 帮我们把对象创建出来了，程序能跑。

但长期看，这种设计会让代码越来越难维护。修改 A 可能影响 B，修改 B 又可能影响 A。测试也会变得麻烦，因为很难单独隔离其中一个类。

更好的方式通常是重新梳理职责。

比如，可以把 A 和 B 共同依赖的逻辑抽到第三个 Bean 中；也可以通过 事件 机制解耦；或者重新划分服务边界，让依赖关系变成单向的。

Spring 的三级缓存是一种框架层面的兜底机制，不应该成为业务设计上的依赖。

## 八、总结

Spring 三级缓存解决循环依赖的本质，是提前暴露单例 Bean 的早期引用。

一级缓存 singletonObjects 保存完整创建好的单例 Bean。

二级缓存 earlySingletonObjects 保存提前暴露的 Bean 的早期引用。

三级缓存 singletonFactories 保存 ObjectFactory，用来延迟生成 Bean 的早期引用。

当 A 和 B 发生属性注入循环依赖时，Spring 会先实例化 A，并把 A 的 ObjectFactory 放入三级缓存。创建 B 的过程中，如果 B 需要 A，Spring 就会通过三级缓存拿到 A 的早期引用，让 B 先完成创建。B 创建完成后，A 再继续完成自己的属性填充和初始化。

三级缓存最关键的价值，不只是解决循环依赖，而是为了兼容 AOP。

因为 Spring 最终暴露出去的 Bean 可能是代理对象。如果过早把原始对象暴露出去，就可能导致有些地方拿到原始对象，有些地方拿到代理对象。三级缓存通过 ObjectFactory 延迟生成早期引用，让 AOP 有机会提前介入，从而尽量保证引用一致。

不过，Spring 三级缓存并不能解决所有循环依赖。

它主要解决单例 Bean 的属性注入循环依赖，无法解决构造器循环依赖，也无法解决 prototype Bean 的循环依赖。

最后还是要记住一句话：

Spring 可以帮我们解决一部分循环依赖，但更好的做法，是从设计上避免循环依赖。