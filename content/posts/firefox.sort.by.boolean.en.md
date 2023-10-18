---
title: "Array.sort() in Firefox can use boolean values for judgment"
date: 2022-04-03T20:23:28+08:00
categories: ["debug"]
tags: ["Array", "sort", "Firefox"]
draft: false
---

Once I was debugging, I found that in Firefox `Array.sort()` can take boolean values in addition to number values. However, in Node.js `Array.sort()` only supports number values.
Although it didn't cause bugs, I'm curious about the difference.

# Difference between JS engines

Here are some tests:

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

It turns out that in Node.js and Deno (both use V8), `Array.sort()` takes only numbers. While in Firefox, `Array.sort()` can take both.

# Array.sort() in TypeScript

In TypeScript, `Array.sort()` is

```
(method) Array<T>.sort(compareFn?: ((a: T, b: T) => number) | undefined): T[]
```

Try to use boolean values in `Array.sort()`, but Typescript throws error:

```
Argument of type '(a: { num: number; }, b: { num: number; }) => boolean' is not assignable to parameter of type '(a: { num: number; }, b: { num: number; }) => number'.
  Type 'boolean' is not assignable to type 'number'.ts(2345)
```

# Array.sort() in MDN and ECMA

Let's check out the document on MDN[^mdn], it says

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

The [document](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort#description) doesn't indicate the `sort()` can take boolean values.
Continue to check out the ECMA standard[^ecma], which says

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

[It](https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.sort) also doesn't point out the `sort()` can take boolean values.

# Summary

`Array.sort()` in Firefox can take boolean values, but there is no relevant description in documents. It may just be an implementation difference that probably not causing bugs.

[^mdn]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
[^ecma]: https://tc39.es/ecma262/multipage/indexed-collections.html#sec-array.prototype.sort
