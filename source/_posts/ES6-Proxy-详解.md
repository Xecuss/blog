---
title: ES6 Proxy 详解
date: 2021-07-11 01:01:06
tags:
---
Vuejs是一个非常好用的前端框架，它最大的特征就是「响应式」。即将模型和视图绑定起来，只需要直接修改模型的值，视图会自动更新。为了实现这一个特征，首先需要的就是能监听目标的变化。而我们都知道，在vue2和vue3中的响应式实现是不一样的，vue2中使用的是getter/setter，而vue3中使用的则是ES6的一个新对象——Proxy。那么Proxy相对于getter/setter有哪些优点呢？今天我们就来聊聊这个ES6的Proxy。

## Proxy基础
首先看看Proxy最基础的定义：
> Proxy 对象用于创建一个对象的代理，从而实现基本操作的拦截和自定义（如属性查找、赋值、枚举、函数调用等）。

这里的重点是：**创建一个对象的代理**，看到代理，熟悉设计模式的朋友可能马上就反应过来了：Proxy实现的就是设计模式中的代理模式。

> 代理模式的定义：由于某些原因需要给某对象提供一个代理以控制对该对象的访问。这时，访问对象不适合或者不能直接引用目标对象，代理对象作为访问对象和目标对象之间的中介。

这个定义其实是很好理解的，甚至可能看名字就知道了。可能有的朋友知道，在网络领域也有一个同名的叫做Proxy的技术。例如Nginx实现的就是一种反向Proxy，它根据我们发送过去的请求的内容，对请求做出各种处理，然后再交给真正的处理程序。而我们ES6中的Proxy其实实现的功能和它也是一样的。

## Proxy的语法

```typescript
const p = new Proxy(target, handler);
```
* target：
要使用 Proxy 包装的目标对象（可以是任何类型的对象，包括原生数组，函数，甚至另一个代理）。
* handler：
一个通常以函数作为属性的对象，各属性中的函数分别定义了在执行各种操作时代理 p 的行为。

handler 对象是一个容纳一批特定属性的占位符对象。它包含有 Proxy 的各个捕获器（trap）。

所有的捕捉器是可选的。如果没有定义某个捕捉器，那么就会保留源对象的默认行为。

* handler.getPrototypeOf()：
Object.getPrototypeOf 方法的捕捉器。
* handler.setPrototypeOf()：
Object.setPrototypeOf 方法的捕捉器。
* handler.isExtensible()：
Object.isExtensible 方法的捕捉器。
* handler.preventExtensions()：
Object.preventExtensions 方法的捕捉器。
* handler.getOwnPropertyDescriptor()：
Object.getOwnPropertyDescriptor 方法的捕捉器。
* handler.defineProperty()：
Object.defineProperty 方法的捕捉器。
* handler.has()：
in 操作符的捕捉器。
* handler.get()：
属性读取操作的捕捉器。
* handler.set()：
属性设置操作的捕捉器。
* handler.deleteProperty()：
delete 操作符的捕捉器。
* handler.ownKeys()：
Object.getOwnPropertyNames 方法和 Object.getOwnPropertySymbols 方法的捕捉器。
* handler.apply()：
函数调用操作的捕捉器。
* handler.construct()：
new 操作符的捕捉器。

## 和setter/getter对比

### 语法方面：
1. getter/setter是直接定义到源对象的属性，而Proxy会生成一个新的Proxy对象
2. getter/setter只能定义现存的属性，而对于未定义的无效，这也是为什么vue2里不能直接给对象添加key的原因
3. getter/setter只能对get/set生效（废话），而Proxy能对前面列出的包括in操作等13种操作有效

### 兼容性方面：
由于Proxy能拦截的13种操作中，除了get/set以外都无法被拦截，所以Proxy是一个无法polyfill的特性，这意味着IE将会彻底被放弃支持。

### 性能方面：
关于性能，笔者看到有[两篇文章](https://thecodebarbarian.com/thoughts-on-es6-proxies-performance)都说性能是setter/getter比Proxy更快，不过那两篇文章的时间都比较早了，笔者使用最新版Edge浏览器在[这个网站](https://www.measurethat.net/Benchmarks/Show/11510/2/getter-setter-vs-proxy-vs-events#latest_results_block)跑了一下，发现性能应该是Proxy更快一点。
![](benchmark1.jpg)
笔者自己也使用node v12.13写了一个简单的测试，结果也是Proxy快一点
![](benchmark2.jpg)
不过，两者差距并不是非常大，日常开发应该不需要担心。

## Reflect

说到ES6的Proxy，还有一个和它紧密相关的东西，叫做Reflect(反射)，这是Reflect的定义：
> Reflect 是一个内置的对象，它提供拦截 JavaScript 操作的方法。这些方法与proxy handlers的方法相同。Reflect不是一个函数对象，因此它是不可构造的。

其实很多其他的编程语言也有反射的概念，反射一般用于*在运行时拿到一个类的所有属性和方法*。但是ECMAScript本身就是一种动态类型的语言，使用反射似乎是没有必要的。那为什么会增加一个Reflect对象呢？下面的实例部分将告诉你原因。

## 使用Proxy的实例

### 对于目标的所有修改都进行记录
首先是一个简单的例子，我们现在对于一个对象建立一个日志功能，每次对对象进行操作的时候就输出当前操作类型和结果。那么很容易就能写出以下代码：
```typescript
const p = new Proxy(target, {
    get(target, prop, reciver) {
        console.log(`Get Value ${prop}: ${target[prop]}`)
    },
    set(target, prop, value, reciver) {
        console.log(`Set Value ${prop}: ${target[prop]} -> ${value}`)
    },
    // ......
})
```
像这样，将所有的trap都实现一遍就可以了。但是这似乎不够优雅，我们的每一个handler其实代码都是类似的，是否可以直接通过相同的写法来实现呢？
然而实际上你会发现，有几个handler似乎并不能直接写成类似的形式，例如has trap：
```typescript
{
    has(target, prop, reciver) {
        console.log(`Has Opt get  ${prop in target}`)
        return prop in target;
    }
}
```
这个has的操作和之前就大不相同了，有没有什么办法可以让它也能写成相同的形式呢？这时候就是Reflect出场的时刻了，我们可以将in操作写成这样的形式：
```typescript
{
    has(target, ...args) {
        console.log(`Has Opt get ${Reflect.has(target, ...args)}`);
        return Reflect.has(target, ...args);
    }
}
```
由于Reflect上的操作和Proxy是一一对应的，所以我们可以直接使用Reflect上对应的方法来调用target本身的操作。

实际上，我们还可以利用Reflect的特性，直接获得所有可以进行Proxy的trap，然后直接调用，这样看起来代码就能十分优雅：
```typescript
const proxyLogger = (target) => {
    const handler = Object.getOwnPropertyNames(Reflect)
        .reduce((current, key) => {
            current[key] = (target, ...args) => {
                const res = Reflect[key](target, ...args);
                console.log(`${key} opt get ${res}`);
                return res;
            };
            return current;
        }, Object.create(null));

    return new Proxy(target, handler);
}
```