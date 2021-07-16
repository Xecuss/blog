---
title: ES6 Proxy 详解
date: 2021-07-11 01:01:06
tags: ['code', 'ECMAScript']
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

### 对于目标的所有操作进行记录
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

### 防止子引用出错

在实际开发中，有时候一个对象的结果可能很在很深的层级，比如这样去引用结果中的一个字段：
```typescript
const res = res.data.list[0].title;
```
然而这种操作其实是非常危险的，其中任何一个属性没有就会导致报错，然后之后的操作就无从进行了。而且对于这么长的引用做检查也是一件很麻烦的事情。当然，我们可以使用ES2020中的新特性：可选链来进行这个判断，就像这样：
```typescript
const res = res?.data?.list?.[0]?.title;
```
这样如果中间的某个值为null，整个结果会变成null而不会阻塞接下来的代码。不过可选链毕竟是一个相当新的特性，实际上我们使用Proxy也可以达到类似的效果。我们可以写这样一个Proxy，当访问这个Proxy目标上不存在的对象时，它会生成一个警告并返回一个和它一样的Proxy，这样无论我们引用多少层，操作都会被Proxy所捕获而不会出错。
```typescript
const propertyErrorhandle = (target) => {
    const handler = {
        get(target, prop, receiver) {
            if(target[prop] !== undefined) {
                return target[prop];
            }
            else {
                console.warn(`warn: read undefined property ${prop}`);
                return new Proxy(Object.create(null), handler);
            }
        }
    }

    return new Proxy(target, handler);
}
```
试用一下：
```typescript
const a = propertyErrorhandle({});

console.log(a.data.list[0]);
console.log('Hello World');
```
![](proxy-prop.jpg)

### 伪数组
伪数组指一个元素，它看起来像个数组，用起来像个数组，但是实际上不是数组（这是我自己定义的）。由于Proxy能对未定义的属性也能生效的特性，所以非常适合用来实现一个伪数组。

笔者今天介绍的一种伪数组，叫做位数组（BitArray），什么是位数组呢？我们把一个整数转换成二进制，将他的每一位看成是一个true/false，那么一个32位的无符号整型变量就可以当成一个长度为32的Boolean数组来使用。

BitArray非常适合用来存储很多的true/false值，在存储同类型的多选选项时非常节省空间。

我们希望能构建一个BitArray类，它可以这样来使用：
```typescript
const bitArray = new BitArray();
//将第四位设置为1
bitArray[3] = true;

console.log(bitArray[3]);
//此时值为(2)1000 === 8
console.log(bitArray.value);
```
使用new来实例化一个BitArray，并且既可以当成数组使用，也可以获得它的实际值来方便之后方便上传到数据库。

既然要用new来实例化，那么就要用到construct trap了：
```typescript
const BitArray = new Proxy(function(){ }, {
    construct(target, args, newTarget) {
        let data = {
            value: 0
        };
        const handler = {};
        return new Proxy(data, handler);
    }
});
```
这样我们实例化之后就能得到一个Proxy对象。
> P.S 其实不使用Proxy也能通过new得到一个Proxy对象，即在class的constructor里直接返回一个Proxy，但是笔者感觉这样写容易造成一些困惑，所以这里选择了使用Proxy的写法。

get trap是比较好写的，如果prop是一个数字，则返回对应的位上的值，这个操作可以通过移位运算和掩码1来实现，需要第几位的值就右移几位，然后和1位与；如果prop不是数字，则返回对应的原始值，代码如下：
```typescript
const handler = {
    get: (target, prop, receiver) => {
        let offset = Number(prop);
        if(isNaN(offset)){
            return target[prop];
        }
        else{
            return Boolean((target.value >> offset) & 1);
        }
    }
};
```

set trap的写法就要稍微复杂一点，将某一位offset置为1和置为0的方法是不一样的。先看置为1的情况，要将某一位置为1，也就是无论这一位原来是什么值，执行这个操作之后都一定会变为1。能满足这个需要的就是位或操作了。我们需要将这一位和1进行位或，这样无论原来的值是什么，最后这一位都会变成1。至于别的位置，和0位或就可以得到原来的值。所以我们让1左移offset位，就能构造出满足需要的二进制串了。

再看置为0的情况，类似的，如果要将某一位置为0，我们需要无论它是什么值最后都会变成0。能满足这个需要的就是位与操作。将这一位和0进行位与，其它位置和1进行位与，这样就能达成目标。所以我们先让1左移offset位，然后进行按位取反，就能构造出满足需要的二进制串了。

最后我们可以得到这样的set：
```typescript
const handler = {
    set: (target, prop, value, receiver) => {
        let offset = Number(prop);
        if(isNaN(offset)){
            throw new Error('禁止设置属性');
        }
        else{
            let trueValue = Boolean(value);
            if(trueValue){
                target.value |= (1 << offset);
            }
            else{
                target.value &= ~(1 << offset);
            }
            return true;
        }
    }
};
```
完整的代码就不贴出来了，占地方，根据上面几个拼一下就好。

最后运行之前的代码：
![](bitArray1.jpg)

当然你也可以根据喜好给这个BitArray附加更多的数组属性，比如我加上了迭代器，让它可以支持for...of操作：
```typescript
const handler = {
    get: (target, prop, receiver) => {
        if(prop === Symbol.iterator) {
            return function* (){
                let i = 0;
                for(i = 0; i < 32; i++){
                    yield this[i];
                }
                return;
            };
        }
        let offset = Number(prop);
        if(isNaN(offset)){
            return target[prop];
        }
        else{
            return Boolean((target.value >> offset) & 1);
        }
    }
}
```