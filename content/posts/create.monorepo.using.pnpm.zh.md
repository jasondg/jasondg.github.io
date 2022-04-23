---
title: "使用 pnpm 以 Monorepo 方式建立项目"
date: 2022-04-04T15:01:39+08:00
categories: ["log"]
tags: ["pnpm", "monorepo", "workspace"]
draft: false
---

# 为什么需要 Monorepo

在开发中，功能模块间的耦合会导致项目难以维护，因此需要分包来进行解耦合。

但是单纯地进行分包，就需要单独发布，单独安装，会增加一些繁琐的步骤。并且，快速迭代开发时，一个包需要依赖其他的包的最新修改，此时，因为需要操作相关包发布更新，开发效率会有一定降低。

而 Monorepo 将所有包都放置于同一个项目内，在分包的前提下通过直接链接本地依赖，解决了等待更新的问题。

# 如何创建 Monorepo

以下操作使用 pnpm 作为包管理器，如果还未安装，可以参见 [通过 corepack 使用 pnpm 与 yarn 作为包管理器](/posts/try.pnpm.and.yarn) 进行 pnpm 的启用。

首先在项目根目录下创建`pnpm-workspace.yaml`文件，添加需要作为 workspace 的目录项，例如 pnpm 的官方示例[^pnpm-workspace-yaml]：

```
packages:
  # all packages in subdirs of packages/ and components/
  - 'packages/**'
  - 'components/**'
  # exclude packages that are inside test directories
  - '!**/test/**'
```

同时新建 workspace 的文件夹，

```
/packages
    /someWorkspace
    /web
    /...
```

之后就可以进行依赖的添加。如果需要在项目全局进行安装，需要添加`-w`参数，而如果在 workspace 内进行安装，需要添加`--filter <workspace>`参数，同时可以使用`-r`参数为所有包添加依赖，例如：

```
// install to monorepo
pnpm add react -w

// install to workspace
pnpm add react --filter @project/someWorkspace

// install recursively
pnpm add react -r
```

如果需要依赖其他 workspace，可以执行

```
pnpm add @project/someWorkspace --filter @project/web
```

pnpm 会自动处理为 workspace 依赖，例如：

```
{
    "dependencies": {
        "@project/someWorkspace": "workspace:^1.0.0"
    }
}
```

并且在发布时会替换为正确的依赖项。

同理，`package.json` 中的 scripts 也是类似的配置：

```
// monorepo
{
    "scripts": {
        "dev:web": "pnpm dev --filter @project/web"
    }
}

// workspace (@project/web)
{
    "scripts": {
        "dev": "start dev server"
    }
}

```

这样就建立了基本的 Monorepo 项目。

如果使用的是 yarn，可以参见[官方文档](https://yarnpkg.com/features/workspaces/)。

选择 pnpm 主要是因为 pnpm 的实现方式更直接，且其在个人开发环境下对硬盘更友好，安装速度会有一定的提升。如果是组织协作等场景，还需要视具体情况选择包管理器。

# 遇到的问题

## 使用 TypeScript

在 Monorepo 中使用 TypeScript 时，会在 `import` 时提示

    Cannot find module '@project/someWorkspace' or its corresponding type declarations.ts(2307)

因此需要进行`tsconfig.json`的配置，添加相应 workspace 的 paths，例如：

```
{
    "compilerOptions": {
        ...
        "paths": {
            "@project/web": ["./packages/web/src"],
            "@project/someWorkspace": ["./packages/someWorkspace/src"]
        }
    }
}
```

## 使用 ESLint

在 Monorepo 中使用 ESLint 与 eslint-plugin-import / eslint-import-resolver-typescript 时，会在 `import` 时提示

    Unable to resolve path to module '@project/someWorkspace'. eslint(import/no-unresolved)

因此需要进行`.eslintrc.json`的配置，禁用相应的规则

```
{
    "rules": {
        "import/no-unresolved": 0
    }
}
```

这个问题也有一些讨论[^github-1][^github-2][^github-3][^stackoverflow-1]，但尝试后发现，最直接的解决方法就是禁用`import/no-unresolved`规则。因为 TypeScript 也会检查`import`语句，所以目前看还未造成太大问题。

## 工程管理粒度

Monorepo 方式会将所有包置于一个项目中，对其中一个包的修改也需要满足整体项目的管理要求。

一方面，这强化了工程管理，对各个包的管理更为紧密，使得整个项目可维护性更高；另一方面，这也增加了管理成本，丧失了一些灵活性。

# 总结

使用 Monorepo 方式建立项目，能够更好地进行分包管理，降低项目中多个模块之间的耦合度，同时避免单独分包的繁琐操作，提高迭代效率。但同时也会增加管理成本，降低一些灵活性。

因此，需要根据实际情况选择是否使用 Monorepo。而在合适的情况下，Monorepo 是很值得尝试的，尽管目前可能遇到一些小问题，但相信随着技术完善，更多的 corner cases 会被覆盖。

[^pnpm-workspace-yaml]: https://pnpm.io/pnpm-workspace_yaml
[^github-1]: https://github.com/import-js/eslint-plugin-import/issues/2247
[^github-2]: https://github.com/alexgorbatchev/eslint-import-resolver-typescript/issues/45
[^github-3]: https://github.com/alexgorbatchev/eslint-import-resolver-typescript/issues/36
[^stackoverflow-1]: https://stackoverflow.com/questions/69932369/setting-up-eslint-import-resolver-typescript-in-monorepo
