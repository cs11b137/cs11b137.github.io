+++
title = "用云端 NLP API 做一个简单的文本分析工具"
date = 2021-12-05
draft = false
categories = ["AI 探索"]
tags = ["NLP", "文本分析", "云服务"]
description = "一次早期的 AI 尝试：用现成云端 NLP 接口，对评论/工单做情感和分类分析，并落成一个可用的小工具。"
+++

很早之前就对「AI 能不能帮忙看评论、看工单」这件事好奇过，但自己训模型门槛挺高，于是先选了条**最省事的路**：

> 找一个云厂商提供的 NLP API，用 HTTP 请求的方式做一层简单封装，做一个「文本分析小工具」。

这篇文章记录的是那次尝试的过程。

> 注：下面不强调具体厂商名字，主线思路在于「怎么用」，而不是「用谁」。

---

## 1. 想做一个什么样的小工具？

需求非常朴素：

- 输入一段文本（比如用户评论、客服工单）；  
- 自动判断它是「正向 / 负向 / 中性」；  
- 选做：给出一个大致类别，比如「产品问题 / 物流问题 / 售后问题」。

最终希望变成一个简单的 Web 页面：

- 左边输入框；  
- 右边展示：情感结果、类别标签、关键句/关键词。

---

## 2. 挑选一个云 NLP 服务

大部分云厂商都会提供类似能力：

- 情感分析（Sentiment Analysis）；  
- 文本分类（Text Classification）；  
- 关键词提取、实体识别等。  

通常接入步骤类似：

1. 注册账号；  
2. 在控制台创建一个 NLP 应用 / 项目；  
3. 拿到 API Key / Secret / Endpoint；  
4. 看一眼文档里示例请求。

示例请求（伪代码）：

```http
POST /nlp/sentiment
Authorization: Bearer <API_KEY>
Content-Type: application/json

{
  "text": "这次发货太慢了，等了一个星期才收到。"
}
```

响应大概会是：

```json
{
  "sentiment": "negative",
  "confidence": 0.94
}
```

或者更细一些：

```json
{
  "positive": 0.02,
  "neutral": 0.04,
  "negative": 0.94
}
```

---

## 3. 在前端简单封一层

用最熟悉的方式来就好，比如 Vue + Axios：

```ts
// api/nlp.ts
import axios from 'axios';

const client = axios.create({
  baseURL: import.meta.env.VITE_NLP_BASE_URL,
  headers: {
    'Authorization': `Bearer ${import.meta.env.VITE_NLP_KEY}`,
  },
});

export async function analyzeText(text: string) {
  const [sentRes, classifyRes] = await Promise.all([
    client.post('/nlp/sentiment', { text }),
    client.post('/nlp/classify', { text }),
  ]);

  return {
    sentiment: sentRes.data,
    categories: classifyRes.data,
  };
}
```

在 `.env` 里配置好：

```bash
VITE_NLP_BASE_URL=https://nlp-api.example.com
VITE_NLP_KEY=xxxxx
```

这样可以避免把 key 写死在代码里。

> 实际生产环境建议把调用放到后端/BFF 层，再提供一个安全的前端接口，这里为了演示只在前端直连。

---

## 4. 页面交互：尽量「可视化」一点

页面结构大致是：

- 上半部分：输入框 + 分析按钮；  
- 下半部分：结果卡片。

```vue
<template>
  <div class="nlp-tool">
    <el-input
      v-model="text"
      type="textarea"
      :rows="6"
      placeholder="粘贴一段用户评论或工单内容试试"
    />
    <el-button type="primary" :loading="loading" @click="onAnalyze">
      分析文本
    </el-button>

    <div v-if="result" class="result-panel">
      <ResultSentiment :sentiment="result.sentiment" />
      <ResultCategories :categories="result.categories" />
    </div>
  </div>
</template>
```

`ResultSentiment` 组件可以把结果做得好玩一点：

- 负向：红色表情 + 建议优先处理；  
- 正向：绿色表情 + 简短总结；  
- 中性：灰色。

这样这个小工具不仅能输出「机器结果」，也能给人类一点直觉上的感受。

---

## 5. 它到底准不准？

做完之后，最有趣的部分其实是：

> 拿真实的评论/工单数据来试，看看机器和你的直觉差异在哪。

大概有几种情况：

1. 机器判断得挺准  
   - 比如「这次发货太慢了」→ 负向；  
   - 「客服态度很好」→ 正向。  

2. 机器有点「过于乐观」或「过于悲观」  
   - 带有一些委婉抱怨的句子，有时候会被识别成中性；  
   - 含有否定语气结构时，模型不一定理解上下文。

3. 机器看不懂行业黑话  
   - 很多垂直行业有自己的术语和表达；  
   - 通用模型在这块容易翻车。

但即便不完美，它已经有一个非常实用的用途：

> **帮你快速「筛」出高风险或高价值的文本，让人力聚焦在更重要的部分。**

比如：

- 先用情感分析把负向/极端负向的找出来；  
- 再用分类结果 roughly 看看主要集中在哪几类问题上。

---

## 6. 一点点工程化考虑

哪怕只是一个「玩具项目」，我也顺手做了几件事：

1. 把 NLP 调用封在一个单独模块，而不是在组件里直接写 `axios.post`；  
2. 给返回结果定义了 TS 类型，避免到处都是 `any`；  
3. 简单加了错误提示：接口出错时用 Message 提示，避免静默失败。

这样如果以后要：

- 换一家云厂商；  
- 把调用移到服务端；  
- 增加比如关键词提取、实体识别；  

也不会推翻整个页面，只需要改调用层和类型定义。

---

## 7. 小结：早期 AI 尝试的意义

这次用云端 NLP API 做的小工具，说实话：

- 不算「高深技术」，基本没有自己训模型；  
- 效果也达不到严肃场景直接上线的程度。  

但它给我的感觉是：

1. 把「AI 能力」当成一个服务去消费，其实门槛并不高；  
2. 作为前端/全栈，可以先从 **「把 AI 接进业务」** 这一步开始，而不是一开始就钻研各种模型细节；  
3. 哪怕只是一个内部小工具，能让团队直观感受到：「原来我们已经可以这么用 AI 了」。

等未来有更成熟的场景和数据，再考虑：

- 用更适配领域的模型；  
- 自己搭一层 BFF 做统一的 AI 能力接入；  
- 把这些能力真正织入现有系统的业务流里。  

这次尝试，就当是那个「第一块乐高」吧。

