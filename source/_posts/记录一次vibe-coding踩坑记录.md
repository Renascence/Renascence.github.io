---
title: 记录一次vibe coding踩坑记录
date: 2026-09-10 14:23:39
tags: deepseek nodejs
---

最近在用 Node.js 写一个 BFF 层，接口里需要多次请求后端并聚合数据，逻辑类似这样：

```ts
const dataA = await requestA();
const dataB = await requestB();
const dataC = await requestC();

return merge(dataA, dataB, dataC);
```

## 问题：每个请求都要手动带 Cookie

但在转发请求时涉及登录信息校验，后端接口需要验证 Cookie，所以必须手动带上 Cookie，类似这样：

```ts
const cookie = req.headers.cookie;

const dataA = await requestA({ headers: { cookie } });
const dataB = await requestB({ headers: { cookie } });
const dataC = await requestC({ headers: { cookie } });

return merge(dataA, dataB, dataC);
```

如果接口很多，每个都手动传入 Cookie 就太繁琐了。

于是我和 Copilot 说：“优化一下传参方式，避免每个请求都手动传入 Cookie。”

## 第一次优化：创建带 Cookie 的 axios 实例

随后代码被改成了下面这样：

```ts
// api 请求文件
const createRequest = (cookie) => {
  const request = axios.create({ baseURL: HOST });
  request.interceptors.request.use((config) => {
    config.headers.Cookie = cookie;
    return config;
  });
  return request;
};

// 业务调用：创建实例并放到全局
const globalInst = createRequest(req.headers.cookie);
// request 内部依赖全局的 axios 实例
const requestA = () => {
  globalInst.get('xxxxx');
};
```

乍一看没问题，测试也 OK。

但每次请求都会创建一个 axios 实例，并发情况下可能出现 axios 实例覆盖，导致用户 B 用到用户 A 的 Cookie。这个问题就严重了。

## 追问：并发下会不会串 Cookie？

于是我追问：

“多用户并发的情况下，axios 会相互覆盖导致 bug 吗？”

Copilot 很诚实地回答：同一个实例被多个用户共用，就会串号、互相覆盖，导致严重问题。😂

吐槽：既然你知道会导致问题，为什么不一次帮我改好？🤦‍♂️

## 最终方案：用 AsyncLocalStorage 维护请求上下文

于是代码改成了下面这样：

```ts
// 引入 Node.js 原生 async_hooks 模块，主要用于追踪异步操作、维护异步上下文，以及做监控、性能分析和链路追踪
import { AsyncLocalStorage } from 'async_hooks';

const serverApi = axios.create({ baseURL: HOST });
const requestStorage = new AsyncLocalStorage<Record<string, string>>();

/** 在请求拦截器中自动注入当前上下文的 Cookie */
serverApi.interceptors.request.use((config) => {
  const store = requestStorage.getStore();
  if (store?.Cookie) {
    config.headers.Cookie = store.Cookie;
  }
  return config;
});

/** 在指定的 AsyncLocalStorage 上下文中运行，让每个请求的认证信息互不干扰 */
export function runWithContext<T>(context: Record<string, string>, fn: () => Promise<T>): Promise<T> {
  return requestStorage.run(context ? context : {}, fn);
}

// 业务代码：
await runWithContext({ Cookie: req.headers.cookie }, async () => {
  requestA();
  requestB();
  requestC();
});

// requestA 直接使用 serverApi
const requestA = () => {
  return serverApi.get('xxxxx');
};
```

## 总结

Vibe Coding 如今越来越常见，我在项目中也已经用了一段时间。

可能是个人的原因，我仍然不太放心把业务代码完全交给 AI，所以通常是自己搭好整体框架，在细节实现部分交给 AI。这样每次需要 review 的代码不会太多，有问题也更容易及时发现。

在之前的公司，开发完一个大需求、准备合并到 master 发版时，文件改动往往很多，因此规定：代码分支的 merge 至少需要小组内 2 人以上通过，尽可能保证代码质量，代价就是比较占用人力。

回到 Vibe Coding 上，我的使用感受是：在快速搭建网页、能够明确描述具体展示需求时，它确实很好用，能大幅减少人力投入；但在与业务耦合较紧的模块里做改动，仍需小心谨慎，人工 review 必不可少。