+++
title = "菜单、按钮、接口：一套权限系统怎么落地到前端"
date = 2021-04-02
draft = false
categories = ["中后台架构"]
tags = ["权限系统", "RBAC", "前端权限"]
description = "从「角色-权限-资源」模型出发，讲一讲权限从后端到前端路由、按钮显隐、接口鉴权的整体设计思路。"
+++

中后台项目到了一定体量之后，绕不开一个问题：**权限怎么做？**

- 不同角色看到的菜单不一样；  
- 同一个页面里，有人能点「导出」、有人只能看；  
- 接口层还要做真正的权限校验。

这篇文章从实践角度，串一下我现在常用的一套设计：从「角色-权限-资源」模型，到前端路由、按钮显隐，再到接口鉴权。

---

## 1. 模型：角色、权限点、资源

先把概念讲清楚：

- **用户（User）**：具体的人；  
- **角色（Role）**：一类权限集合，例如「运营」、「风控」、「管理员」；  
- **权限点（Permission）**：最小粒度的能力，比如「订单列表查看」「订单导出」；  
- **资源（Resource）**：被操作的对象，比如菜单、按钮、接口。  

关系一般是：

```text
User → 多个 Role → 多个 Permission → 作用在不同 Resource 上
```

前端最关心的其实只有一件事：

> 登录之后，我拿到的「权限点列表」是什么？

因为**菜单、按钮显隐、前端路由保护**，本质上都是围绕这串权限点做判断。

---

## 2. 权限点命名：模块 + 资源 + 动作

命名如果乱，后面就会非常难维护。实践中我会用统一格式：

```text
<模块>:<资源>:<动作>
```

例如：

- `order:list:view` —— 订单列表查看；  
- `order:list:export` —— 订单列表导出；  
- `order:detail:update` —— 订单详情修改；  
- `user:manage:create` —— 用户管理新增。  

在数据库或配置里，用一个简单的结构描述：

```ts
interface Permission {
  code: string;      // order:list:view
  name: string;      // 订单列表-查看
  description?: string;
}
```

前端拿到的就是一串 `code`：

```json
["order:list:view", "order:list:export", "order:detail:update"]
```

---

## 3. 菜单与路由：根据权限生成

### 3.1 菜单数据从后端来

让后端维护一份菜单树：

```ts
interface MenuItem {
  id: string;
  parentId?: string;
  title: string;
  path: string;
  icon?: string;
  // 访问该菜单需要的权限点（可以是单个或多个）
  permission?: string;
  children?: MenuItem[];
}
```

登录成功后，前端请求：

```http
GET /api/auth/profile

// 响应示例
{
  "user": { "id": "u1", "name": "张三" },
  "menus": [/* 菜单树 */],
  "permissions": ["order:list:view", "order:list:export"]
}
```

前端：

1. 用 `menus` 渲染侧边栏；  
2. 用 `permissions` 初始化权限 Store（Vuex / Pinia 等）；  
3. 根据 `menus` 动态生成路由表。

### 3.2 动态路由生成示意

```ts
function generateRoutes(menus: MenuItem[]): RouteRecordRaw[] {
  const routes: RouteRecordRaw[] = [];

  const travel = (items: MenuItem[]) => {
    items.forEach(item => {
      const route: RouteRecordRaw = {
        path: item.path,
        name: item.id,
        component: lazyLoadView(item.path),
        meta: {
          title: item.title,
          permission: item.permission,
        },
      };
      routes.push(route);
      if (item.children) travel(item.children);
    });
  };

  travel(menus);
  return routes;
}
```

之后在路由守卫里，结合 `meta.permission` 做一次前端校验（有权限再放行）。

---

## 4. 按钮显隐：指令 + 工具函数

### 4.1 指令控制展示

以 Vue 为例，封一个 `v-permission` 指令：

```ts
// directives/permission.ts
import type { DirectiveBinding } from 'vue';
import { useUserStore } from '@/store/user';

function hasPermission(code: string) {
  const store = useUserStore();
  return store.permissions.includes(code);
}

export default {
  mounted(el: HTMLElement, binding: DirectiveBinding<string>) {
    const code = binding.value;
    if (code && !hasPermission(code)) {
      el.parentNode && el.parentNode.removeChild(el);
    }
  },
};
```

使用：

```vue
<el-button v-permission="'order:list:export'">
  导出
</el-button>
```

没有 `order:list:export` 权限的用户，根本看不到这个按钮。

### 4.2 逻辑层也要兜一层

展示控制是一层，逻辑上也要做好防御：

```ts
import { hasPermission } from '@/utils/permission';

async function handleExport() {
  if (!hasPermission('order:list:export')) {
    ElMessage.error('您没有导出权限');
    return;
  }
  // 真正导出逻辑
}
```

双保险：既避免误操作，也兼容某些「通过接口返回权限」后再动态渲染的场景。

---

## 5. 接口鉴权：真正的「硬防线」

前端的权限控制更多是体验与引导，真正的安全一定在接口层。

后端配置（简化示意）：

```ts
// 后端接口与权限点绑定
POST /api/order/export  →  order:list:export
```

请求到达服务端时：

1. 从 Token / 会话中解析出用户 ID、角色、权限点列表；  
2. 判断是否包含接口所需权限点；  
3. 不满足则返回统一错误码，如 `ACCESS_DENIED`。

前端统一处理：

```ts
if (resp.code === 'ACCESS_DENIED') {
  ElMessage.error('权限不足，如有疑问请联系管理员');
}
```

这样即使有人用抓包工具构造请求，也会被拦在服务端。

---

## 6. 一个比较完整的链路

把上面几块串起来，会是这样一条链路：

1. 用户登录 → 服务端签发 Token，并返回菜单树 + 权限点列表；  
2. 前端：  
   - 存储 Token；  
   - 初始化 `userStore.permissions`；  
   - 根据菜单生成路由；  
3. 页面渲染时：  
   - 菜单由后端数据 + 前端过滤生成；  
   - 路由守卫根据 `meta.permission` 和 `permissions` 决定是否放行；  
   - 按钮通过 `v-permission` 控制显隐；  
4. 触发接口调用时：  
   - 前端可以先用 `hasPermission` 做一次前置校验；  
   - 服务端再做最终鉴权。  

一句话总结：**菜单、按钮和接口三层权限要互相「说得上话」。**

---

## 7. 踩坑与经验

1. **权限点颗粒度不要太细**  
   - 每一个按钮一个权限点看起来很安全，但维护起来会疯；  
   - 建议一个功能能力对应一个权限点，比如「导出」、「审核」、「删除」。

2. **菜单权限和功能权限要区分**  
   - 有人可以「看到菜单但只能看数据」，有人既能看又能改；  
   - 菜单本身可以一个 `menu:view` 权限，操作功能独立权限点。

3. **避免在前端硬编码权限点字符串到处飞**  
   - 建议集中定义枚举，例如：

   ```ts
   export const PERM = {
     ORDER_LIST_VIEW: 'order:list:view',
     ORDER_LIST_EXPORT: 'order:list:export',
   } as const;
   ```

   - 这样 refactor 和查找都容易很多。

4. **权限变更的实时性要有预期**  
   - 一般来说，角色权限调整后，生效时间可以设计成「下次登录」；  
   - 如果要求高实时性，可以结合 WebSocket 或强制 Token 失效策略。

---

## 8. 小结

一套权限系统要落地到前端，核心是三件事：

1. 定义好统一的「权限点」编码；  
2. 菜单 & 路由 & 按钮都围绕这套编码做展示控制；  
3. 接口层做最终鉴权，前端再做一些体验上的配合。  

只要模型清晰、命名统一，后续再加页面、加按钮、加接口，都会变成一件「套路化」的事情，而不是每次都从头发明一套新的权限规则。

