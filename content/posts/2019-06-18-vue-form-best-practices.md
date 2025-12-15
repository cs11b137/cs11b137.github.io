+++
title = "复杂表单实战：我在项目里踩过的 Vue 坑"
date = 2019-06-18
draft = false
categories = ["前端基础"]
tags = ["Vue", "表单", "表单校验"]
description = "记录一个复杂表单项目中，关于校验、动态字段和联动逻辑的设计与踩坑。"
+++

表单是前端最常见、也最容易写成一坨的东西。简单表单随便放几个 `v-model` 都能跑，一旦字段多起来、存在动态联动、跨步校验，代码就开始失控。

这篇文章基于我做过的一个"几十个字段 + 多步 + 条件显示"的表单项目，总结一些在 Vue 里踩过的坑和解决方案。

---

## 需求背景：一个略显「超标」的表单

当时的表单大概长这样：

- 3 个步骤：基础信息 → 业务信息 → 确认提交
- 每步有 8～15 个字段
- 部分字段需要「根据上一步选择动态显示或隐藏」
- 有跨字段校验（比如：结束日期 ≥ 开始日期）
- 需要支持「草稿保存」+「再次打开自动回填」

如果**一股脑全写在一个组件里**，大概会出现：

- data 里一个巨大的 `form = { ... }`。
- 各种 `watch` 做联动逻辑。
- methods 里各种 `validateXXX`，互相依赖。
- 维护成本爆炸。

---

## 核心思路：**拆数据结构 + 拆 UI + 拆校验**

### 1. 数据结构要先定清楚

我一般会在一开始，就写出一个完整的 `formModel`：

```js
const defaultForm = () => ({
  basic: {
    name: '',
    idNo: '',
    phone: '',
  },
  business: {
    type: '',
    amount: null,
    startDate: '',
    endDate: '',
  },
  extra: {
    agree: false,
    comment: '',
  },
});
```

在组件里：

```js
data() {
  return {
    form: defaultForm(),
  };
}
```

这样做的好处：

- 字段分组一目了然（basic / business / extra）。
- 默认值清晰，重置时只需 `this.form = defaultForm()`。
- 后面多步拆组件的时候，很容易针对某一组传参。

---

### 2. UI 组件分步拆解

与其一口气在一个页面上写所有表单，不如**每一步拆成一个子组件**：

- `StepBasic.vue`
- `StepBusiness.vue`
- `StepExtra.vue`

每个子组件只负责「渲染 + 局部简单校验」，数据透传上层统一管理。

父组件大致结构：

```vue
<template>
  <div>
    <StepBasic
      v-if="currentStep === 1"
      v-model="form.basic"
    />
    <StepBusiness
      v-if="currentStep === 2"
      v-model="form.business"
    />
    <StepExtra
      v-if="currentStep === 3"
      v-model="form.extra"
    />

    <!-- 上一步 / 下一步 / 提交按钮 -->
  </div>
</template>
```

子组件写法（以 StepBasic 为例）：

```vue
<template>
  <div>
    <label>姓名</label>
    <input v-model="localValue.name" />

    <label>证件号</label>
    <input v-model="localValue.idNo" />
  </div>
</template>

<script>
export default {
  name: 'StepBasic',
  props: {
    value: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      localValue: { ...this.value },
    };
  },
  watch: {
    localValue: {
      deep: true,
      handler(val) {
        this.$emit('input', val);
      },
    },
    value: {
      deep: true,
      handler(val) {
        this.localValue = { ...val };
      },
    },
  },
};
</script>
```

> Vue 2 时代这种 `v-model` + `localValue` 的写法挺常见。Vue 3 里可以用 `v-model` 自定义修饰符会更优雅，这里先不展开。

---

## 动态字段：不要「if 写死在模板里」

常见写法是：

```html
<div v-if="form.business.type === 'A'">
  <!-- 一坨字段 -->
</div>
<div v-if="form.business.type === 'B'">
  <!-- 另一坨字段 -->
</div>
```

这会导致模板变得非常长，而且逻辑不好复用。

更推荐的是**用配置驱动**：

```js
// fieldsConfig.js
export const businessFields = [
  {
    prop: 'amount',
    label: '金额',
    showWhen: (form) => true,
  },
  {
    prop: 'discountRate',
    label: '折扣率',
    showWhen: (form) => form.business.type === 'A',
  },
  {
    prop: 'extraInfo',
    label: '补充说明',
    showWhen: (form) => form.business.type === 'B',
  },
];
```

组件里：

```vue
<div v-for="field in visibleFields" :key="field.prop">
  <label>{{ field.label }}</label>
  <input v-model="localValue[field.prop]" />
</div>
```

```js
computed: {
  visibleFields() {
    return businessFields.filter(f => f.showWhen(this.formRoot)); // formRoot 指向整个 form
  },
},
```

这样：

- 动态显示逻辑放在配置里，想改只改 `showWhen`。
- 也方便后面抽离出通用的「配置化表单组件」。

---

## 校验：用规则系统，而不是到处手写 `if`

以前我常在 `submit` 里写一堆：

```js
if (!this.form.basic.name) { alert('请输入姓名'); return; }
if (!this.form.basic.idNo) { alert('请输入证件号'); return; }
if (!this.form.business.amount || this.form.business.amount <= 0) { ... }
```

后来发现，还是应该让校验**也配置化**。

假设用的是 Element UI 的 Form：

```js
const rules = {
  'basic.name': [
    { required: true, message: '请输入姓名' },
  ],
  'business.amount': [
    { required: true, message: '请输入金额' },
    { type: 'number', min: 0.01, message: '金额必须大于 0' },
  ],
};
```

跨字段校验可以用自定义 validator：

```js
'business.endDate': [
  {
    validator: (rule, value, callback) => {
      const { startDate, endDate } = form.business;
      if (endDate && startDate && endDate < startDate) {
        callback(new Error('结束日期不能早于开始日期'));
      } else {
        callback();
      }
    },
    trigger: 'change',
  },
],
```

总结一下：

- **常规规则→配置化**，尽量放到 `rules` 对象里。
- **复杂逻辑→自定义 validator**，用函数来组织逻辑，而不是散落在各个方法里。

---

## 草稿保存与回填

草稿保存是复杂表单的常见需求：用户填了一半离开，下次回来要恢复。

基本思路：

1. 表单提交接口之外，增加「保存草稿」接口（或 localStorage 本地存储）。
2. 草稿保存实际就是「把当前 `form` 的 JSON 存起来」。 
3. 打开页面时请求草稿，如果有，就用草稿替代默认值。

伪代码：

```js
methods: {
  saveDraft() {
    // 简单场景可以直接存 localStorage
    localStorage.setItem('FORM_DRAFT', JSON.stringify(this.form));
  },
  loadDraft() {
    const raw = localStorage.getItem('FORM_DRAFT');
    if (raw) {
      this.form = Object.assign(defaultForm(), JSON.parse(raw));
    }
  },
},
created() {
  this.loadDraft();
}
```

注意：

- 回填时**一定记得先用 `defaultForm()` 打个底**，再 `Object.assign`，防止接口字段变更导致字段丢失。
- 如果是和后端交互草稿接口，建议在后端也做一次字段兜底。

---

## 我踩过的几个坑

1. **在子组件直接修改 props**

   ```js
   // 坑
   props: ['value'],
   created() {
     this.value.xxx = '123'; // 违反单向数据流
   }
   ```

   构造一个 `localValue`，通过 `v-model` 同步回父组件会更健康。

2. **watch 里逻辑过多**

   如果你发现某个 `watch` 函数超 30 行，那大概率说明逻辑该拆了，可以抽出成普通方法、甚至独立模块。

3. **校验时混用了多套规则**

   有些字段既写了组件库的规则，也又手写了额外的 `if` 判断，容易出现「一个地方改了校验，另一个地方忘记改」的问题。建议统一走一套规则体系。

---

## 总结

复杂表单本质是个**状态管理 + UI 编排问题**：

- 先把数据结构设计好（默认值、分组）。
- 再用组件拆分 UI（一步一个组件）。
- 动态展示和校验尽量配置化，不要写死在模板里。
- 共性逻辑抽出来，下一次做类似表单就会轻松很多。

表单不会消失，但写起来可以比「大 if 地狱」舒服得多。
