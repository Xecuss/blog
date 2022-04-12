---
title: 如何在tsx中使用vue-router4的keep-alive
tags:
  - code
  - vue
  - typescript
abbrlink: 53167
date: 2021-03-21 21:09:04
---
vue3已经发布了有一段日子了，作为一个坚定的typescript使用者，我早已对vue3大大优化的ts支持垂涎三尺了，也已经用vue3做了一些个人小项目了。由于希望在模板中也能享受智能语法提示和类型检查，那传统的.vue形式的单文件组件就满足不了我的需求了，必须使用tsx的方案，然而在使用tsx的时候碰到了不少的坑，这次这个就是其中之一。

众所周知，vue-router是vue的官方前端路由方案。在vue2的时代我们如果要使用带缓存的路由的话，一般会这么写：
```html
<template>
    <keep-alive>
        <router-view />
    </keep-alive>
</template>
```
于是来到vue3我直接如法炮制：
```tsx
return <KeepAlive>
    <RouterView />
</KeepAlive>;
```
嗯，果然没生效。。。

于是在必应上一通搜索，得到如下答案：
```html
<router-view v-slot="{ Component }">
    <keep-alive include="Home,About">
        <component class="view" :is="Component" />
    </keep-alive>
</router-view>
```
啊这。。。

首先v-slot这种用法在tsx里应该是不能直接这么使用的，毕竟tsx不比模板，**写tsx的本质其实是在写渲染函数**，于是去翻阅[babel-tsx-plugin](https://github.com/vuejs/jsx-next/tree/dev/packages/babel-plugin-jsx)的文档，可以看到：
> 注意: 在 jsx 中，应该使用 v-slots 代替 v-slot
> ```tsx
> const App = {
>    setup() {
>         const slots = {
>             foo: () => <span>B</span>,
>         };
>         return () => (
>             <A v-slots={slots}>
>                 <div>A</div>
>             </A>
>         );
>     },
> };
> ```
乍一看好像写法非常奇怪，其实想一下之前那句话就知道了：**写tsx的本质其实是在写渲染函数**；而所谓的插槽slots，其实也不过就是props上的一个属性而已，**插槽的本质其实也是一个渲染函数**，我们在模板中写```v-slot=" slotValue"```时，实际上就是写了一个参数为slotValue的渲染函数。

那这里的处理也就很明确了，router-view里的默认slot里是一个参数，然后这个参数带有一个Component属性。我们可以得到这样的一个渲染函数，我这里使用了另一种插槽的写法：
```tsx
return <RouterView>
    {{
        default: ({Component}: { Component: () => JSX.Element }) => {
            return <KeepAlive>
                <Component />
            </KeepAlive>
        }
    }}
</RouterView>
```
总结一下，其实就是传入slots函数，对其参数进行解构，获得Component参数。不同于模板需要使用```<component :is="" />```的语法，在tsx里完全可以直接渲染Component。

使用tsx写vue组件虽然获得了语法智能提示和类型检查，不过由于并非原生支持，vue对于tsx的支持毕竟还是有限的，在使用的过程中还是会碰到各种坑点。