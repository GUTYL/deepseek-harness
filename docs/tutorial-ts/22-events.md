# 第 22 章 · 事件系统与五种投递模式

## 本节要学的概念

- 事件是「不认识对方也能通信」的机制,靠 `ctx.on` 声明、`ctx.emit` 等派发。
- 五种投递模式(emit/parallel/serial/bail/waterfall)各自语义。
- waterfall 的「中间件 + 短路」语义,以及项目的铁律「要么 next() 要么截断」。

## 事件 vs 服务:两种通信

第 21 章的 Service 是「**直接找某个人**」(`ctx.greeter.greet(...)`)。**事件(event)**是「**对广播喊话,谁想听谁听**」——发的人不需要知道谁在听。

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'stats/report'(name: string, count: number): void   // ① 声明一个新事件(第 14 章)
  }
}

this.ctx.emit('stats/report', name, next)   // ② 发送(广播)

ctx.on('stats/report', (name, count) => {   // ③ 接收(可逆注册,卸载自动移除)
  console.log(`[stats] ${name} -> ${count}`)
})
```

三件套对应你在前面学过的:

- **声明**事件 = `declare module` 合并 `Events` 接口(第 14 章)。
- **类型化派发器** = 第 15 章 `dispatch.ts` 那套,从 `Events` 推导出精确参数。
- **可逆监听** = `ctx.on` 是 effect,插件卸载自动清理(第 20 章)。

这个模式贯穿全项目:工具调用结果、模型请求、审批决策,都用事件来沟通。

## 五种投递模式

一个事件到底「怎么派发给多个监听者」,取决于它声明的**模式(mode)**。这是事件契约的核心(见 [`cordis-primer.md`](../cordis-primer.md) 的「Dispatch Modes」):

| 模式 | 调用方式 | 是否 await | 顺序/并发 | 有无返回值 |
|---|---|---|---|---|
| `emit` | `ctx.emit` | 否 | 按注册顺序观察 | 无 |
| `parallel` | `await ctx.parallel` | 是 | 所有监听并发 | 无 |
| `serial` | `await ctx.serial` | 是 | 按注册顺序 | 有(首个非空返回胜出) |
| `bail` | `ctx.bail` | 否 | 按注册顺序 | 有(serial 的同步版) |
| `waterfall` | `ctx.waterfall` | 是 | 中间件式包裹 | 有 |

区分它们的核心两个维度:**要不要等**(emit/bail 不 await,其余 await)和**监听者之间如何协作**(独立观察 / 并发 / 顺序短路 / 中间件包裹)。

事件用 `@mode` 标注它的模式(你在 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts) 里见过的 `@mode emit` 就是这种标注),生成目录会据此检查「声明」和「派发点」是否一致。所以在项目里,每个事件**该怎么派发是它的公开契约**,不能随便换。

## waterfall:项目的「拦截」主力

五种模式里,`waterfall` 最特殊,也最常用。它是**洋葱圈中间件(around-middleware)**:每个监听者收到 `(...args, next)`,调用 `next()` 把控制权交给下一个,拿到下一个的返回值再包一层返回。

```ts
ctx.on('demo/transform', async (input, next) => {
  const downstream = await next()      // 先让下游跑完
  return downstream.toUpperCase()      // 再改造结果(包裹)
})

ctx.on('demo/transform', async (input, next) => {
  if (input.includes('blocked')) return '** blocked **'   // 截断,不调 next
  return next()
})
```

两个关键行为(来自 [`cordis-tutorial/04-events.md`](../cordis-tutorial/04-events.md)):

1. **包裹**:调 `next()` 然后改它的返回值。
2. **短路**:不调 `next()` 直接返回,后面所有监听者(含默认实现)都跳过。

由此得出项目的**一条铁律**(根 AGENTS.md 明确写了):

> **waterfall 监听者要么调用 `next()`,要么就是刻意截断。** 只是「观察/记日志」却忘了 `next()`,会悄悄吞掉下游默认行为。

所以 waterfall 监听者分成两类:

- **观察/标注型**:必须 `next()`(可能顺便改一下共享对象)。
- **决策型**:当它「拥有这个决策」时不 `next()`,直接给出答案。

项目用 waterfall 做真正的决策点,例如 `agent/request`(替换模型调用配置)、`approval/request`(审批策略代替用户回答)——这些在第 25 章会遇到。

## 呼应第 15 章:类型化派发

现在回头看第 15 章精读的 [`dispatch.ts`](../../packages/core/agent/src/dispatch.ts),它的 `AgentEventDispatch` 接口正好对应这里的模式:

```ts
interface AgentEventDispatch {
  emit<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>): void
  serial<K extends AgentSubjectEvent>(...): Promise<Awaited<Return<Events[K]>>>
  waterfall<K extends AgentSubjectEvent>(...): Return<Events[K]>
}
```

注意它**没有** `parallel` 和 `bail`——因为这个派发器只封装了 agent 主题事件实际用到的三种模式。而 `serial` 返回 `Promise<Awaited<...>>`、`waterfall` 返回裸的 `Return<...>`,正是上表「serial 有返回值、waterfall 有返回值、emit 无返回值」的类型投影。第 15 章的类型机器,就是**把这张「模式表」翻译成精确的 TS 签名**。

## 你现在能自己读了

1. 回到 [`cordis-primer.md`](../cordis-primer.md) 的「Dispatch Modes」表和「Cordis Waterfall Semantics」两节,现在你能把它们当成查表式的速查,而不是陌生概念。

2. 打开 [`packages/core/agent/src/runtime-types.ts`](../../packages/core/agent/src/runtime-types.ts),找 `@mode waterfall`、`@mode emit` 这些标注,看哪些事件是 waterfall(可拦截决策),哪些是 emit(纯通知)。思考:为什么「模型请求前」是 waterfall,而「状态变化通知」是 emit?

## 小结

- 事件三件套:`declare module` 声明 + 类型化派发 + 可逆监听 `ctx.on`。
- 五种投递模式,核心区分「是否 await」和「监听者如何协作」。
- waterfall 是中间件:调 `next()` 包裹,不调 `next()` 短路;铁律是「要么 next 要么截断」。

## 思考题

1. `emit` 和 `parallel` 一个区别是「是否 await」,这对「观察型监听器」意味着什么?为什么通知类事件常用 emit?
2. 一个 waterfall 的「日志型」监听器忘了 `next()`,会发生什么?为什么这是最隐蔽的 bug 之一?
3. 为什么「一个事件用什么模式」是它的公开契约,不能随便改?(结合「监听者按模式假设语义」来答。)

---

上一章:[第 21 章 · Service 与依赖注入](21-services.md)
下一章:[第 23 章 · LLM 词汇表:消息与流](23-llm-vocabulary.md)
