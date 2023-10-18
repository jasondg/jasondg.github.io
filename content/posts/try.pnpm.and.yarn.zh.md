---
title: "通过 corepack 使用 pnpm 与 yarn 作为包管理器"
date: 2022-04-03T21:02:50+08:00
categories: ["log"]
tags: ["pnpm", "yarn", "nodejs"]
draft: false
---

Node.js 16 中加入了 [corepack](https://nodejs.org/api/corepack.html) 工具，能够直接启用 pnpm 与 yarn 作为包管理器，而不需要通过 npm 进行额外的安装。

尝试了在日常开发中使用 pnpm 与 yarn 替代 npm,记录了一些体验上的优化和实际使用中遇到的问题。

# 通过 corepack 启用 pnpm 与 yarn

corepack 目前还是实验性功能，需要按照文档[^corepack]进行手动开启。

- 执行 `corepack enable` （如果 Node.js 未以 user 权限进行安装，需要 sudo 或 root 权限进行运行，windows 上需要以管理员模式运行）
- 配置 `package.json` 中的 `packageManager` （使用 pnpm 时即使不配置也可正常使用，使用 yarn 时如果不进行`packageManager`的配置，会使用 yarn 1 而非 yarn 2+）
- 如果需要在离线环境下启用，可以使用 `corepack prepare` 进行预载

# pnpm 与 yarn 的优势

## Phantom dependencies

Phantom dependencies 是指依赖被安装后，非该依赖链上的代码也可以使用该依赖，例如

```
// root
{
    "dependencies": {
        "a": "^1.0.0",
        "c": "^1.0.0"
    }
}

// a
{
    "dependencies": {
        "b": "^1.0.0"
    }
}
```

安装后文件路径为

```
/node_modules
    /a
    /b
    /c
```

其中 b 依赖可以被直接`import`/`require`，这造成了隐式依赖，可能出现不易发现的问题，详细说明可以参见 rushjs 的解释[^phantom-deps]。

## NPM doppelgangers

NPM doppelgangers 是指同一依赖存在不同版本时，尽管 npm 进行了 hoist，仍会产生多个重复的依赖包，例如

```
{
    "dependencies": {
        "a": "^1.0.0", => { "e": "^1.0.0" }
        "b": "^1.0.0", => { "e": "^1.0.0" }
        "c": "^1.0.0", => { "e": "^2.0.0" }
        "d": "^1.0.0"  => { "e": "^2.0.0" }
    }
}
```

安装后文件路径为

```
// npm 进行 hoist

/node_modules
    /a
    /b
    /c
        /e@2
    /d
        /e@2
    /e@1

// 或

/node_modules
    /a
        /e@1
    /b
        /e@1
    /c
    /d
    /e@2
```

这样会导致重复依赖的存在不可避免，影响性能甚至会导致编译工具的一些 bug，详细说明可以参见 rushjs 的解释[^npm-doppelgangers]。

## pnpm 与 yarn 的优化

pnpm 采用 symlink 方式[^pnpm-symlink]，在项目中通过建立共享存储的 hardlink 来创建非平铺（non-flat）的 node_modules，从而避免了上述的问题，同时也减少了硬盘空间的占用。

yarn (2+) 采用 [Plug'n'Play](https://yarnpkg.com/features/pnp) 方式，通过一个 cjs 控制依赖的加载，以 zip 方式提供依赖，从而能够约束 phantom dependencies 问题，同时减少了硬盘的 IO。  
并且在这种方式下，将依赖 commit 进 git 成为可能，配合 yarn 的 [zero install](https://yarnpkg.com/features/zero-installs/) 功能，能够使项目更加稳健，不因依赖安装产生问题。

# 使用中的一些问题

## yarn

在遇到一些不很规范的包时，yarn PnP 会因为包调用 phantom dependencies 而出现依赖缺失，需要使用 loose 模式[^yarn-pnp-loose]绕过。

同时，yarn PnP 会以 zip 格式安装依赖，需要额外安装 [sdk](https://yarnpkg.com/getting-started/editor-sdks/) 与相关插件等使编辑器能够正常工作，在一些受限环境下使用不友好。

## pnpm

使用 hardlink 创建 non-flat 的 node_modules 有时候会导致文件读取问题。  
一个例子为，曾经遇到过在 Windows 10 环境下，通过 vite 运行依赖 @mui/icons-material 的应用时，出现 too many files 问题。  
临时的解决方案为使用

```
// ok
import { xxx } from "@mui/icons-material/XXX"
```

替代

```
// error: too many files
import { xxx } from "@mui/icons-material"
```

但这个问题在 Linux 下没能复现，不能确定是什么原因。并且由于手边没有能复现的环境，暂时也不能排查问题。

# 总结

Node.js 增加 corepack 后，相信 yarn 与 pnpm 会被更为广泛地应用，从而解决一些 npm 一直存在的问题。

从个人使用角度，yarn 与 pnpm 也确实提升了体验与开发效率，值得作为 Node.js 下主要的包管理工具。

[^corepack]: https://nodejs.org/api/corepack.html#workflows
[^phantom-deps]: https://rushjs.io/pages/advanced/phantom_deps/
[^npm-doppelgangers]: https://rushjs.io/pages/advanced/npm_doppelgangers/
[^pnpm-symlink]: https://pnpm.io/motivation
[^yarn-pnp-loose]: https://yarnpkg.com/features/pnp#pnp-loose-mode
