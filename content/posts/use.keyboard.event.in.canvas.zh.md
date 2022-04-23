---
title: "在 canvas 中监听键盘事件"
date: 2022-04-23T21:43:59+08:00
categories: ["debug"]
tags: ["canvas", "keyboard event"]
draft: false
---

# 问题与解决方案

## 一般情况

一次在 React 中进行 canvas 开发时，发现 canvas 的 onKeyDown 无法生效。

刚开始不清楚是否是 event delegate 的问题，在 canvas 上绑定 onClick，测试能够正常触发事件。

检索后找到了 Stack Overflow 上的一个[回答](https://stackoverflow.com/questions/12886286/addeventlistener-for-keydown-on-canvas)，其中提到 canvas 默认无法被 focus，所以无法绑定键盘事件。

其给出的解决方案很简单

```
<canvas tabindex="1"></canvas>
```

`tabindex`使 canvas 能够被 focus，从而能正常监听键盘事件。

## 第三方库创建的 canvas

添加`tabindex`的方式虽然很简洁，但对于第三方库创建的 canvas 而言，直接修改仍然存在问题。如果第三方库没有提供相应功能，尽管通过 DOM 方法能够修改 canvas 的属性，侵入式的修改也可能造成潜在问题。

在 canvas 不能直接控制的情况下，可以在`document.body`上绑定`keydown`事件，并关联相应的处理逻辑。这种方式使用了全局对象，在较大的项目中会增加维护成本。

另一种方式是通过 canvas 绑定鼠标/触控事件，产生交互时设置 canvas 的外层可控 element 为焦点，并在外层 element 上监听键盘事件。这种方式增加了一些复杂度，不过能更好地约束作用域。

## 多 canvas 交互

同一页面存在多个 canvas 需要监听键盘事件时，如之前部分所说，一种方式是，在使用`document.body`监听的情况下，记录一个全局的 LRU 列表或最新指向的引用/指针，每次 canvas 有交互时记录最新的指向，从而执行相应 canvas 的处理逻辑。

而使用另一种方式，即切换 focus 至外层时，多个 canvas 的交互并不互相干扰，这也是此种方式的一个优势。

# 一些拓展

## activeElement

可以使用`document.activeElement`获得当前的 focus 对象，便于调试[^whatwg-activeelement]。

## focus steps

因为好奇 canvas 的 focus 为什么不会冒泡至 parent element，检索了 whatwg 上关于 focus 的部分[^whatwg-focus]，根据 focus 的[获取步骤](https://html.spec.whatwg.org/multipage/interaction.html#get-the-focusable-area)，focus 仅会向内部寻找 focusable area，而不会像事件那样向上冒泡。当 focus 失败时就会返回 document 作为 focus 对象。

[^whatwg-activeelement]: https://html.spec.whatwg.org/multipage/interaction.html#focus-management-apis
[^whatwg-focus]: https://html.spec.whatwg.org/multipage/interaction.html#focusing-steps
