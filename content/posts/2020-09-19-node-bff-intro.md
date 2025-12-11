+++
title = "给前端加一层 BFF：Node 中间层的第一步"
date = 2020-09-19
draft = false
categories = ["全栈 / BFF"]
tags = ["Node.js", "BFF", "接口聚合"]
description = "用一个小例子讲清楚 BFF 的价值、路由聚合以及简单缓存策略。"
+++

写前端写久了，你大概率遇到过这些场景：

- 同一个页面要调 3～4 个后端服务，字段名都不一样。
- 为了兼容多个客户端（H5、小程序、App），接口被搞得非常「通用」，前端要做一堆字段转换。
- 某些数据其实可以缓存，但后端坚持不做，只能前端自己「临时记一下」。

这个时候，很多团队会在前后端之间加一层 **BFF（Backend For Frontend）**。

这篇文章以一个小例子来说明：什么是 BFF、为什么要用、第一步怎么做。

---

## 1. 什么是 BFF？

一句话解释：

> **BFF 是介于前端和后端业务服务之间的一层「适配器」，为特定前端提供定制化接口。**

传统架构：

```text
浏览器 / 小程序  --->  各个后端服务
```

引入 BFF 后：

```text
浏览器 / 小程序  --->  BFF  --->  各个后端服务
```

BFF 做的事情包括但不限于：

- 聚合多个后端接口；
- 做一些轻量的业务组合逻辑；
- 做权限校验、请求限流；
- 做缓存（比如热门数据）；
- 为不同客户端做「视图模型」适配。

常用技术栈：Node.js + Koa/Express/Nest 等。

---

## 2. 一个具体例子：用户资产概览

假设我们有一个前端页面「用户资产总览」，需要展示：

- 用户基本信息（来自 `user-service`）；
- 用户账户列表（来自 `account-service`）；
- 用户最近 5 笔交易（来自 `trade-service`）。

### 2.1 没有 BFF 时的做法

前端直接请求多个后端服务：

```ts
GET /user-service/api/user/profile
GET /account-service/api/account/list
GET /trade-service/api/trade/latest?limit=5
```

前端需要：

- 处理不同的返回格式；
- 做错误合并处理（任何一个接口失败都要考虑）；
- 处理多次往返延迟。

---

### 2.2 有 BFF 后的做法

前端只请求 BFF：

```ts
GET /bff/api/dashboard/overview
```

BFF 内部：

```js
// 示例用 Express
const express = require('express');
const axios = require('axios');

const app = express();

app.get('/bff/api/dashboard/overview', async (req, res) => {
  const userId = req.query.userId;

  try {
    const [userRes, accountRes, tradeRes] = await Promise.all([
      axios.get('http://user-service/api/user/profile', { params: { userId } }),
      axios.get('http://account-service/api/account/list', { params: { userId } }),
      axios.get('http://trade-service/api/trade/latest', { params: { userId, limit: 5 } }),
    ]);

    res.json({
      code: 0,
      data: {
        user: userRes.data.data,
        accounts: accountRes.data.data,
        trades: tradeRes.data.data,
      },
    });
  } catch (err) {
    // 统一错误处理
    console.error(err);
    res.status(500).json({
      code: 500,
      message: 'INTERNAL_ERROR',
    });
  }
});

app.listen(3000, () => {
  console.log('BFF listening on port 3000');
});
```

前端拿到的数据结构清晰统一：

```json
{
  "code": 0,
  "data": {
    "user": { /* ... */ },
    "accounts": [ /* ... */ ],
    "trades": [ /* ... */ ]
  }
}
```

**好处：**

- 页面加载时只需要一次 HTTP 往返；
- 接口字段由 BFF 统一适配前端需求；
- 错误处理逻辑集中在 BFF。

---

## 3. BFF 的典型职责

### 3.1 字段转换与视图模型

后端返回的数据往往更偏「领域模型」，前端则需要更贴合 UI 的「视图模型」。

示例：

```js
// user-service 返回
{
  "userId": "u_123",
  "userName": "张三",
  "gmtCreate": 1599800000000
}

// BFF 转成视图模型
{
  "id": "u_123",
  "name": "张三",
  "registerTime": "2020-09-11 10:00:00"
}
```

这样前端可以用更统一的字段名，不必到处写格式化逻辑。

---

### 3.2 权限与鉴权

用户信息、角色信息往往来自 Token / SSO 系统。

- BFF 可以在进入路由前统一解析 Token；
- 把 `userId`、`roles`、`permissions` 注入到请求上下文；
- 再根据这些信息去调用下游服务。

伪代码：

```js
app.use(authMiddleware);

function authMiddleware(req, res, next) {
  const token = req.headers['authorization'];
  // 验证 token...
  req.user = { id: 'u_123', roles: ['ADMIN'] };
  next();
}
```

这样每个业务路由里就可以访问 `req.user`，不用每次重新解析。

---

### 3.3 简单缓存

对于某些读多写少、变化不频繁的数据，可以在 BFF 层做些缓存：

```js
const NodeCache = require('node-cache');
const cache = new NodeCache({ stdTTL: 60 }); // 60s 缓存

app.get('/bff/api/configs', async (req, res) => {
  const cacheKey = 'config_all';
  const cached = cache.get(cacheKey);
  if (cached) {
    return res.json({ code: 0, data: cached, fromCache: true });
  }

  const remote = await axios.get('http://config-service/api/configs');
  cache.set(cacheKey, remote.data.data);
  res.json({ code: 0, data: remote.data.data, fromCache: false });
});
```

**注意：**

- BFF 缓存适合小数据量、短期缓存；
- 不要把所有东西都往缓存里塞，容易失控。

---

## 4. BFF 的一些实践经验

### 4.1 保持「薄而专一」

BFF 不应该变成另一个「巨型后端系统」，而应：

- 做轻量的聚合、转换；
- 尽量不承载复杂的业务规则（那些应该在后端领域服务里）。

一句话：**BFF 更偏「视图层服务」，不是业务大脑。**

---

### 4.2 与前端紧密演进

BFF 的改动节奏往往要跟前端保持一致：

- 新页面要上线 → 先在 BFF 加好接口 → 再接前端；
- 接口字段变更 → 优先在 BFF 做兼容 → 再慢慢调整前端。

有些团队甚至会把 BFF 代码和前端放在同一个仓库，用 Monorepo 管理，这样可以：

- 在同一个 PR 里改前端 + BFF；
- 保证版本一致性。

---

### 4.3 日志与监控

BFF 是前端的「最近一跳服务」，日志和监控非常关键：

- 打印清晰的访问日志（谁、在哪个接口、耗时）；
- 对下游服务超时、错误情况做统计；
- 监控 BFF 自身错误率和性能。

例如简单的日志中间件：

```js
app.use(async (req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`[${req.method}] ${req.originalUrl} ${res.statusCode} - ${duration}ms`);
  });
  next();
});
```

---

## 5. 什么时候适合上 BFF？

个人经验是：当你同时满足以下 2～3 条时，就可以认真考虑：

1. 有多个前端客户端（Web、App、小程序等）。
2. 前端需要从多个后端服务取数据，并且组合逻辑越来越复杂。
3. 经常需要为某个前端做特定改造，但又不方便动后端服务。 
4. 前端团队有一定的 Node 能力，能维护一层中间服务。

如果只是一个小项目、一个后端、几个简单接口，BFF 反而会增加复杂度，没有必要。

---

## 6. 小结

BFF 的核心价值可以概括成三点：

1. **为前端屏蔽后端复杂性**：聚合多个服务、统一字段、做简单逻辑。
2. **为不同前端提供定制接口**：H5、App、小程序可以有不同视图模型。 
3. **在前端和后端之间增加一层「缓冲区」**：方便做灰度、缓存、限流、埋点等。

如果你已经在维护一个稍微复杂点的前端项目，建议可以从一个非常小的 BFF 开始：

- 先挑一个页面，把原本多个接口合成一个；
- 再慢慢把字段转换、鉴权、缓存等能力补上；
- 等团队习惯了这套模式，再考虑更系统化的架构。

BFF 不一定要一上来就「高大上」，从一个简单的 Node 应用开始就很好。

