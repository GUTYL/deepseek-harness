# 第 25 章 · Agent 主循环:turn/step 状态机

## 本节要学的概念

- 用全部 TS 知识,读懂 `ReactLoopAgent`——项目的主循环驱动。
- 理解 turn(回合)/ step(步骤)两个层次,和 `Phase` 状态机。
- 「模型可见 ⟺ 已记录」这条铁律如何落到代码。

## 主角:ReactLoopAgent

打开 [`packages/core/agent-loop/src/agent.ts`](../../packages/core/agent-loop/src/agent.ts)。这是第 24 章说的「Provider 实现 Definition」中的那半个 Provider——`ReactLoopAgent implements Agent`。

它的职责一句话:`驱动一个会话,跨过 turn 和 step 的边界。` 文件顶部注释原话:「Default Agent driver over queued turns and step-boundary input. Every request is derived from the session log.」

先把两个概念钉死(来自 [`architecture.md`](../architecture.md) 的「Turn flow」):

- **step(步骤)**:一次模型请求 + 它调用的所有工具。
- **turn(回合)**:0 个或多个 step。在第一个输入被认领前开始,在「没有任何事欠着」时结束。

## 状态机:Phase 判别联合

读代码第一眼看到的是这个(第 33~46 行附近):

```ts
type Phase =
  | { kind: 'idle'; lastTurn: number }
  | {
    kind: 'maintenance'
    abort: AbortController
    lastTurn: number
    wakeRequested: boolean
  }
  | { kind: 'running'; abort: AbortController; turn: number; step: number; wakeRequested: boolean }
```

这是第 7 章的**判别联合**:用 `kind` 标签区分「空闲 / 维护 / 运行」三种状态。每个状态带不同字段:

- `idle`:只记得上次的 turn 号。
- `maintenance`:带一个 `AbortController`(用于取消)。
- `running`:带 `abort`、当前 `turn`、`step`、`wakeRequested`(需不需要被唤醒)。

`AbortController` 你可以理解为「一个可中途拉响的取消信号」——主循环用它支持「用户打断」。

## 构造函数:把前面的概念全串起来

构造函数(第 80~97 行)是「Cordis 概念 + TS 语法」的浓缩:

```ts
constructor(loopCtx, id, options, session) {
  this.dispatch = agentEvents(loopCtx, this)       // 第 15 章的类型化派发器,复用
  this.inbox = new Inbox(session, {                 // 输入队列(收件箱)
    inserted: (message) => { this.dispatch.emit('agent/inbox/inserted', { message }) },
    // ...
  })
  const lastTurn = session.events.findLast(event => event.type === 'turn/start')?.data.turn ?? 0
  this.phase = { kind: 'idle', lastTurn }
  this.scope = createScope(loopCtx, this)           // 第 24 章的 scope 原语
  this.ctx = this.scope.ctx.extend({ agent: this }) // 作用域化上下文
  this.runtimeContext = new RuntimeContextProjection(this.ctx, session)
}
```

逐项对照前面章节:

- `agentEvents(loopCtx, this)` —— 第 15 章那个从 `Events` 推导出的派发器。`this` 是当前 agent,被注入进 payload。
- `new Inbox(...)` —— **收件箱**,输入(用户消息、注入的上下文)通过它统一进入。
- `session.events.findLast(...)?.data.turn ?? 0` —— 第 21 章的可选链 `?.` 和空值合并 `??`(TS 基本语法)。从 session 日志里找最近一次 turn 号,决定从哪继续。
- `createScope(loopCtx, this)` + `.ctx.extend({ agent: this })` —— 第 24 章的「作用域」:创建一个「这个 agent 专属」的子上下文,里面挂上 `ctx.agent = this`。

注意 `?.` 和 `??` 这类现代 TS/JS 语法,项目里处处可见,你已经会读。

## turn():流程图逐行翻译

`turn()` 方法(第 246~330 行)基本是架构文档那张流程图的逐行实现。核心骨架:

```ts
private async turn(): Promise<boolean> {
  // 开启回合边界:记录一条 durable 事件
  this.session.append('turn/start', { turn })
  // ...
  while (true) {
    const decision = await this.preStep(target, { turn, step })
    if (decision.kind === 'reject') { turnEnds = { kind: 'blocked' }; return false }
    // 记录用户消息
    this.session.append('step/start', { turn, step })
    for (const message of decision.messages) {
      this.session.append('user/message', message, { surfaceOp: 'append' })
    }
    const stepEnd = await this.step(decision.assembly)   // 走一步
    this.session.append('step/end', { turn, step })
    // 若还欠请求或来了新输入 → 下一步
  }
  // ...
  this.session.append('turn/end', { turn, reason: turnEnds! })
}
```

几个要点:

- `this.session.append('turn/start', {...})` —— **往 session 日志里追加记录**。`append` 的第一个参数是事件名(第 14 章 `SessionEventMap` 里的键),第二个是 payload。
- `decision.kind === 'reject'` —— 第 6/7 章的判别联合收窄:走 `agent/pre-step` 这个 waterfall(第 22 章),插件可以返回「拒绝(reject)」或「进入(enter)」,`decision` 的类型是 `PreparedStep` 判别联合。
- `turnEnds!` 的 `!` —— 第 3/16 章的非空断言,作者担保走到这里 `turnEnds` 一定已赋值。

## step():一次模型调用 + 工具执行

`step()`(第 332~401 行)做两件事:调模型,执行它要的工具。关键一行在 `buildRequest`:

```ts
const { request, preparedCall } = await this.buildRequest(
  turn, step, assembly.tools, system,
  this.session.deriveMessages(),   // ← 从日志推导模型要看到的历史
  signal,
)
```

`this.session.deriveMessages()` 是**理解项目架构的支点**:它从 session 日志**推导出模型的对话历史**。这引出一条铁律(第 1 章提过,现在落地):

> **模型可见 ⟺ 已记录**。任何要进模型请求的东西,都必须能从 session 日志重建。

这就是为什么「新增一个模型可见的输入,就要新增一个 session 事件」——你要扩展 `SessionEventMap`(第 14 章的声明合并),并从日志里渲染它。session 日志是**唯一真相源**:模型上下文、UI 回放、fork、resume、遥测、持久化,全从这条流衍生。

再看流式调用(第 346~350 行):

```ts
const stream = preparedCall?.stream(request) ?? this.loopCtx.llm.stream(request)
for await (const chunk of stream) {
  chunkSeqs.push(this.session.append('assistant/chunk', { turn, step, chunk }).seq)
  assembler.push(chunk)
}
```

`for await (const chunk of stream)` 是**异步迭代**(async iteration)——逐块消费第 23 章的 `StreamChunk` 流,每块都 `append('assistant/chunk', ...)` 记日志(所以能 token 级回放)。`preparedCall?.stream(...) ?? this.loopCtx.llm.stream(...)` 是可选链 + 空值合并。

## kick() 与容错边界

`kick()`(第 210~223 行)是驱动入口:

```ts
private async kick(): Promise<void> {
  try {
    while (await this.turn()) {}
  } catch (_error) {
    // Reported failures and cancellation are contained at the driver boundary.
  } finally {
    if (this.phase.kind === 'running') {
      const { turn, wakeRequested } = this.phase
      this.setPhase({ kind: 'idle', lastTurn: turn })
      if (wakeRequested && this.inbox.hasPending) this.wakeDriver()
    }
  }
}
```

注意 `catch (_error)` 里参数名带下划线 `_error`——对应项目开启的 `noUnusedParameters`:未使用的参数要用下划线前缀声明,让编译器知道「这是我故意不用的」。这个细节呼应第 2 章的严格开关。

## 你现在能自己读了

把 [`packages/core/agent-loop/src/agent.ts`](../../packages/core/agent-loop/src/agent.ts) 从头到尾完整读一遍(约 496 行)。现在你应该能:

- 认出 `Phase`、`PreparedStep` 是判别联合(第 7 章)。
- 读出 `turn()`/`step()` 是架构文档流程图(第 25 章开头那张)的翻译。
- 理解每个 `session.append(...)` 在「记录真相」,每个 waterfall/serial 是扩展点(第 22 章)。
- 看懂 `?.`、`??`、`!`、`_error` 这些 TS 语法细节。

如果还有读不懂的,回到对应章节查;这套教程的章节号就是你查「这是哪个语法」的索引。

## 小结

- `ReactLoopAgent` 用 `Phase` 判别联合做状态机,驱动 turn/step 两层循环。
- `session.append` 记录一切;`deriveMessages()` 从日志推导模型历史,支撑「模型可见⟺已记录」。
- 流式模型输出用 `for await`,每块都记日志,保证回放保真。

## 思考题

1. `Phase` 的 `idle`/`maintenance`/`running` 三种状态,各带哪些不同字段?为什么 `idle` 不需要 `abort` 而另外两个需要?
2. 「模型可见⟺已记录」这条铁律,在 `step()` 的哪一行体现?如果违反它,会发生什么(提示:回放/恢复会怎样)?
3. `catch (_error)` 里参数名为什么加下划线?这对应哪个 tsconfig 开关(第 2 章)?

---

上一章:[第 24 章 · 核心包地图与能力接缝](24-package-map-seams.md)
下一章:[第 26 章 · 工具执行管道与收尾](26-tool-pipeline.md)
