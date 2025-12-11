+++
title = "一次代码，多端运行：H5 + 小程序的复用实践"
date = 2021-01-23
draft = false
categories = ["多端开发"]
tags = ["代码复用", "小程序", "H5", "多端架构"]
description = "结合实际项目，聊聊如何抽象业务核心、隔离端能力差异，让一套代码尽可能在 H5 和小程序之间复用。"
+++

多端已经是老话题了：

- 产品想要 **H5 + 小程序 + 后面可能再来个 App**；  
- 但人手并不会随平台数量线性增加。  

所以问题就变成了：

> **能不能尽量做到：一套业务逻辑，多端复用，只在各端做少量适配？**

这篇文章从一个实际项目出发，聊聊我是怎么拆分层次、抽象业务核心、隔离端能力差异，让 H5 和小程序在同一套代码下运行得比较舒服。

---

## 1. 能复用和不能复用的先分清

先别谈框架，先想清楚几件事：

**通常可以复用的：**

- 业务模型：订单、用户、商品这些核心数据结构；  
- 领域逻辑：状态流转、金额计算、表单校验等纯逻辑；  
- 接口层：请求参数组装、响应数据提取、错误码处理；  
- 公共工具：日期、金额、校验、格式化等。  

**通常不能直接复用的：**

- UI 组件和布局（DOM vs 小程序组件）；  
- 路由与页面生命周期；  
- 存储层（`localStorage` vs `wx.setStorage` 等）；  
- 原生能力（扫码、相册、位置等）。  

所以设计时的目标是：

> **把「业务核心」从这些平台差异里剥离出来，用一层「适配器」去对接不同端能力。**

---

## 2. 目录与分层：core + adapters

我们当时的多端项目采用的是类似 Monorepo 的结构：

```bash
project-root
├── packages
│   ├── core          # 端无关的业务核心（可被多端复用）
│   ├── h5            # H5 端工程（Vue/React 等）
│   └── miniapp       # 小程序端工程（uni-app / Taro / 原生）
└── package.json
```

### 2.1 `core` 里放什么？

核心模块大致有：

- `models/` —— 订单、用户等类型定义和基础逻辑；  
- `services/` —— 业务服务，比如下单、取消订单、查询列表；  
- `apis/` —— 针对后端的 API 封装（不依赖具体请求实现）；  
- `utils/` —— 工具方法。  

关键点是：**`core` 不能直接用浏览器 API 或小程序 API**，一旦用了就失去可复用性。

### 2.2 请求与存储用「接口」抽象

举个例子，`core/apis/httpClient` 不直接 new Axios 或调用 `wx.request`：

```ts
// packages/core/apis/http.ts
export interface HttpClient {
  get<T>(url: string, config?: any): Promise<T>;
  post<T>(url: string, data?: any, config?: any): Promise<T>;
}

let client: HttpClient;

export function setHttpClient(c: HttpClient) {
  client = c;
}

export function getHttpClient() {
  if (!client) {
    throw new Error('HttpClient not set');
  }
  return client;
}
```

在 H5 端：

```ts
// packages/h5/src/bootstrap.ts
import axios from 'axios';
import { setHttpClient } from '@project/core/apis/http';

setHttpClient({
  get: (url, config) => axios.get(url, config).then(res => res.data),
  post: (url, data, config) => axios.post(url, data, config).then(res => res.data),
});
```

在小程序端：

```ts
// packages/miniapp/src/bootstrap.ts
import { setHttpClient } from '@project/core/apis/http';

setHttpClient({
  get(url, config) {
    return new Promise((resolve, reject) => {
      wx.request({
        url,
        method: 'GET',
        data: config?.params,
        success(res) { resolve(res.data); },
        fail(err) { reject(err); },
      });
    });
  },
  post(url, data) {
    return new Promise((resolve, reject) => {
      wx.request({
        url,
        method: 'POST',
        data,
        success(res) { resolve(res.data); },
        fail(err) { reject(err); },
      });
    });
  },
});
```

业务代码只关心：

```ts
import { getHttpClient } from './http';

export async function fetchOrderList(params) {
  return getHttpClient().get('/order/list', { params });
}
```

这样就实现了真正的「多端同一份业务逻辑」。

---

## 3. 业务服务如何做「薄 UI，厚 Service」

在 `core/services` 里，我习惯把业务串起来：

```ts
// packages/core/services/orderService.ts
import { fetchOrderList } from '../apis/order';
import type { Order, OrderQueryParams } from '../models/order';

export async function queryOrders(params: OrderQueryParams): Promise<Order[]> {
  const res = await fetchOrderList(params);
  // 这里可以做一些统一的转换、排序、过滤
  return res.list || [];
}

export function canCancel(order: Order): boolean {
  return order.status === 'CREATED' || order.status === 'PENDING';
}

export function getOrderStatusText(order: Order): string {
  // ...
  return '已支付';
}
```

在 H5 端和小程序端页面里直接用这些 service：

```ts
// H5 / 小程序 里都是类似的调用
const list = await queryOrders({ page: 1, pageSize: 10 });
```

UI 差异就留在端内处理，比如表格渲染样式、列表滚动加载等。

---

## 4. 多端差异的几个典型适配点

### 4.1 路由与跳转

- H5：`router.push({ name: 'OrderDetail', params: { id } })`  
- 小程序：`wx.navigateTo({ url: '/pages/order-detail/index?id=' + id })`  

做法是也抽象成一个 `Navigator` 接口：

```ts
// core/router/navigator.ts
export interface Navigator {
  goOrderDetail(id: string): void;
  goOrderList(): void;
}

let navigator: Navigator;

export function setNavigator(n: Navigator) {
  navigator = n;
}

export function useNavigator() {
  if (!navigator) throw new Error('Navigator not set');
  return navigator;
}
```

H5 和小程序各自实现一套 `Navigator`，业务逻辑只关心 `useNavigator().goOrderDetail(id)`。

### 4.2 环境信息 & 存储

- token、本地缓存、当前环境（dev/stage/prod）等等，也可以抽象成一组 Env / Storage 接口；  
- `core` 中只通过这些接口获取信息。

---

## 5. 踩过的坑和经验

1. **不要一开始就追求「所有代码都完全共用」**  
   - 一些强 UI 相关的东西（组件、页面布局）放在各端自己实现维度更合理；  
   - 共用优先放在：数据结构、业务规则、接口封装。

2. **记得为「以后要支持新端」预留空间**  
   - 不要在 `core` 里写 `if (isMiniapp) { ... }` 这种判断；  
   - 用「接口 + 端内适配」的方式更可扩展。

3. **类型定义越早统一越好**  
   - 特别是使用 TypeScript 时，共享 `models` 可以避免端内字段名乱飞；  
   - 同时也让多端改动时更有边界感。

---

## 6. 小结

多端复用的关键不是某个框架，而是：

1. **先把「和端无关的业务逻辑」揪出来；**  
2. 用一层稳定的接口把它们和「端能力」隔开；  
3. 在各端实现自己的适配层。  

当业务迭代变快、端数变多时，这套分层会真正体现价值：

- 新增一个端，多数情况下只需要实现一套适配；  
- 需求逻辑变化，多端只改 `core` 里的一份代码。  

这才是真正意义上的「一次代码，多端运行」。

