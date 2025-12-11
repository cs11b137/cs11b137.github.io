+++
title = "单页应用路由设计心得：守护好你的「返回键」"
date = 2020-02-15
draft = false
categories = ["前端架构"]
tags = ["路由", "SPA", "前端体验"]
description = "从实际项目出发讨论路由结构、面包屑、标签页和返回行为的设计和实践。"
+++

单页应用（SPA）带来了更流畅的交互体验，但也制造了一个经典问题：**「返回键」不好使了。**

- 用户点浏览器返回，结果直接退回登录页，心态爆炸；  
- 或者在一个中台系统里，从列表点到详情、从详情点到编辑、再返回时，筛选条件全没了，只剩一个「空空如也」。

这篇文章就结合我自己踩过的坑，聊聊：**如何在 SPA 里设计更「靠谱」的路由结构和返回行为。**

---

## 1. 路由不是简单的 URL 映射

很多项目的路由配置看起来像这样：

```js
const routes = [
  { path: '/list', component: List },
  { path: '/detail/:id', component: Detail },
];
```

能跑，但如果你问：

- 列表页的筛选条件放哪？
- 返回列表时要不要还原滚动位置？
- 详情页打开方式是「覆盖当前」还是「新标签页」？

这些问题路由本身没有回答。  
所以对我来说，路由不仅仅是 path → component 的映射，它还应该承载一些**页面行为约定**。

---

## 2. 基本路由结构：从「业务模块」出发

在中后台项目里，我现在更倾向按「业务模块」来规划路由：

```js
const routes = [
  {
    path: '/order',
    component: Layout,
    children: [
      { path: 'list', name: 'OrderList', component: OrderList },
      { path: 'detail/:id', name: 'OrderDetail', component: OrderDetail },
      { path: 'edit/:id', name: 'OrderEdit', component: OrderEdit },
    ],
  },
];
```

几个小习惯：

1. **路径短而语义化**：`/order/list` 比 `/orderListPage` 更统一。
2. **给路由起 name**：后面做编程式导航、标签页缓存都会用到。
3. **模块下的子路由聚在一起**：方便看清一个模块的整体结构。

---

## 3. 返回行为：列表页的「状态保留」

最常见的场景：

1. 在订单列表页选择了各种条件，翻到第 4 页；
2. 点进某个订单详情；
3. 返回时希望停留在「刚才那一页 + 筛选条件」。

如果不做额外处理，很多 SPA 会出现：

- 返回后列表重置成默认条件；
- 页码回到第一页；
- 滚动条也回到顶部。

### 3.1 把列表查询条件放进 URL

一个简单做法是：**把查询条件放进 query**：

```js
// /order/list?status=PAID&page=4&keyword=foo
{
  path: 'list',
  name: 'OrderList',
  component: OrderList,
}
```

在 `OrderList` 里：

```js
created() {
  const { status, page, keyword } = this.$route.query;
  this.searchForm = {
    status: status || 'ALL',
    keyword: keyword || '',
  };
  this.currentPage = Number(page) || 1;
  this.fetchData();
}

methods: {
  onSearch() {
    this.$router.push({
      name: 'OrderList',
      query: {
        ...this.searchForm,
        page: 1,
      },
    });
  },
  onPageChange(page) {
    this.$router.push({
      name: 'OrderList',
      query: {
        ...this.$route.query,
        page,
      },
    });
  },
}
```

这样：

- 从列表进入详情，再返回时，URL 仍然带着之前的查询条件和页码；
- `created` 里根据当前 URL 还原状态。

**好处：**

- 自带「可分享性」：复制 URL 发给别人就是同一个列表视图；
- 刷新页面不会丢状态。

---

### 3.2 保留滚动位置

有时列表很长，即使页码和筛选条件保留了，滚动条也重新在顶部，体验会打折扣。

Vue Router 可以通过 `scrollBehavior` 做滚动还原：

```js
const router = new VueRouter({
  mode: 'history',
  routes,
  scrollBehavior(to, from, savedPosition) {
    if (savedPosition) {
      return savedPosition;
    } else {
      return { x: 0, y: 0 };
    }
  },
});
```

- `savedPosition` 在浏览器前进/后退时会自动生成。
- 手动调用 `router.push` 不会触发 `savedPosition`，会滚动到顶部。

**实践上的经验：**  

- 列表页默认滚到顶部没问题；  
- 详情页 → 返回列表，通常会触发浏览器历史栈的回退，这时 `savedPosition` 会生效，还原到刚才的位置。

---

## 4. 面包屑与标签页：别和真实路由「打架」

中后台常见的两种导航元素：

- 面包屑（Breadcrumb）  
- 顶部/侧边的「多标签页」导航

如果这两者和路由没有关系，而是单纯「UI 自己管理一个数组」，就容易出问题：

- 刷新页面后面包屑全没了；
- 从地址栏直接访问某个详情页时，标签页里没有对应项；
- 动态路由 + 权限控制时更容易乱套。

我的做法是：

1. 在路由 meta 里配置面包屑信息：

```js
{
  path: 'detail/:id',
  name: 'OrderDetail',
  component: OrderDetail,
  meta: {
    title: '订单详情',
    breadcrumb: ['订单管理', '订单列表', '订单详情'],
  },
}
```

2. 标签页根据当前访问过的路由 name + params 动态生成：

```js
// 简化示意
const tabs = [
  { key: 'OrderList', title: '订单列表', to: { name: 'OrderList' } },
  { key: 'OrderDetail-123', title: '订单详情-123', to: { name: 'OrderDetail', params: { id: 123 } } },
];
```

只要 `tabs` 的 `to` 是真实可用的路由对象，就不会和路由系统割裂。

---

## 5. 模态详情 vs. 路由详情

有些项目会用「弹窗 + 抽屉」展示详情：

- 好处：不用跳转页面，用户可以快速来回看不同记录；  
- 坏处：刷新页面、分享 URL、浏览器返回行为都不太好处理。

**一个折中方案是：**

- 把详情仍然设计为一个独立路由：`/order/detail/:id`；
- 在列表页里，如果是内部点开的详情，可以使用抽屉展示，并通过路由表示当前状态；
- 刷新页面或外部访问该路由时，则正常渲染「详情页布局」。

例如：

```js
// 列表页点击：
this.$router.push({
  name: 'OrderList',
  query: {
    ...this.$route.query,
    detailId: row.id,
  },
});
```

模板：

```vue
<Drawer v-if="$route.query.detailId">
  <OrderDetail :id="$route.query.detailId" />
</Drawer>
```

这样：

- URL 里有当前查看的详情 id；
- 刷新不会丢；
- 返回也能正确还原。

---

## 6. 小结：几个我现在会默认考虑的问题

每当我在设计一个中后台模块的路由时，我会先问自己：

1. 这个模块的**主列表路由**是什么？它的查询条件和分页怎么表示在 URL 里？  
2. 详情 / 编辑 / 创建分别用什么路由？是否需要和列表共用布局？  
3. 返回列表时，是否需要保留筛选条件、页码和滚动位置？  
4. 面包屑和标签页是否都能够**从路由推导出来**？  
5. 是否有需要用「路由 + 抽屉/弹窗」混合展示的场景？

SPA 带来的很多坑，本质上是因为：

> **我们只把路由当成「跳转路径」，而没有把它当成「页面状态的表达」。**

把这些问题在设计阶段想清楚，后面维护时会轻松很多，用户也不会再被「失忆的返回键」折腾。

