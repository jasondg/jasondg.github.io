---
title: "CSS 问题记录集"
date: 2022-05-29T15:32:12+08:00
categories: ["debug"]
tags: ["css", "flex"]
draft: false
---

## Layout 相关

### flex items 撑开 container

当 flex container 内的 item 使用`flex-grow： 1`时，如果 item 的大小超出 container 的大小，会使 container 被撑开。例如动态修改 item 的 width/height 时，如果超出 container 的大小且 container 未做限制，会使 container 被撑开至最大的 item 的大小；当使用 ResizeObserver 触发 item 的大小修改时可能导致死循环。

在 Stack Overflow 上有相关的 [问题](https://stackoverflow.com/questions/36247140/why-dont-flex-items-shrink-past-content-size)，根据 W3C [Automatic Minimum Size of Flex Items](https://www.w3.org/TR/css-flexbox-1/#min-size-auto)，flex container 默认情况下会以 items 的最大宽/高作为自身的最小宽/高，所以会被过大的 item 撑开。

所以对 flex container 加上`min-width:0`/`min-height:0`就可以使 container 不被 item 撑开。
