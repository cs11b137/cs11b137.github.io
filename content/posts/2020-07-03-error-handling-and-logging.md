+++
title = "别再只 console.log 了：前端错误收集与日志治理"
date = 2020-07-03
draft = false
categories = ["监控与运维"]
tags = ["错误监控", "日志", "埋点"]
description = "介绍前端异常捕获方案、日志上报字段设计以及简单报表实践。"
+++

大部分前端开发一开始排查问题都是靠：

```js
console.log('看看这里是什么');
```

这没问题，但当项目变大、用户变多、场景变复杂时，只靠 `console.log` 就完全不够用了——**用户出问题时你根本看不到现场。**

这篇文章聊聊我在实际项目里做前端错误收集和日志治理的一些经验。

---

## 1. 我们到底需要记录什么？

先看一个真实场景：

- 用户反馈：**「页面有时候会白屏」**  
- 操作步骤：「我就是正常点点点，突然就白了」  
- 浏览器：未知  
- 系统：未知  
- URL：未知  

如果我们没有任何前端错误监控工具，这个问题几乎无从排查。

所以，一套基础的前端日志/错误系统至少要能回答：

- 什么时间？
- 哪个用户？
- 在哪个页面？
- 用的什么浏览器/系统？
- 做了什么操作？
- 报了什么错（堆栈、接口信息等）？

所以日志结构（特别是错误日志）要包含几个核心字段：

```ts
interface FrontendLog {
  level: 'info' | 'warn' | 'error';
  message: string;
  stack?: string;
  url: string;
  userId?: string;
  time: number;
  ua: string; // user agent
  extra?: Record<string, any>;
}
```

---

## 2. 捕获错误的几个入口

### 2.1 全局 JS 运行时错误

```js
window.onerror = function (message, source, lineno, colno, error) {
  reportError({
    message,
    stack: error && error.stack,
    url: window.location.href,
    extra: { source, lineno, colno },
  });
};
```

注意：

- 这个能捕获**同步脚本错误**。
- 对跨域脚本，如果没有正确配置，会只得到 `Script error.`，需要在 script 标签上加 `crossorigin` 且服务端正确设置 CORS/SourceMap。

---

### 2.2 Promise 未处理异常

现代前端很多逻辑是基于 Promise / async 的，`window.onerror` 捕不到，需要监听：

```js
window.addEventListener('unhandledrejection', function (event) {
  reportError({
    message: event.reason && event.reason.message,
    stack: event.reason && event.reason.stack,
    url: window.location.href,
    extra: { type: 'unhandledrejection' },
  });
});
```

这可以帮你发现「忘记 try/catch」或者 Promise 链没有 `.catch` 的问题。

---

### 2.3 资源加载错误（图片、脚本等）

```js
window.addEventListener('error', function (event) {
  const target = event.target || event.srcElement;
  if (target instanceof HTMLScriptElement || target instanceof HTMLLinkElement || target instanceof HTMLImageElement) {
    reportError({
      message: 'ResourceLoadError',
      url: window.location.href,
      extra: {
        tagName: target.tagName,
        src: target.src || target.href,
      },
    });
  }
}, true);
```

关键点：

- 第三个参数传 `true`，使用捕获阶段才能拿到资源加载错误。
- 可以帮你发现 CDN 路径错误、静态资源 404/403 等问题。

---

### 2.4 HTTP 请求错误（如 Axios 拦截器）

对于接口错误，我通常在请求库层面统一拦截：

```js
service.interceptors.response.use(
  (response) => response,
  (error) => {
    // 上报接口错误
    reportError({
      message: 'ApiError',
      url: window.location.href,
      extra: {
        api: error.config && error.config.url,
        method: error.config && error.config.method,
        status: error.response && error.response.status,
        response: error.response && error.response.data,
      },
    });
    return Promise.reject(error);
  }
);
```

网络层的错误往往和具体业务强相关，上报时注意**不要带敏感数据**（比如用户隐私、完整报文等）。

---

## 3. 把日志发到哪里？

最简单粗暴的方式是：

```js
function reportError(payload) {
  try {
    navigator.sendBeacon('/log/fe', JSON.stringify(payload));
  } catch (e) {
    // 兜底逻辑：也可以用 ajax
    const img = new Image();
    img.src = '/log/fe?data=' + encodeURIComponent(JSON.stringify(payload));
  }
}
```

为什么推荐 `sendBeacon`：

- 异步、不会阻塞页面卸载；
- 适合在 `beforeunload` 等场景上报日志。

日志接收服务可以先做得很简单：

- 一个后端接口，把日志写入数据库/日志系统；
- 后面再接报表/分析工具。

在设计日志接口时要注意：

- 控制单条日志大小（避免把整个响应体都塞进去）；
- 控制频率（防止某个循环报错刷爆服务）；
- 避免敏感信息（脱敏用户数据）。

---

## 4. 日志内容如何「结构化」？

比起一坨字符串，更推荐结构化日志。比如请求错误：

```json
{
  "level": "error",
  "type": "ApiError",
  "message": "Request failed with status code 500",
  "url": "https://app.example.com/order/list",
  "time": 1593755030000,
  "userId": "u_12345",
  "extra": {
    "api": "/api/order/list",
    "method": "post",
    "status": 500,
    "bizCode": "ORDER_LIST_INTERNAL_ERROR"
  }
}
```

这样在做查询统计时，就可以：

- 按 `type` 聚合错误类型；
- 按 `api` 看哪个接口最爱报错；
- 按 `bizCode` 分析业务错误。

**经验：**

- `type` 字段非常重要，代表错误的大类，比如 `JsError`、`ApiError`、`ResourceError`；
- `bizCode` 可以由后端返回，代表更细的业务错误类型。

---

## 5. 简单做几个「可视化报表」

有日志之后，如果没人看，也等于没有。至少可以做几张最基本的报表：

1. **错误趋势图**  
   - 按天（或小时）统计错误数量。
   - 看看有没有某天突然变高。

2. **按错误类型统计 TOP N**  
   - 哪种错误出现最多（JsError / ApiError / ResourceError）。

3. **按接口统计 TOP N**  
   - 哪些接口报错最多，方便集中排查。

4. **按浏览器/操作系统分布**  
   - 某些错误是否只在特定浏览器出现。

这些报表不必一开始就做得很复杂，用 BI 工具简单拉几个图，就已经能帮助你捕捉很多「用户没来得及反馈，但已经出问题」的情况。

---

## 6. 和「埋点」打通：从出错到复现路径

错误日志只能告诉你「哪里错了」，但很多时候你还想知道：

> 用户在出错之前到底做了什么？

这时候就要配合埋点：

- 对关键操作埋点，比如：点击按钮、表单提交、页面切换等；
- 为每个用户会话分配一个 `sessionId`；
- 错误日志里带上 `sessionId`。

这样你就可以在后台：

1. 看到某一个错误；
2. 根据这个错误的 `sessionId`，拉出该会话下的操作轨迹；
3. 基本还原用户的操作流程。

当然，埋点本身也是一个很大的话题，这里只强调一点：**和错误日志打通后，价值会翻倍。**

---

## 7. 什么时候该「忽略」错误？

不是所有错误都值得上报，有些属于「环境问题」或「恶意脚本」，比如：

- 某些浏览器插件注入脚本导致的错误；
- 明显不是你业务域名下的资源加载失败；
- 偶发的网络抖动（比如 499/timeout），可以带重试逻辑。

我的做法是：

1. 在 `reportError` 之前做一层过滤：

```js
function shouldReport(err) {
  if (!err) return false;
  if (err.message && err.message.includes('Script error')) return false;
  // 其他过滤规则...
  return true;
}
```

2. 在后台报表中对「噪音错误」做一个标记或过滤。

目的不是追求日志「一条不漏」，而是让真正有价值的错误能被看见。

---

## 8. 小结：从 log 到治理

错误日志不是目的，**把问题发现出来并解决掉**才是。

一个比较理想的闭环是：

1. 前端统一捕获错误 → 上报结构化日志；  
2. 后端存储 + 简单报表 → 每周/每天看一次；  
3. 为高频错误创建任务单 → 排查 + 修复；  
4. 修复后继续观察对应指标是否下降。  

相比一味地在代码里加 `console.log`，这套体系一旦跑起来，你会明显感受到：

- 用户反馈的问题更容易复现；
- 很多问题你可以先发现、先解决，而不是等用户骂；
- 团队对「线上质量」这件事会更有感知。

**一句话总结：**

> 把错误「看见」并系统性地记录下来，是把项目从「能跑」变成「靠谱」的第一步。

