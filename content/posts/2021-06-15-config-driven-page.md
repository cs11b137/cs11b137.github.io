+++
title = "配置驱动页面：表单、表格到「低代码」的第一步"
date = 2021-06-15
draft = false
categories = ["配置化与低代码"]
tags = ["配置驱动", "表单", "表格", "低代码"]
description = "分享一个用 JSON 配置来描述表单和表格，然后在前端解析渲染的实践，是向低代码靠近的一小步。"
+++

在中后台项目里，经常会出现这样的需求：

- 一堆「结构相似、字段不同」的表单；  
- 一堆「上面查询、下面表格」的列表页。  

如果每个页面都手写一次：表单 + 校验 + 表格列 + 操作按钮，不仅枯燥，还很难保持统一风格。

这篇文章讲的是我在项目里做的一次尝试：

> **用一份 JSON 配置描述页面，再写一个「解释器组件」负责渲染表单和表格。**  
> 这是往低代码方向迈出的第一小步。

---

## 1. 先从「配置表单」开始

以一个常见的查询表单为例，我们希望用一份配置来描述：

- 有哪些字段；  
- 每个字段是什么类型（输入框、下拉、日期范围等）；  
- 默认值、占位文案、选项等等。

### 1.1 配置结构示例

```ts
// types.ts
export type FieldType = 'input' | 'select' | 'daterange' | 'switch';

export interface FormField {
  prop: string;
  label: string;
  type: FieldType;
  placeholder?: string;
  options?: Array<{ label: string; value: string | number }>;
  span?: number; // 栅格宽度
  required?: boolean;
}
```

某个页面里的配置：

```ts
// orderListPageConfig.ts
export const searchFields: FormField[] = [
  { prop: 'keyword', label: '关键词', type: 'input', placeholder: '订单号/客户名' },
  { prop: 'status', label: '状态', type: 'select', options: [
    { label: '全部', value: '' },
    { label: '待支付', value: 'PENDING' },
    { label: '已支付', value: 'PAID' },
  ]},
  { prop: 'dateRange', label: '创建时间', type: 'daterange' },
];
```

### 1.2 通用 FormRenderer 组件

```vue
<!-- FormRenderer.vue（示意） -->
<template>
  <el-form :model="model" class="config-form" label-width="90px">
    <el-row :gutter="16">
      <el-col
        v-for="field in fields"
        :key="field.prop"
        :span="field.span || 8"
      >
        <el-form-item :label="field.label" :prop="field.prop">
          <component
            :is="getComponent(field)"
            v-model="model[field.prop]"
            v-bind="getComponentProps(field)"
          />
        </el-form-item>
      </el-col>
    </el-row>

    <el-form-item>
      <el-button type="primary" @click="$emit('search', model)">查询</el-button>
      <el-button @click="onReset">重置</el-button>
    </el-form-item>
  </el-form>
</template>
```

`getComponent` 根据类型返回不同组件：

```ts
function getComponent(field: FormField) {
  switch (field.type) {
    case 'input':
      return 'el-input';
    case 'select':
      return 'el-select';
    case 'daterange':
      return 'el-date-picker';
    default:
      return 'el-input';
  }
}
```

`getComponentProps` 返回对应属性（占位、选项等）。

使用时页面只需要：

```vue
<FormRenderer
  :fields="searchFields"
  v-model="searchModel"
  @search="handleSearch"
/>
```

---

## 2. 再把「表格」也配置起来

同理，表格列也可以用配置描述：

```ts
export type ColumnType = 'text' | 'money' | 'datetime' | 'statusTag';

export interface TableColumn {
  prop: string;
  label: string;
  width?: number;
  type?: ColumnType;
  align?: 'left' | 'center' | 'right';
  formatter?: (row: any) => string;
}
```

示例：

```ts
export const tableColumns: TableColumn[] = [
  { prop: 'orderNo', label: '订单号', width: 180 },
  { prop: 'customerName', label: '客户姓名' },
  { prop: 'amount', label: '金额', type: 'money', align: 'right' },
  { prop: 'status', label: '状态', type: 'statusTag' },
  { prop: 'createdAt', label: '创建时间', type: 'datetime', width: 160 },
];
```

通用 `<TableRenderer>` 组件根据配置渲染：

```vue
<el-table :data="data">
  <el-table-column
    v-for="col in columns"
    :key="col.prop"
    :prop="col.prop"
    :label="col.label"
    :width="col.width"
    :align="col.align || 'left'"
  >
    <template #default="{ row }">
      <span v-if="col.type === 'money'">
        {{ formatMoney(row[col.prop]) }}
      </span>
      <span v-else-if="col.type === 'datetime'">
        {{ formatDateTime(row[col.prop]) }}
      </span>
      <StatusTag
        v-else-if="col.type === 'statusTag'"
        :value="row[col.prop]"
      />
      <span v-else>
        {{ col.formatter ? col.formatter(row) : row[col.prop] }}
      </span>
    </template>
  </el-table-column>
</el-table>
```

---

## 3. 拼装成「配置驱动列表页」

最后，我们再加上分页和操作区，用一个通用 `ListPage` 组件把这些组合起来：

```vue
<ListPage
  :search-fields="searchFields"
  :columns="tableColumns"
  :request="fetchOrderList"
/>
```

内部做的事情：

1. 使用 `FormRenderer` 渲染查询区；  
2. 监听 search 事件发起请求；  
3. 使用 `TableRenderer` 渲染表格；  
4. 管理分页信息、loading 状态；  
5. 为常见操作预留插槽（比如顶部按钮、自定义列等）。

页面本身只负责：

- 提供配置；  
- 实现 `fetchOrderList` 请求；  
- 处理业务上的自定义操作（导出、批量操作等）。

---

## 4. 这算不算低代码？

严格意义上，它还不是「给运营可视化拖拽」那种低代码平台，但它具备了几个低代码的核心特征：

1. **页面结构由配置驱动**  
   - 改一个字段、加一个查询条件，只要改配置；  
   - 结构统一、风格一致。

2. **表单和表格的「解释器」可复用**  
   - 新开一个类似的页面，成本非常低；  
   - 只要写配置而不是从零写模板。

3. **有能力进一步抽象到可视化**  
   - 后续完全可以在后台给这些 JSON 配置配一套可视化编辑器；  
   - 前端渲染层不需要改。

所以可以把它看作是：**从「纯代码写死页面」向「配置驱动」过渡的第一步。**

---

## 5. 踩坑与权衡

1. **配置不要「无限泛化」**  
   - 一上来就支持各种嵌套布局、奇怪交互，配置会很快失控；  
   - 建议先 cover 70% 常见场景，剩下的靠插槽自定义。

2. **调试体验要考虑**  
   - 对开发同学来说，完全看不到模板可能会不适应；  
   - 可以在文档里配一些「配置 → 渲染结果」的示例截图，降低心智负担。

3. **和设计规范打通**  
   - 配置里如果乱写样式，会破坏统一性；  
   - 把样式收敛到组件内部，让配置更专注于“结构与含义”。

---

## 6. 小结

配置驱动页面，本质是在做三件事：

1. 把页面中高度重复的结构（表单、表格、分页）抽出解释器；  
2. 用一份 JSON 来描述「页面长什么样」；  
3. 让新增/修改页面变成「改配置」而不是「改模板」。  

它还谈不上「完整低代码平台」，但已经能明显减少重复劳动、统一交互体验，也为以后更上层的可视化搭建预留了空间。

