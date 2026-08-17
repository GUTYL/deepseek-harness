# 第 26 章 · 工具执行管道,与全书的收尾

## 本节要学的概念

- 读懂 `tool-calls.ts`:并行/独占调度、按序提交、取消补记录。
- 用一段真实代码,验收你学到的「并发 + 顺序 + 数据一致性」设计意识。
- 全书总结,和继续深入的路标。

## 工具箱:模型要「调工具」时怎么办

第 25 章里,`step()` 在拿到模型的回复后,如果模型「想调用工具」,就会进入 [`packages/core/agent-loop/src/tool-calls.ts`](../../packages/core/agent-loop/src/tool-calls.ts) 的 `executeToolCalls`。

它要解决一个真实世界的难题:模型可能一口气发出多个工具调用:

```text
1. 读文件 A
2. 改文件 A   ← 必须等 1 完成(有依赖)
3. 跑测试      ← 必须等 2 完成
```

有些调用有依赖(必须顺序),有些没有(可并行加速)。调度器要正确安排,同时处理并发、顺序、取消。

## 并行 vs 独占:按执行模式分组

核心逻辑(约第 84~100 行):

```ts
while (next < planned.length) {
  const first = planned[next]!
  const mode = ctx.tools.executionMode(first.exec).kind
  const group = mode === 'parallel' ? planned.slice(next) : [first]
  const outcome = await runGroup(ctx, turn, step, group, mode, signal, acceptContext)
  next += outcome.consumed
  // ...
}
```

逐点读:

- `ctx.tools.executionMode(first.exec).kind` —— 从工具注册表(第 24 章的 `ctx.tools`)查询「这个调用的执行模式」。
- `mode === 'parallel' ? planned.slice(next) : [first]` —— 三元表达式:并行的抓一批,独占的单独一道屏障。
- `planned[next]!` —— 第 3/16 章的非空断言,数组索引访问后作者担保非空(第 3 章 `noUncheckedIndexedAccess`)。

所以「执行模式」分两类(注释也讲明了):

- **exclusive(独占)**:建立屏障,必须等前面全完成。
- **parallel(并行)**:可与其他并行调用一起跑,但有界(`maxParallelToolCalls`,见 `runGroup` 里的 `ctx.agentLoop.config`)。

## 两个「严谨」的设计细节

### 细节一:按模型顺序提交结果

工具可以并发跑,但**结果必须按模型发出的顺序**写回日志。`runGroup` 里的 `commitReady`(约第 146~160 行)只推进「连续的槽位」:

```ts
const commitReady = async (): Promise<void> => {
  while (committed < group.length) {
    const slot = slots[committed]
    if (slot === undefined) break        // 后面的还没跑完,停下
    // ... 提交这个结果,committed++
  }
}
```

`committed` 只前进「连续已完成」的槽位——这是「**产出与顺序解耦**」的经典写法:并发是并发的,但落盘顺序是确定的。

### 细节二:取消也要补记录

中途取消时,还没开始跑的工具调用,要**补写一个合成错误结果**(第 249~259 行的 `appendSkippedToolCall`):

```ts
function appendSkippedToolCall(session, turn, step, block) {
  const callSeq = appendToolCall(session, turn, step, block)
  appendToolResult(session, turn, step, block, {
    content: [{ type: 'text', text: 'Error: tool call aborted before dispatch' }],
    isError: true,
    // ...
  }, callSeq)
}
```

为什么?回到第 25 章的铁律——**模型可见 ⟺ 已记录**。如果日志里有一条 `tool/call` 却没有对应的 `tool/result`,回放(模型历史推导)就会断裂。所以即使被取消,也要让「调用了没结果」这件事在日志里自我闭合。

> 这就是「设计哲学落到具体代码」的样子:第 25 章的「日志是唯一真相源」,在取消路径上逼出了「补记录」这种看起来反直觉、实则必要的代码。

## 你已读完整个项目

到这里,你已经把项目最硬核的一条主链读通了。回顾一下学过的完整闭环:

```text
用户消息 → Inbox(第 25 章)
  → 唤醒主循环,Phase 切到 running
  → turn() 开启回合,append('turn/start')
  → preStep:走 agent/pre-step waterfall(第 22/25 章)
  → step():deriveMessages() 从日志推导历史(第 25 章)
  → ctx.llm.stream() 流式调用(第 23 章词汇表 + 第 24 章 seam)
  → 模型要调工具 → executeToolCalls 调度(第 26 章)
  → append('tool/call') + append('tool/result')
  → turn/end
```

而支撑这一切的,是你在第 1~19 章学的 TypeScript:判别联合、映射类型、条件类型、声明合并、品牌类型……每一样都在上面的代码里实实在在地出现过。

## 全书结语:三条最重要的收获

1. **类型是「诚实的描述」**:TS 每一处写法,都在更精确地描述「值是什么、会怎样」。读懂类型,就懂了代码的意图。
2. **结构类型 + 可扩展 = 插件化**:声明合并、Map+keyof、可扩展联合,映射的是「一切皆插件」的架构哲学。
3. **真相源与边界**:日志是唯一真相源(模型可见⟺已记录),不可信输入在边界用 `unknown` + 守卫收窄。这两条是项目最深的工程素养。

## 继续深入的路标

读完全书,你可以按需深入(详见 [`architecture.md`](../architecture.md) 和各文档):

1. **动手**:过一遍 [`cordis-tutorial/`](../cordis-tutorial/index.md)(7 章,可运行),把你「读」到的亲手写一遍。
2. **写插件**:从 [`user/develop/basic/index.md`](../user/develop/basic/index.md) 开始写第一个真正的 harness 插件。
3. **加工具**:照着 [`cookbook/adding-a-tool.md`](../cookbook/adding-a-tool.md) 走一遍「加一个模型可调用工具」的完整流程。
4. **查参考**:每个子系统的 API 见 [`subsystems/`](../subsystems/README.md);事件的「谁产生谁消费」见 [`event-producer-consumer.md`](../event-producer-consumer.md)。
5. **读懂更多包**:带着本教程的方法(定问题→认零件→看数据流→别怕 cast),去读 `subagent`、`workflow`、`session` 等更多包。

## 思考题(全书收尾)

1. 用一段话,从「用户发消息」讲到「工具结果写回日志」,把你理解的完整执行链讲一遍(不查资料)。
2. 「模型可见⟺已记录」在 `tool-calls.ts` 的取消路径上是如何体现的?为什么「补一个合成错误」是必要的?
3. 回顾全书,你觉得自己最难啃、现在最通透的 TS 特性是哪一个?它在项目里的哪段代码出现?

---

上一章:[第 25 章 · Agent 主循环:turn/step 状态机](25-agent-loop.md)

---

**全书终。** 回到[总目录](index.md)重温学习路径。
