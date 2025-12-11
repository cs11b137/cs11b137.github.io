+++
title = "写几个小脚本帮自己省时间：前端 CLI / 脚手架实践"
date = 2024-09-28
draft = false
categories = ["工程化"]
tags = ["CLI", "脚手架", "自动化", "Node.js"]
description = "分享几个日常使用的小 CLI / 脚手架：自动生成页面、接口封装和业务模块，让那些重复到麻木的工作变成一条命令。"
+++

有段时间我发现自己经常在做几件很机械的事：

- 新建一个页面：建目录、复制模板、改标题、改路由  
- 加一个接口：建 service 文件、写类型、写请求封装  
- 搭一个模块：列表页 + 详情页 + 编辑页，一遍遍重复

于是索性写了一组小 CLI 工具，把这些动作变成命令行：

> `npx fe-tools create-page`  
> `npx fe-tools create-api`  
> `npx fe-tools create-module`

这篇文章就简单记录一下这几个脚本的思路。

---

## 1. `create-page`：新页面一条命令

很多中台项目新增页面的套路都差不多：

1. 在 `views` 下建目录  
2. 填一个基础骨架组件  
3. 在路由里挂载  
4. 有的还要加到菜单配置里

用 Node.js 写了一个 `create-page` 命令：

```bash
npx fe-tools create-page
```

命令执行后，做几件事：

1. 命令行交互询问：  
   - 页面英文名（路由 path）  
   - 页面中文标题  
   - 页面类型：列表 / 表单 / 详情 / 组合

2. 基于模板生成对应文件，例如：

```vue
<template>
  <PageWrapper title="投资人概览">
    <!-- TODO: 填充页面内容 -->
  </PageWrapper>
</template>

<script setup lang="ts">
// 预留接口调用和状态管理位置
</script>
```

3. 自动往某个路由模块里追加配置

这样新页面出现的那一刻就已经可以访问，只需要填内容。

---

## 2. `create-api`：接口封装生成器

写接口封装时经常要做：

- 定义请求参数类型  
- 定义响应数据类型  
- 封装 `request` 函数

这些都是套路，于是又写了一个 `create-api`：

```bash
npx fe-tools create-api
```

约定输入可以是一个简单 JSON：

```jsonc
{
  "name": "getInvestorList",
  "url": "/api/investor/list",
  "method": "POST",
  "request": {
    "page": "number",
    "pageSize": "number",
    "keyword?": "string"
  },
  "response": {
    "total": "number",
    "list": "Investor[]"
  }
}
```

脚本生成的代码大致是：

```ts
export interface GetInvestorListReq {
  page: number
  pageSize: number
  keyword?: string
}

export interface Investor {
  // TODO: 补全字段
}

export interface GetInvestorListRes {
  total: number
  list: Investor[]
}

export function getInvestorList(params: GetInvestorListReq) {
  return request<GetInvestorListRes>({
    url: '/api/investor/list',
    method: 'POST',
    data: params
  })
}
```

后续只需要补充 `Investor` 的字段即可。

---

## 3. `create-module`：常规 CRUD 模块一键生成

很多中台模块的形态都是：

- 列表 + 搜索  
- 新建 / 编辑弹窗  
- 行操作（启用/禁用等）

所以又抽了一层「模块脚手架」：

```bash
npx fe-tools create-module
```

交互时会问：

- 模块英文名：`risk-rule`  
- 模块中文名：`风控规则`  
- 需要哪些能力：列表 / 表单 / 详情 / 导出 等

生成内容包括：

- 列表页 + 弹窗组件骨架  
- 对应路由配置  
- 初始 API 文件  
- 列定义、搜索表单配置等模板

这些模板代码并不追求极致抽象，核心原则是：

> 生成出来的东西要「看得懂、愿意改」，而不是一堆黑魔法。

---

## 4. 技术实现：CLI 的几个常用积木

几个常用库：

- `commander`：命令解析  
- `inquirer`：命令行交互  
- 模板引擎（ejs/handlebars）或者简单模板字符串  
- `fs` / `path`：读写文件

一个极简例子：

```ts
import { Command } from 'commander'
import inquirer from 'inquirer'
import fs from 'fs'
import path from 'path'

const program = new Command()

program
  .command('create-page')
  .action(async () => {
    const answers = await inquirer.prompt([
      { name: 'name', message: '页面英文名（用于路径）' },
      { name: 'title', message: '页面中文标题' }
    ])

    const dir = path.join(process.cwd(), 'src/views', answers.name)
    fs.mkdirSync(dir, { recursive: true })

    const content = `<template>
  <PageWrapper title="${answers.title}">
  </PageWrapper>
</template>

<script setup lang="ts">
</script>
`
    fs.writeFileSync(path.join(dir, 'index.vue'), content, 'utf-8')
    console.log('页面已生成：', dir)
  })

program.parse(process.argv)
```

再逐步演化出模板系统和配置。

---

## 5. 使用体验与小结

这些 CLI 带来的直接收益：

1. **减少重复劳动**  
   - 新页面/新模块的「机械工作量」明显下降  
   - 人更多在想业务结构和交互

2. **统一工程结构**  
   - 新人跟着模板写，很自然地保持一致结构  
   - 换项目迁移成本变低

3. **方便与 AI 结合**  
   - CLI 负责生成骨架  
   - AI 可以帮忙填充接口调用、状态管理、甚至部分页面逻辑

写 CLI 本身也没什么神秘的，本质就是：

> 把自己已经做过很多次、模式比较稳定的事情，封装成一套脚本。

如果你也想写，可以先从一个问题开始：

> 「这三个月里，哪件事我重复做得最多？」

从那里切入，做一个小脚本，很快就能尝到甜头。
