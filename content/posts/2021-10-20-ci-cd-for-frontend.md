+++
title = "给前端项目接上 CI/CD：自动构建与预发布环境"
date = 2021-10-20
draft = false
categories = ["工程化"]
tags = ["CI/CD", "自动化部署", "前端发布流程"]
description = "梳理一次给前端项目搭建 CI/CD 流水线的过程：从代码提交，到自动构建、部署预发布环境，再到可回滚的上线流程。"
+++

以前做前端上线，经常是这样的：

- 本地 `npm run build` 打包；  
- 用 FTP 或手动拷贝把 dist 丢到服务器；  
- 出了问题再手动覆盖回去。  

这种方式有几个明显缺点：

- 完全依赖人的操作习惯，出错难追踪；  
- 没有固定的「预发布环境」，测试体验不好；  
- 回滚麻烦，基本靠「手快」。  

这篇文章记录的是：**一次给前端项目接上 CI/CD 的实际过程**，目标是：

> 提交代码 → 自动构建 → 部署到预发布 → 人工验证 → 一键上线 & 可快速回滚。

---

## 1. 流程目标先画出来

先不管用什么平台（GitHub Actions、GitLab CI、Jenkins…），流程大致分为几步：

1. **CI 阶段（持续集成）**
   - 安装依赖；
   - 跑 lint / 单测；
   - 打包构建。

2. **CD 阶段（持续交付/部署）**
   - 构建产物上传到制品库或对象存储；  
   - 自动部署到预发布环境；  
   - 人工验证无问题后，再部署到生产；  
   - 所有版本有记录，可一键回滚。

把这张「大图」先画给团队看，大家对齐预期后，再来谈具体实现。

---

## 2. 以 GitHub Actions 为例写一个最小流水线

### 2.1 基础 CI：安装 + 构建

`.github/workflows/ci.yml`：

```yaml
name: Frontend CI

on:
  pull_request:
    branches: [ main, develop ]
  push:
    branches: [ develop ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Use Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build

      - name: Archive production artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist
```

这样每次 PR / 推送到 develop，都能自动帮你检查：

- 能不能正常安装依赖；  
- lint 是否通过；  
- 构建是否成功。

这一步就已经能避免不少「构建失败还被合到主干」的问题。

---

## 3. 预发布环境部署

接下来是 CD 部分。以「静态站部署到 Nginx + 对象存储」为例，我们可以：

1. 在 CI 构建产物后，上传到一个对象存储/制品库；  
2. 触发一段部署脚本把对应版本发布到「预发布环境」。

举个简化示例（仍然用 GitHub Actions）：

```yaml
# 在 CI 成功的基础上新增一个 job
deploy_staging:
  needs: build
  runs-on: ubuntu-latest
  if: github.ref == 'refs/heads/develop'

  steps:
    - name: Download artifact
      uses: actions/download-artifact@v3
      with:
        name: dist
        path: dist

    - name: Deploy to staging server
      uses: easingthemes/ssh-deploy@v2
      with:
        ssh-private-key: ${{ secrets.SSH_KEY }}
        remote-user: deploy
        server-ip: ${{ secrets.STAGING_HOST }}
        remote-path: /var/www/your-app-staging
        local-path: dist
```

这段会把 `dist` 同步到预发布服务器指定目录。

项目内部约定：

- 访问 `https://staging.xxx.com` 就是预发布；  
- 只有通过了预发布验证，才允许合并到主干或触发生产部署。

---

## 4. 生产环境上线与回滚

生产环境的部署要更谨慎一些，可以有几种触发方式：

1. 手动在 CI 平台点「Deploy to Production」；  
2. 打 Tag 时触发；  
3. 推送到 `main` 分支时触发，但建议加人工确认步骤。

示例（手动触发 + 简单回滚能力）：

```yaml
name: Deploy Production

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version or branch to deploy'
        required: true
        default: 'main'

jobs:
  deploy_prod:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
        with:
          ref: ${{ github.event.inputs.version }}

      - name: Use Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy to production
        uses: easingthemes/ssh-deploy@v2
        with:
          ssh-private-key: ${{ secrets.SSH_KEY }}
          remote-user: deploy
          server-ip: ${{ secrets.PROD_HOST }}
          remote-path: /var/www/your-app
          local-path: dist
```

回滚的时候，只需要：

- 再次触发同一个 workflow；  
- `version` 输入改成上一个 Tag 或 commit hash；  
- CI 会用那一版代码重新构建并覆盖部署。

如果你使用的是 S3/OSS + CDN 的方式，也可以：

- 按版本号上传到不同前缀；  
- 用一个 `current` 的软链接/配置文件指向当前版本；  
- 回滚时只需要切回旧的前缀。

---

## 5. 在流水线中加入质量门禁

有了 CI/CD 之后，还可以顺手做几件「质量相关」的小事：

1. **Lint 不通过禁止合并到主干**；  
2. 有单测的话，把 `npm test` 放进 CI；  
3. 接入简单的构建产物体积检测（防止某次引入包导致 bundle 暴涨）；  
4. 构建失败时通过机器人推送到群里，提醒相关同学处理。

这些都属于「顺带就能做」的事情，对整体体验提升很明显。

---

## 6. 落地过程中的小问题

1. **一开始脚本写得过于复杂**  
   - 想在第一次就把灰度、蓝绿发布都上齐，结果自己都搞晕了；  
   - 后来改成：先最小可用流程跑通，再慢慢加花活。

2. **环境变量管理混乱**  
   - CI 和服务器上的环境变量不一致，导致「本地 OK、CI 失败」；  
   - 把所有环境变量（后端域名、静态资源域名等）都收敛到 `.env.*` 和 CI 平台 Secrets 管理。

3. **日志与监控缺位**  
   - 早期部署失败只能看脚本输出，很难定位；  
   - 后来尽量让脚本有清晰日志，以及对接一些简单监控（如 Nginx 状态、前端监控上报）。

---

## 7. 小结

给前端项目接上 CI/CD，本质是在做三件事：

1. **把「能自动化的步骤全部自动化」**  
   - 安装依赖、lint、构建、上传、部署。

2. **显式区分「开发 / 预发布 / 生产」**  
   - 每个环境有清晰的入口和配置；  
   - 预发布成为上线前的必经之路。

3. **让上线和回滚都变成「标准动作」**  
   - 用脚本和流水线固化流程；  
   - 减少对人的记忆和手感依赖。  

做完这些之后，你会明显感觉到：

- 发版节奏更可控；  
- 出问题时更镇定，因为你知道「怎么回滚」；  
- 团队对「上线」这件事的恐惧感会小很多。

