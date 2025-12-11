+++
title = "前后端协作改造记：从「口头协议」到 Mock 驱动开发"
date = 2021-08-30
draft = false
categories = ["工程协作"]
tags = ["接口规范", "Mock", "前后端协作"]
description = "记录一次把接口文档、Mock、联调流程规范化的过程，以及过程中踩过的坑。"
+++

很多项目的前后端协作，一开始都是这样：

> 「你先把页面搭着，接口我晚点给你。」  
> 「返回字段大概就这些，不行再改。」  

结果就是：

- 接口文档滞后甚至缺失，全靠口头约定；  
- 字段今天叫 `userName` 明天叫 `customerName`；  
- 前端经常等后端联调，后端也抱怨前端改来改去。  

这篇文章是一次真实改造的复盘：我们从这种「口头协议」状态，逐步过渡到：

> 有规范的接口文档 + 可用的 Mock 服务 + 基于契约的联调流程。

---

## 1. 先承认问题：我们缺的不是工具，而是「契约」

一开始我们也装过各种工具：

- Swagger / OpenAPI；  
- YApi / Rap2 之类的平台；  
- 各种 Mock 插件。  

但用一段时间就废掉了，根本原因是：**文档没人维护，接口变更不走流程**。

所以改造的第一步，不是换工具，而是统一几个规则：

1. **所有新接口必须先有文档，再开发**；  
2. 文档就是契约：前端后端都按契约来实现；  
3. 接口变更要么走版本，要么走兼容字段，不再「直接改」。

工具只是为了让这套流程变得没那么痛苦。

---

## 2. 选择和收敛接口文档工具

我们最终选的是：**OpenAPI 规格 + 一套接口管理平台（如 YApi 或内部平台）**：

- 提供可视化的接口列表、请求例子、响应示例；  
- 支持一键导出 JSON / SDK 生成；  
- 自带 Mock 功能。  

接口粒度大致是：

```yaml
paths:
  /api/order/list:
    post:
      summary: 订单列表
      parameters: []
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/OrderListRequest'
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderListResponse'
```

当后端定义好这些 schema 后，前端就可以基于文档做代码生成、Mock 数据、类型定义。

---

## 3. Mock 驱动开发：不用等后端就能联调

### 3.1 基本思路

1. 后端先在接口平台上定义好接口和示例数据；  
2. 平台生成对应的 Mock 地址；  
3. 前端在 `.env.mock` 环境下把接口域名切换到 Mock 服务；  
4. 页面开发和联调可以在 **后端代码还没写完之前就开始**。

### 3.2 前端环境配置

例如在 Vite 项目里：

```bash
# .env.mock
VITE_API_BASE_URL=https://mock-api.example.com
```

请求工具：

```ts
// http.ts
const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
});
```

需要真实联调时，只要切换到 `.env.dev` 即可。

---

## 4. 前端如何「信任」文档？——用类型和 SDK 来约束

当接口文档是 OpenAPI 时，可以用生成器把它变成 TS 类型和请求函数：

- 避免手写接口类型；  
- 字段改名时能在编辑器里暴露问题。

生成结果大概是：

```ts
// apiTypes.ts
export interface OrderListRequest {
  page: number;
  pageSize: number;
  status?: string;
}

export interface OrderListItem {
  orderId: string;
  customerName: string;
  amount: number;
}

export interface OrderListResponse {
  list: OrderListItem[];
  total: number;
}
```

和：

```ts
// apiClient.ts
export function fetchOrderList(params: OrderListRequest) {
  return http.post<OrderListResponse>('/api/order/list', params);
}
```

这样前端的调用方式就固定了：

```ts
const res = await fetchOrderList({ page: 1, pageSize: 20, status: 'PAID' });
```

一旦后端修改字段名/类型，生成脚本重新跑一遍，前端这边就会有类型错误提示。

---

## 5. 联调流程也需要「规范」

我们给联调流程加了几条简单规则：

1. **开发阶段优先连 Mock 环境**  
   - 页面基本交互完成后，再切换到 Dev 环境真实联调；  
   - 避免双方频繁「互等」。

2. **接口变更走兼容策略**  
   - 增加字段：向后兼容；  
   - 字段重命名：旧字段保留一段时间，同时新增新字段，并在文档里注明；  
   - 废弃接口：用新接口替代，并给出迁移指南。

3. **联调前对齐文档版本**  
   - 接口平台有版本号，每次大改之前 bump 一次；  
   - 前后端确认当前都是基于哪个版本来开发。

---

## 6. 过程中踩过的一些坑

1. **Mock 和真实数据差异太大**  
   - 一开始 Mock 数据写得太「理想」，导致线上真实数据一来就各种边界问题；  
   - 后来要求：Mock 数据要尽量贴近真实，比如「长字符串」「特殊字符」「缺失字段」。

2. **接口平台当「文档站」，不当「单一事实源」**  
   - 有些同学在 wiki/文档软件里又写了一套接口说明；  
   - 久而久之，大家不知道看哪一份。  
   - 后来统一约定：**以接口平台为唯一标准**，其他地方只放链接。

3. **忘了给 Mock 分环境**  
   - 有时候需要模拟错误/超时等情况；  
   - 把「正常场景」和「异常场景」分成两套 Mock 配置，可以通过请求参数/头信息控制。

---

## 7. 改造之后带来的变化

一段时间之后，能明显感受到：

- 新需求时，**讨论接口结构变成一个标准步骤**；  
- 前端开发不会再因为「接口还没好」而完全停滞；  
- 接口字段的变更不会悄悄进行，而是需要更新文档和版本；  
- 出现线上问题时，可以直接对照接口文档排查。

最关键的不是我们用了什么工具，而是整个团队都开始认同：

> **接口文档是「契约」，而不是「补充说明」。**

---

## 8. 小结

从「口头协议」到 Mock 驱动开发，可以概括为三件事：

1. 选一套统一的接口描述方式（OpenAPI 等），并认定它是单一事实源；  
2. 把 Mock 融入开发流程，让前后端都可以基于文档并行工作；  
3. 用类型和生成工具，尽量减少「文档和实现不一致」的问题。  

当接口变得有契约、有版本、有 Mock 后，前后端协作会从「互相埋怨」慢慢变成「一起打磨契约」，这就是工程化改造最有价值的地方。

