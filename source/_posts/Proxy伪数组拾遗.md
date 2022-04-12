---
title: Proxy伪数组拾遗
tags:
  - code
  - ECMAScript
abbrlink: 15908
date: 2021-07-27 10:21:25
---
在上一篇文章中，我们介绍了使用Proxy来实现一个伪数组，这篇文章是对于上一篇文章的一些补充。

## 1. Array.prototype.map的问题

之前提到，如果让伪数组的prototype指向数组的prototype，就可以直接调用数组的各种方法了，实际上真的如此吗？简单通过以下代码判断：
```typescript
Array.prototype.map.call(bitArray, (bit) => {
    console.log('run here!');
    return bit ? '真' : '假';
});
```
按照我们的想法，这行代码应该打印出32个"run here"，并得到一个由'真'和'假'组成的数组。但是实际上，得到的结果是：*(32) [空 x32]*(在edge下运行得到），而且一个run here都不会被打印。

这是为什么呢？经过一番查找，笔者看到了[这个回答](https://www.zhihu.com/question/60919509/answer/181753797)，原来是因为map在执行的时候，并不是直接执行callback的代码，而是会首先判断该索引在目标上是否存在，如果存在才会执行callback。

不过回答里提到的这个hasProperty也不知道是对应哪个trap的，所以就用上一篇文章写的proxyLogger，以bitArray为target制造一个新的proxy对象，对其调用方法就知道是对应什么trap了：
![](pBitArray.png)

可以看到其实是对应has这个trap的，所以我们加上对应的has trap就可以成功调用map了。

## 2. instanceof 能否判断出Proxy？
这是上次分享会分享了上一篇文章后被提问问到的问题，笔者当时按照Proxy设计的原则认为是无法判断的，instanceof应该也要通过调用某些对象上的基本操作才能得到结果。

分享会后笔者就去用proxyLogger测试了一下：
![](instanceof.png)

果然instanceof是通过prototype来判断是否为目标的实例的，那么我们只需要对这个trap也特殊处理以下就可以了。

## 3. Object.prototype.toString
还可以使用如下代码对bitArray来进行测试：
```typescript
Object.prototype.toString.call(bitArray)
```
对于数组，结果应该是：*[Object Array]*，但是对于我们这里的bitArray，如果完全按照上文的写法，将会抛出一个错误：尝试将symbol类型转换为数字。这里有两个问题：一是说明这个方法读取了一个symbol类型的属性，二是说明我们的get拦截器写得有点问题。

对于第二个问题，只需在转换为数字之前使用typeof判断一下，如果为symbol特殊处理一下就好了。

对于第一个问题，通过[查询MDN](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/toStringTag)可以知道，这个属性是Symbol.toStringTag，Object.toString就是通过这个来得到结果的。所以我们也对这个属性特殊处理一下，让它返回```'Array'```，这样一来结果就合数组一样了。

## 4. Array.isArray
做完以上三个操作，我们的Proxy看起来已经非常像一个数组了，然而，当笔者拿出究极判断方法```Array.isArray```后，还是得到了false。

![](arrayisarray1.png)

甚至笔者拿出带log的proxy后，发现这个方法甚至没有使用任何基本操作：

![](arrayisarray2.png)

在尝试通过搜索引擎搜索无果后，笔者最终只好打开了标准文档，还好，关于Array.isArray的内容不算复杂：

> The abstract operation IsArray takes one argument argument, and performs the following steps:
> 
> 1. If Type(argument) is not Object, return false.
> 2. If argument is an Array exotic object, return true.
> 3. If argument is a Proxy exotic object, then
>     a. If the value of the [[ProxyHandler]] internal slot of argument is null, throw a TypeError exception.
>     b. Let target be the value of the [[ProxyTarget]] internal slot of argument.
>     c. Return IsArray(target).
> 4. Return false.

简单来说就是：抽象方法IsArray，如果用于判断的目标对象是一个Proxy的话，会先拿到其内部槽[[ProxyTarget]]，然后对[[ProxyTarget]]调用IsArray作为结果。

由于其操作是通过内部槽进行的，所以我们的基本操作是无法拦截到这个操作的，而我们的[[ProxyTarget]]是一个对象，所以结果就会返回false了。