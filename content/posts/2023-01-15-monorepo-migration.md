+++
title = "把多个前端项目合进一个 Monorepo 是怎样的体验"
date = 2023-01-15
draft = false
categories = ["工程化"]
tags = ["Monorepo", "Turborepo", "pnpm", "依赖管理", "CI/CD"]
description = "记录一次从「多仓库」迁移到 Monorepo 的过程：如何选型、划分包、管理依赖，以及 CI 流水线怎么改。"
+++

之前团队的前端项目是这样管理的：

- 每个系统一个 Git 仓库；  
- 各自有一套构建脚本、一份依赖清单；  
- 公共组件、工具要么复制粘贴，要么用「半私有」的 NPM 包。  

结果就是：

- 组件 bug 修一份，要手动同步 N 个仓库；  
- 依赖升级经常有「版本错乱」；  
- CI 流水线一堆重复配置。  

于是我们决定把几个核心前端项目合到一个 Monorepo 里。  
这篇文章就是那次迁移的实战记录：**工具选择、包划分、依赖管理、CI 改造，全流程。**

---

## 1. 为什么要搞 Monorepo？

总结下来就是三点：

1. **公共代码复用**  
   - 组件库、Hooks、工具函数不用再复制粘贴；  
   - bug 修一次，多项目同步升级。

2. **一致的工程规范**  
   - 统一 ESLint/Prettier/Commit 规范；  
   - 统一构建脚手架和发布流程。

3. **协同开发体验好**  
   - 打开一个仓库就能看到相关项目；  
   - 修改组件立刻能在多个应用里验证。

当然，Monorepo 不是银弹，仓库会变大、CI 要更精细。  
所以「是否值得」的标准是：

> **项目之间有明显共享，且会长期共同演进。**

---

## 2. 工具选型：pnpm + Turborepo

我们调研了一圈：Lerna、Nx、Yarn Workspaces、pnpm Workspace、Turborepo……  
最后选的是：**pnpm Workspace + Turborepo**，原因：

- pnpm：硬链接节省磁盘空间、依赖管理清晰、速度快；  
- Turborepo：任务编排 + 缓存好用，学习成本相对较低。

### 2.1 仓库结构

```bash
repo-root
├── apps
│   ├── manager-web        # 管理端
│   ├── investor-web       # 投资人 Web
│   └── mini-program       # 小程序（uni-app 等）
├── packages
│   ├── ui-components      # 通用 UI 组件库
│   ├── shared-utils       # 工具函数
│   └── eslint-config      # 统一 ESLint 配置
├── package.json
├── pnpm-workspace.yaml
└── turbo.json
```

`pnpm-workspace.yaml`：

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

---

## 3. 包划分：应用 vs 库

在 Monorepo 里，我习惯先分两类：

1. **Apps（应用）**  
   - 真正对外跑的前端应用：管理端、运营端、小程序 H5 等；  
   - 有自己的路由、页面、部署配置。

2. **Packages（库）**  
   - 通用组件、工具库、hooks、eslint/prettier 配置等；  
   - 尽量无副作用，可发布到私有 NPM（可选）。

举例：

- `@org/ui-components`：封装常用表单、表格、弹窗、布局；  
- `@org/shared-utils`：日期格式化、金额处理、权限工具等；  
- `@org/eslint-config`：统一 eslint 配置，一行引用。

这样每个 App 只需要：

```jsonc
// apps/manager-web/package.json
{
  "dependencies": {
    "@org/ui-components": "workspace:*",
    "@org/shared-utils": "workspace:*"
  }
}
```

pnpm 会自动把 workspace 内的包链接起来。

---

## 4. 依赖管理：哪些在根，哪些在各自

迁移过程中最容易乱的就是依赖。我们的原则是：

1. **构建相关 / 工程工具放在根**  
   - 比如：TypeScript、ESLint、Prettier、Vitest、Commitlint 等；  
   - 通过 `extends` 或脚本在各包中复用。

2. **业务运行时依赖尽量在各包**  
   - 如：Vue/React、UI 库、axios 等；  
   - 避免所有 App 被同一个版本绑死（但核心技术栈尽量对齐）。

3. **禁止「幽灵依赖」**  
   - 用 `pnpm` 默认的严格模式，防止 A 包偷偷用到根依赖；  
   - 依赖谁，就在自己的 `package.json` 写清楚。

迁移初期踩的一个坑是：**很多旧项目依赖是「能跑就行」模式**，合进 Monorepo 后这些问题会暴露出来，反而逼着我们把依赖梳理干净。

---

## 5. Turborepo：任务编排 & 缓存

`turbo.json` 示例：

```jsonc
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "lint": {
      "outputs": []
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "build/**"]
    },
    "test": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

特点：

- **局部构建**：改了哪个包，就只跑那条链路；  
- **缓存**：相同输入（代码 + 环境）下，build/test/lint 可以直接命中缓存；  
- CI 上也能复用缓存，大大加快速度。

开发体验上：

```bash
pnpm turbo run dev --filter=manager-web
pnpm turbo run lint
pnpm turbo run build --filter=./apps/*
```

不需要在每个目录来回 `cd`。

---

## 6. CI 改造：从「多条流水线」到「一条智能流水线」

原来的形态是：

- 每个仓库一条 CI；  
- 每个 job 都是「install → lint → test → build → deploy」。

Monorepo 后，我们做了几件事：

1. **安装依赖一步到位**  
   - 在根目录跑一次 `pnpm install`。

2. **利用 Turborepo 只构建受影响项目**  
   - CI 脚本直接跑：`pnpm turbo run build --filter=...`；  
   - Turborepo 根据变更文件自动决定要跑哪些包。

3. **部署仍然分项目**  
   - 比如 `apps/manager-web` 成功构建后，触发对应的部署 job；  
   - 每个 App 仍然有自己的环境变量、部署目标。

简化后的 CI 流程更像：

1. 安装依赖；  
2. 执行 turbo 的 `lint/test/build`；  
3. 根据构建成功的 App 列表触发对应部署。

---

## 7. 迁移过程中的坑与经验

1. **一口吃太多会噎着**  
   - 刚开始不要一次性把所有项目拖进 Monorepo；  
   - 先选关系紧密的两三个项目试点，跑顺了再拉更多。

2. **公共包边界要克制**  
   - 一开始很容易「什么都想抽成包」；  
   - 建议从真正稳定、复用度高的模块开始，比如 UI 组件库、工具函数。

3. **文档和脚本要跟上**  
   - 新同学要知道：哪个目录放什么、如何本地启动、如何只构建某个 App；  
   - 在根目录放一份 `CONTRIBUTING.md` 非常有用。

4. **版本升级要有节奏**  
   - 核心依赖版本升级前，先在一个 App 验证；  
   - 通过后再统一升级并跑一轮 CI。

---

## 8. 小结

把多个前端项目合进 Monorepo 的体验，总结下来：

- 前期迁移成本不低，需要 touch 很多旧代码和脚本；  
- 一旦跑顺，**公共能力迭代、依赖管理、CI 开销**都会明显改善；  
- 更重要的是，**整个前端工程体系被迫梳理了一遍**，长期收益很大。

如果你也在考虑 Monorepo，可以先从：

1. 梳理公共代码；  
2. 选一两套关联紧密的项目试点；  
3. 用 pnpm + Turborepo 跑出第一条流水线开始。

剩下的，就让这套体系在实践中一点点长起来。

