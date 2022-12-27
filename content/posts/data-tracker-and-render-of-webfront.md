---
title: 浅谈三种主流前端框架的数据追踪渲染
date: 2022-12-23
tags: [前端, Angular, React, Vue]
categories: 知识
toc: true
description: 本文主要分析vue、react、angular之间一些本质上的不同之处，更好地理解和学习前端框架。以及从框架中学习一些优良设计。
---

本文主要分析 vue、react、angular 之间一些本质上的不同之处，更好地理解和学习前端框架。以及从框架中学习一些优良设计。

## 数据绑定与渲染

前端框架带来的第一个特点就是数据绑定，即页面上的操作会影响数据，数据的变动也会影响并更新页面。例如用户操作输入框让 `count` 变量改变。或者代码写一个定时器，定时累加 count ，页面实时刷新。在三种前端框架里都能很容易的实现。

> 注：react 是基于状态的，每次都是全新的数据，也就没有数据绑定类似的概念了，稍后讲。

有了数据绑定就牵扯页面怎么渲染数据了。那么我们就来思考三种前端框架通过数据渲染界面，采用的方式有什么不同。

## Vue

首先是 vue ，vue 支持 `v-model` 指令来实现双向绑定。用户在界面做的操作会直接影响 model 的值。

```html
<input v-model="text" />
```

本质上就是页面自动处理注册了 `onChange` 事件，事件触发就会更改 count 的值。

那么界面是什么时候渲染的呢，`vue` 通过 es5 的提供的 `Object.defineProperty()` 方法。hook 了每个对象 `get` 与 `set` 方法。从而监控对数据的操作。例如每当 count 被赋值，就可以执行类似 rerender 的方法，来重新渲染界面。这就是 vue 说自己是反应式的原因之一吧。

## Angular

angular 则是通过变更检测，也称为脏检测的机制，实现数据渲染到界面。双向绑定语法上 angular 与 vue 类似。都是一条 model 指令。

```html
<input [(ngModel)]="text" />
```

在 angular 中，变更检测执行也很简单。就是**从上至下**的比较对象值或引用地址是否发生了变化。如果变化了，那就更新视图界面。什么时候触发变更检测就是 angular 的 vue 不同的地方。vue 是通过 getter，setter。angular 则是通过 patch 了很多方法，例如按钮等控件元素的各种鼠标事件，以及一些公有方法，例如 settimeout 等等。当然也可以在变更 model 值后，手动调用脏值检测，markAsDirty。手动触发变更检测。

## React

react 则没有数据双向绑定这个概念了，组件内部通过 `setState()` 更新状态（数据）。我们也一般通过类似以下方式实现“双向绑定”类似的效果。

```jsx
render() {
	return <input value={this.text} onChange={(event) => {
		this.setState({text: event.target.value})
	}) />
}
```

为了性能考虑，这种方式不利于局部更新。所以 react 利用了虚拟 DOM 树，通过 diff 算法比较新状态与旧状态。最终打补丁的方式更新真实 dom 树（界面）。

## 结尾

简单了解了下三种主流前端框架核心功能区别。它们分别是如何追踪变化、渲染数据的。从这上面出发可以帮助我们更好地学习，理解三种框架的不同之处。

然后 vue 和 react 是怎么利用虚拟 dom 树，进行 diff 算法的。不过这超出本文的范畴，就不再赘述了，网上有很多讲的很好的文章，例如：[react 和 vue 虚拟 dom 的区别](https://juejin.cn/post/7028107265341128740)
