+++
title = "2019–2020 我的前端工作流与常用工具清单"
date = 2020-12-12
draft = false
categories = ["工具与方法论"]
tags = ["工作流", "开发工具", "效率"]
description = "整理那段时间最顺手的一套编辑器插件、调试方式和项目模板。"
+++

每隔一段时间，我都会回头看一下：**自己每天在用的工具和工作流，哪些真的有效，哪些只是「习惯了就一直没动」**。

这篇文章算是对 2019–2020 这两年自己前端工作流的一次归档，顺便给以后想重构环境的自己当个参考。

---

## 1. 编辑器与基础环境

### 1.1 VS Code 为主，WebStorm 偶尔上场

这两年主要是 VS Code 打主力，原因很简单：

- 生态丰富（插件多）；
- 配置灵活（`settings.json`、`tasks` 等等）；
- 打开仓库速度相对快。

WebStorm 的优势在于：

- 更强的「开箱即用」体验（尤其是对 TS / 重构支持）；
- 有些场景下更稳，但是资源占用也更高。

**我的做法：**

- 日常写业务、处理多个小项目 → VS Code；
- 需要重构大型 TS 项目、或者做复杂的重命名/移动 → 偶尔打开 WebStorm。

---

### 1.2 VS Code 常驻插件

只列一些我真正离不开的：

- **ESLint**：保存自动修复格式/语法问题。
- **Prettier**：统一代码风格（也可以用 ESLint 负责格式化）。
- **Vetur / Volar**：Vue 项目对应的语法高亮与提示。
- **GitLens**：在代码行旁边直接看到最近一次提交记录。
- **Path Intellisense**：自动补全文件路径。
- **TODO Highlight**：高亮 TODO / FIXME，提醒自己有坑没填。

设置里会开：

```json
"editor.formatOnSave": true,
"editor.codeActionsOnSave": {
  "source.fixAll.eslint": true
}
```

这样每次保存就是一次「小型代码体检」。

---

## 2. Git 工作流

这两年基本稳定在一套很简单的 Git 流程：

### 2.1 分支模型

- `master` / `main`：生产分支，只接收经过测试的代码。
- `develop`：日常开发分支。
- `feature/*`：功能分支，例如 `feature/order-list`。
- `hotfix/*`：线上紧急修复。

实际团队里可能会再加 release 分支，但我自己的 side project 通常就前三种就够了。

---

### 2.2 提交信息规范

粗暴实用版的前缀规范：

- `feat:` 新功能
- `fix:` 修 bug
- `refactor:` 重构（无新功能无修复）
- `chore:` 杂项（脚本、配置、依赖升级）
- `docs:` 文档修改
- `style:` 格式调整（不影响逻辑）

例如：

```text
feat: add order list page
fix: handle null customer name
chore: bump axios to 0.21.0
```

配合 GitLens 或 PR 页面，可以一眼看出「这一批提交都在干嘛」。

---

## 3. 前端项目模板与脚手架

### 3.1 Vue 项目的基础模板

2019–2020 的 Vue 项目我一般会自带这些约定：

- 使用 Vue CLI / 自定义脚手架；
- 内置：

  - Axios 封装（请求/响应拦截器、统一错误处理）；
  - 基础布局（顶部导航 + 侧边菜单）；
  - 路由/菜单配置结构；
  - 基本权限控制（登录态校验）。

目录结构大致是：

```bash
src
├── api        # 接口封装
├── components # 通用组件
├── router     # 路由配置
├── store      # Vuex
├── views      # 页面
└── utils      # 工具方法
```

这样的好处是：每开一个新项目，不用再从头搭这些「基建」，更快进入业务开发。

---

### 3.2 常用 NPM 脚本

在 `package.json` 里基本会有：

```json
"scripts": {
  "dev": "vue-cli-service serve",
  "build": "vue-cli-service build",
  "lint": "eslint --ext .js,.vue src",
  "lint:fix": "eslint --ext .js,.vue src --fix"
}
```

再配合 CI：

- 提交前本地至少跑一遍 `lint`；
- CI 里对 PR 跑一次 `lint + build` 做基本兜底。

---

## 4. 调试与排查问题

### 4.1 浏览器 DevTools

核心几个面板：

- **Elements**：检查 DOM / 样式，快速定位布局问题。
- **Network**：看接口请求、响应时间、状态码、返回数据。
- **Sources**：打断点调试，结合 SourceMap。 
- **Performance**：排查长任务、掉帧问题。
- **Application**：本地存储、Cookie、IndexedDB。

习惯：

- 接口联调时，习惯打开 Network，看请求是否走到、参数是否正确；
- 遇到「页面白屏」之类的问题，第一反应就是看 Console 和 Network 是否有报错。

---

### 4.2 移动端调试

常用几种方式：

1. Chrome 远程调试：  
   - 手机打开页面，USB 连接电脑；  
   - Chrome 地址栏输入 `chrome://inspect`；  
   - 找到对应设备，点击 inspect。

2. 使用 vConsole：  
   - 在 H5/m 站里临时集成 vConsole；  
   - 方便在非开发环境简单排查问题（不推荐长期上线）。

3. 小程序开发者工具 / 模拟器：  
   - 直接用微信开发者工具等自带调试能力。

---

## 5. 一些提效小工具

### 5.1 命令行工具

- **npx**  
  不用全局安装脚手架，直接 `npx create-react-app` 之类。

- **npm-run-all / concurrently**  
  多个脚本同时跑，比如前端 dev server + mock server。

- **http-server / serve**  
  快速起一个静态服务，打开构建好的 `dist`。

### 5.2 在线工具

- 正则测试：regex101 / regulex；
- JSON 格式化/校验：很多在线 JSON Viewer；
- Diff 工具：对比两份配置/文档。

这些不是必须，但用熟了之后会非常省时间。

---

## 6. 笔记与知识管理

虽然不是严格意义上的「前端工具」，但对工作流的影响很大。

2019–2020 期间，我主要是：

- 用 Markdown 记笔记（VS Code + 插件或专门的笔记软件）；
- 按「项目 / 主题」组织；
- 把常用命令、复杂配置、踩坑记录下来。

习惯写这些东西最大的好处是：

> 同一个坑，通常只会踩第二次，而不会第三次。

因为第二次你大概率还能翻到之前那篇记录。

---

## 7. 总结：工具为工作流服务，而不是反过来

这两年的一个明显感受是：

- 工具永远在变（新的编辑器、新的脚手架、新的框架）；
- 真正有价值的是那些**「让自己写代码更顺手的习惯」**。

比如：

- 保存即格式化 + 自动修复 lint；
- 新项目有固定的目录结构和约定；
- 提交信息有统一规则； 
- 出问题时，第一时间打开 DevTools / 日志系统，而不是盲猜。

这些东西一旦固化下来，未来换任何技术栈、任何新工具，你都会更快适应。

这篇算是给 2019–2020 的自己留一个快照，未来如果再大改一遍工作流，可以回过头来对比一下：哪些习惯真的留了下来，哪些只是某个阶段的「偏好」。

