---
title: "Firefox 中 Array.sort() 能够使用 boolean 值进行判断"
date: 2022-03-29T21:35:39+08:00
categories: ["debug"]
tags: ["Array", "sort", "Firefox"]
draft: false
---

# 前言

一次调试时，发现 Firefox 中可以通过 boolean 值来使用 Array.sort()，而在 Node.js 中仅能使用 number 类型。尽管这个问题没有造成代码错误，但还是会造成一些困惑。

# JS 引擎间差异

简单运行下测试代码，结果如下：

- Firefox

```
const arr = [{num:1}, {num:3}, {num:2}]
const numSorted = arr.sort((a, b) => a.num - b.num)
console.log(numSorted)

> Array [Object { num: 1 }, Object { num: 2 }, Object { num: 3 }]
```

```
const arr = [{num:1}, {num:3}, {num:2}]
const boolSorted = arr.sort((a, b) => a.num > b.num)
console.log(boolSorted)

> Array [Object { num: 1 }, Object { num: 2 }, Object { num: 3 }]
```

- Node.js

```
const arr = [{num:1}, {num:3}, {num:2}]
const sorted = arr.sort((a, b) => a.num > b.num)
console.log(sorted)

> [ { num: 1 }, { num: 3 }, { num: 2 } ]
```

- Deno

```
const arr = [{num:1}, {num:3}, {num:2}]
const sorted = arr.sort((a, b) => a.num > b.num)
console.log(sorted)

> [ { num: 1 }, { num: 3 }, { num: 2 } ]
```

可以发现，Node 与 Deno 同样使用 V8 引擎，仅能在 Array.sort()中使用 number 类型作为判断，而 Firefox 中能够使用 number 或 boolean 值作为判断。

# TypeScript 中 Array.sort()的类型

在 TypeScript 中 Array.sort()的类型为

```
(method) Array<T>.sort(compareFn?: ((a: T, b: T) => number) | undefined): T[]
```

尝试使用 boolean 类型的 sort(),但 TS 报出类型错误:

```
Argument of type '(a: { num: number; }, b: { num: number; }) => boolean' is not assignable to parameter of type '(a: { num: number; }, b: { num: number; }) => number'.
  Type 'boolean' is not assignable to type 'number'.ts(2345)
```

# MDN 文档与 ECMA 标准

查阅 MDN 文档[^mdn]，

> So, the compare function has the following form:
>
> ```
> function compare(a, b) {
>  if (a is less than b by some ordering criterion) {
>    return -1;
>  }
>  if (a is greater than b by the ordering criterion) {
>    return 1;
>  }
>  // a must be equal to b
>  return 0;
> }
> ```

[说明](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort#description)中并未标明可以使用 boolean 值，继续查阅 ECMA 文档[^ecma],

> 1.  If x and y are both undefined, return +0𝔽.
> 1.  If x is undefined, return 1𝔽.
> 1.  If y is undefined, return -1𝔽.
> 1.  If comparefn is not undefined, then
>     - Let v be ? ToNumber(? Call(comparefn, undefined, « x, y »)).
>     - If v is NaN, return +0𝔽.
>     - Return v.
> 1.  Let xString be ? ToString(x).
> 1.  Let yString be ? ToString(y).
> 1.  Let xSmaller be ! IsLessThan(xString, yString, true).
> 1.  If xSmaller is true, return -1𝔽.
> 1.  Let ySmaller be ! IsLessThan(yString, xString, true).
> 1.  If ySmaller is true, return 1𝔽.
> 1.  Return +0𝔽.

[标准](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.sort)中同样也未指出可以使用 boolean 值进行判断。

# 总结

Firefox 中能够使用 boolean 值进行判断，可能是 Firefox 的实现问题，在相关文档与标准中并没有相关说明。实际写代码时，在 lint 工具等约束后应该并不会产生异常问题，仅是出于好奇进行了一些检索，简单记录一下。

[^mdn]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
[^ecma]: https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.sort
