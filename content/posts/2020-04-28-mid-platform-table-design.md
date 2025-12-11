+++
title = "中后台「表格 + 查询」通用化设计实践"
date = 2020-04-28
draft = false
categories = ["中后台架构"]
tags = ["中台", "表格组件", "查询表单"]
description = "抽象常见查询区、表格、分页、批量操作组件的一套通用方案。"
+++

中后台系统有一个经典页面模样：

> 上面一块查询条件，下面一张表格，再加一个分页条。

做过几套之后你会发现：字段变了、接口变了、样式改改，**本质都差不多**。与其每个项目都重写，不如抽象出一套可复用的「查询 + 表格」方案。

这篇文章记录我在中后台项目里总结的一套通用设计思路。

---

## 1. 标准「列表页」长什么样？

通常一个「标准列表页」包含：

1. **查询区（Search Form）**
   - 输入框、下拉、日期区间、状态选择等
   - 「查询」、「重置」按钮

2. **操作区（Toolbar）**
   - 新增、批量操作、导出等

3. **结果表格（Table）**
   - 列信息、操作列
   - 勾选、多选

4. **分页条（Pagination）**
   - 当前页、总数、页大小

可以抽象成一个数据结构：

```ts
interface ListPageConfig {
  searchItems: SearchItem[];
  columns: Column[];
  actions: ActionItem[]; // 顶部操作
  rowActions: ActionItem[]; // 行内操作
}
```

换句话说：**绝大部分差异都可以通过配置表达出来。**

---

## 2. 查询表单的配置化设计

先从搜索区开始。

### 2.1 基本配置格式

我习惯用一个数组描述查询项：

```js
const searchItems = [
  {
    prop: 'keyword',
    label: '关键词',
    type: 'input',
    placeholder: '请输入订单号/姓名',
  },
  {
    prop: 'status',
    label: '状态',
    type: 'select',
    options: [
      { label: '全部', value: '' },
      { label: '待处理', value: 'PENDING' },
      { label: '已完成', value: 'DONE' },
    ],
  },
  {
    prop: 'dateRange',
    label: '创建日期',
    type: 'daterange',
  },
];
```

再写一个通用的 `<SearchForm />` 组件去渲染它：

```vue
<SearchForm
  :items="searchItems"
  v-model="searchModel"
  @search="handleSearch"
  @reset="handleReset"
/>
```

`searchModel` 形如：

```js
{
  keyword: '',
  status: '',
  dateRange: [],
}
```

组件内部根据 `type` 决定用哪个 UI 控件，表单的布局也可以统一处理。

---

### 2.2 查询行为与 URL 同步

和前面路由那篇文章呼应：**查询条件最好能反映到 URL 中**。

```js
methods: {
  handleSearch() {
    this.$router.push({
      name: 'OrderList',
      query: {
        ...this.searchModel,
        page: 1,
      },
    });
  },
  handleReset() {
    this.searchModel = getDefaultSearchModel();
    this.handleSearch();
  },
}
```

而 `<SearchForm />` 只管内聚：

- 维护自己的 `modelValue`
- 把「点查询按钮」这个事件抛出去

---

## 3. 表格列的通用配置

接下来是表格列。常见需求有：

- 格式化显示
- 显示枚举标签
- 可点击跳转
- 操作列需要根据行数据动态展示按钮

我会用类似这样的配置：

```js
const columns = [
  {
    prop: 'orderNo',
    label: '订单号',
    width: 180,
    clickable: true,
  },
  {
    prop: 'customerName',
    label: '客户姓名',
  },
  {
    prop: 'amount',
    label: '金额',
    formatter: (row) => row.amount.toFixed(2),
    align: 'right',
  },
  {
    prop: 'status',
    label: '状态',
    type: 'statusTag',
    options: {
      PENDING: { text: '待处理', type: 'warning' },
      DONE: { text: '已完成', type: 'success' },
    },
  },
  {
    prop: 'createdAt',
    label: '创建时间',
    type: 'datetime',
  },
];
```

在 `<ListTable />` 组件里，根据 `type`/`formatter` 渲染：

```vue
<el-table-column
  v-for="col in columns"
  :key="col.prop"
  :prop="col.prop"
  :label="col.label"
  :width="col.width"
  :align="col.align || 'left'"
>
  <template v-slot="scope">
    <span v-if="col.type === 'statusTag'">
      <StatusTag :value="scope.row[col.prop]" :options="col.options" />
    </span>
    <span v-else-if="col.type === 'datetime'">
      {{ formatDateTime(scope.row[col.prop]) }}
    </span>
    <span v-else-if="col.clickable" class="link" @click="onCellClick(col, scope.row)">
      {{ scope.row[col.prop] }}
    </span>
    <span v-else>
      {{ col.formatter ? col.formatter(scope.row) : scope.row[col.prop] }}
    </span>
  </template>
</el-table-column>
```

这样做的好处：

- 新增一个字段只需要改配置，不改模板；
- 一些常用显示类型（状态标签、金额、时间）可以沉淀成内置 `type`。

---

## 4. 操作区与行内操作

### 4.1 顶部操作按钮

顶部操作（新增、导出、批量删除等）可以用数组描述：

```js
const actions = [
  {
    key: 'create',
    text: '新增订单',
    type: 'primary',
    show: (ctx) => ctx.permissions.includes('order:create'),
  },
  {
    key: 'export',
    text: '导出',
    type: 'default',
    show: (ctx) => ctx.total > 0,
  },
];
```

`ctx` 可以包含当前用户权限、列表状态等上下文。

在 `<ListToolbar />` 里：

```vue
<el-button
  v-for="act in visibleActions"
  :key="act.key"
  :type="act.type || 'default'"
  @click="$emit('action', act.key)"
>
  {{ act.text }}
</el-button>
```

父组件统一处理事件：

```js
methods: {
  handleAction(key) {
    if (key === 'create') {
      this.goCreate();
    } else if (key === 'export') {
      this.exportData();
    }
  },
}
```

---

### 4.2 行内操作按钮

行内操作同理：

```js
const rowActions = [
  {
    key: 'view',
    text: '查看',
    show: (row) => true,
  },
  {
    key: 'edit',
    text: '编辑',
    show: (row) => row.status === 'PENDING',
  },
  {
    key: 'cancel',
    text: '取消',
    confirm: true,
    confirmText: '确认取消该订单？',
    show: (row) => row.status === 'PENDING',
  },
];
```

在 `<ListTable />` 中增加一列「操作」：

```vue
<el-table-column label="操作" fixed="right" width="180">
  <template v-slot="scope">
    <el-button
      v-for="act in visibleRowActions(scope.row)"
      :key="act.key"
      type="text"
      size="small"
      @click="onRowAction(act, scope.row)"
    >
      {{ act.text }}
    </el-button>
  </template>
</el-table-column>
```

`onRowAction` 再把事件抛出来给父组件处理。

---

## 5. 分页与数据获取的约定

数据获取和分页是列表页的核心逻辑，我会约定一套简单的接口契约：

```ts
interface PageRequest {
  page: number;
  pageSize: number;
  // 其他查询条件...
}

interface PageResponse<T> {
  list: T[];
  total: number;
}
```

在页面组件里：

```js
data() {
  return {
    searchModel: getDefaultSearchModel(),
    tableData: [],
    total: 0,
    page: 1,
    pageSize: 20,
    loading: false,
  };
},
methods: {
  async fetchData() {
    this.loading = true;
    try {
      const params = {
        ...this.searchModel,
        page: this.page,
        pageSize: this.pageSize,
      };
      const res = await api.fetchOrderList(params);
      this.tableData = res.list || [];
      this.total = res.total || 0;
    } finally {
      this.loading = false;
    }
  },
  handlePageChange(p) {
    this.page = p;
    this.fetchData();
  },
  handleSizeChange(size) {
    this.pageSize = size;
    this.page = 1;
    this.fetchData();
  },
}
```

分页组件就可以写死在 `<ListPage />` 里，只暴露相关回调。

---

## 6. 抽象成通用 ListPage 组件

前面这些拆完后，其实我们已经可以封装一个通用的 `<ListPage />`：

```vue
<ListPage
  :search-items="searchItems"
  :columns="columns"
  :actions="actions"
  :row-actions="rowActions"
  :request="fetchOrderList"
/>
```

内部大致做的事情：

1. 根据 `searchItems` 渲染搜索区，并维护 `searchModel`。
2. 根据 `columns` 渲染表格列。
3. 根据 `actions` 和 `rowActions` 渲染操作按钮并抛出事件。
4. 根据 `request` 请求数据，并处理分页。

这样之后，一个新模块的列表页往往只需要：

- 定义几组配置；
- 写几个真正的业务事件处理（新增、编辑、导出）。

---

## 7. 踩过的坑与一些小经验

1. **配置不要过度抽象到看不懂**  
   配置项适当即可，太多层封装会导致新人很难上手。命名尽量直白。

2. **保留逃生出口**  
   通用组件应该允许「自定义渲染」插槽，比如表格列支持 `scoped-slot`，避免被配置完全锁死。

3. **接口约定要和后端提前对齐**  
   包括分页字段名（`page` vs `pageNum`）、总数字段（`total` vs `totalCount`）等等，最好统一。

4. **不要强求所有列表都塞进一个 ListPage**  
   超复杂的页面可以只借用部分思想，比如只用配置化表单 + 通用表格，不必所有都走通用组件。

---

## 8. 总结

中后台页面看起来都差不多，本质上是：

> **配置化的查询 + 配置化的表格 + 一点点业务操作。**

如果你能在一个项目里沉淀出一套好用的「列表页模板」，下一次再做类似需求时，心态会轻松很多，也更有空间把精力放在真正有业务价值的地方。

