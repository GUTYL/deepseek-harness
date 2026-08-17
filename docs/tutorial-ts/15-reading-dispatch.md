# 第 15 章 · 高级综合:完整读懂 `dispatch.ts`

## 本节要学的概念

- 把前 14 章所有知识串起来,逐行读懂一个真实的高阶类型文件。
- 训练「读类型代码」的方法:先认零件,再看整体意图。

## 为什么选这个文件

本教程最大的「压力测试」就是 [`packages/core/agent/src/dispatch.ts`](../../packages/core/agent/src/dispatch.ts)。它全文 176 行,把泛型、条件类型、`infer`、映射类型、`keyof`、模板字面量、内置工具类型、名义类型这些**全部**用上了,而且不是为了炫技——它要解决一个非常具体的问题。

先讲清**它要解决的问题**,再逐段读。否则会迷失在语法里。

## 先理解问题:派发器为什么需要这些类型

回忆第 14 章:项目的事件系统里,事件是通过 `declare module` 合并进一个 `Events` 接口的,每个事件是一个方法签名,例如:

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'agent/pre-step'(payload: { agent: Agent; messages: ... }, next: () => Promise<...>): Promise<...>
  }
}
```

而 agent 相关的事件有个共同特点:它们的**第一个参数**是这个 payload 对象,而且 payload 里总带一个 `agent` 字段(「这次事件是关于哪个 agent 的」)。

现在 agent 想要一个**更好用的派发器**:调用者只需写 `emit('agent/status', { status: ... })`,派发器自动做两件事:

1. **自动注入 `agent`**:调用者不用在 payload 里手写 `agent`,派发器帮你塞进去(防止「payload 里的 agent」和「派发作用域里的 agent」不一致)。
2. **自动按作用域派发**:事件派发到「这个 agent 自己的作用域」里。

难点在**类型**:要让 `emit('agent/status', { status })` 这个调用**有精确的类型检查**——`'agent/status'` 这个名字必须是真实存在的事件名,`{ status }` 这个 payload 必须正好是「该事件 payload 去掉 agent 后的剩余字段」,少写/多写都报错。

`dispatch.ts` 里的那些类型,全部是为了**从 `Events` 这个接口出发,自动推导出「派发器每个方法应该接受什么样的参数」**。理解了目标,下面逐段拆。

## 第一段:拆函数类型的工具

文件开头(第 16~18 行):

```ts
type Params<F> = F extends (...args: infer P) => unknown ? P : never
type Return<F> = F extends (...args: never[]) => infer R ? R : never
```

这两条你在第 12 章已经读懂了:`Params<F>` 取函数 `F` 的参数元组,`Return<F>` 取返回值。它们是后续所有推导的「螺丝刀」——因为 `Events` 里的每个事件就是一个函数签名,你要反复「拆它的参数」「取它的返回」。

## 第二段:筛选出「agent 主题」的事件名

第 28~34 行,这是本章最核心的一段,我们一行行来:

```ts
export type AgentSubjectEvent = {
  [K in keyof Events]: Events[K] extends (this: Scoped<Agent>, ...args: infer P) => unknown
    ? P extends [infer Payload, ...unknown[]]
      ? Payload extends { agent: Agent } ? K : never
      : never
    : never
}[keyof Events]
```

先用一句话概括意图:**从所有事件名里,筛选出「第一个参数是含 agent 的 payload、且 this 类型是 Scoped\<Agent\>」的那些事件名。**

逐步拆,注意每个 `extends` 是「条件判断」还是「约束」:

**外层是映射类型**(第 11 章):

```ts
{ [K in keyof Events]: 「对每个事件做判断」 }
```

`K` 依次是 Events 的每个键,也就是每个事件名。

**第一层判断**(第 29 行):

```ts
Events[K] extends (this: Scoped<Agent>, ...args: infer P) => unknown ? ... : never
```

`Events[K]` 是事件 K 的签名(比如 `'agent/pre-step'` 那个方法类型)。判断它「是否形如 `this: Scoped<Agent>, (...args) => unknown`」——即「这个事件的 `this` 是不是作用域载体 Scoped\<Agent\>」。同时 `infer P` 抓出参数元组。

> `this: X` 是 TS 声明「方法在什么 this 下被调用」的标注。这里用它来「识别 agent 主题事件」:agent 主题事件约定为「this 是 Scoped\<Agent\>」,其他事件不是。用代码注释的原话:这个 `this` 检查是为了把「碰巧 payload 里恰好带 agent 的普通事件」排除出去。

**第二层判断**(第 30 行):

```ts
P extends [infer Payload, ...unknown[]] ? ... : never
```

`P` 是参数元组。判断它「是否至少有一个元素」,并把**第一个元素**捕获为 `Payload`(`[infer Payload, ...unknown[]]` 是元组模式,第 12 章)。

**第三层判断**(第 31 行):

```ts
Payload extends { agent: Agent } ? K : never
```

判断捕获到的 `Payload` 是否「有 `agent: Agent` 字段」。是,则这个键映射成 `K`(事件名本身);否,则映射成 `never`。

**最后收尾**(第 34 行):

```ts
[keyof Events]
```

对上面这个映射类型做**索引访问**(第 10 章),把所有键位置上的值类型取联合。因为「不满足条件」的键被映射成了 `never`,而 `never` 在联合里会被**自动忽略**,所以最终结果就是「所有满足条件的键名」的联合——**即所有 agent 主题事件的名称**。

这就是为什么第 13 章思考题 3 说「不能直接写 `keyof Events`」:直接 `keyof Events` 是「全部事件名」,而经过映射 + 索引访问 + never 过滤,得到的是「筛选后的子集」。

> **小结这段手法**:用「映射类型把不合条件的键变成 never,再用索引访问取联合」来**过滤一个键名集合**。这是 TS 类型编程里极常用的「筛选」套路,记住它。

## 第三段:从事件名反推参数结构

第 37~47 行:

```ts
type PayloadOf<K extends AgentSubjectEvent> = Params<Events[K]> extends [infer Payload, ...unknown[]] ? Payload : never

type Tail<K extends AgentSubjectEvent> = Params<Events[K]> extends [unknown, ...infer R] ? R : never

type PayloadRest<K extends AgentSubjectEvent> = Omit<PayloadOf<K> & object, 'agent'>
```

现在这三条都读得动了:

- `PayloadOf<K>` — 给定一个 agent 事件名 K,取它参数的**第一个对象**(payload 本体)。
- `Tail<K>` — 取它参数**去掉第一个之后的剩余**(对 waterfall 事件就是那个 `next` 回调)。
- `PayloadRest<K>` — 取 payload,**删掉 `agent` 键**(第 11 章的 `Omit`),得到「调用者该传的那部分字段」。

`PayloadRest` 这个名字直接点明了意图:**派发器的调用者只传「payload 减去 agent」**——因为 agent 由派发器自动注入(见第五段)。

## 第四段:派发器接口

第 54~82 行:

```ts
export interface AgentEventDispatch {
  emit<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>): void
  serial<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>): Promise<Awaited<Return<Events[K]>>>
  waterfall<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>, ...rest: Tail<K>): Return<Events[K]>
}
```

三个方法,都是**泛型方法**(第 9 章),类型参数 `K` 约束到 `AgentSubjectEvent`(第 9 章的约束)。看它们怎么用前面的类型:

- 都是 `name: K` —— 名字必须是真实 agent 事件名。
- 都是 `payload: PayloadRest<K>` —— payload 类型精确到「这个事件去掉 agent 后的字段」。你调 `emit('agent/status', { status })` 时,TS 会根据 K 自动算出 `PayloadRest<'agent/status'>` 该长什么样,字段多一个少一个都报错。
- `serial` 返回 `Promise<Awaited<Return<...>>>` —— 而 `waterfall` 返回(非 Promise 的)`Return<...>`。这个差异对应第 22 章会讲的 Cordis 派发语义:`serial` 是异步的,`waterfall` 的返回类型就是事件签名的返回。`Awaited`(第 11 章)用来「拆掉一层 Promise」,让类型对齐。

**这就是整个文件要交付的东西**:一个「每个方法都精确类型化」的派发器接口,把 `Events` 里的信息,自动流转成每个方法的参数/返回类型。调用者享受到的是「像写普通函数一样安全」的体验,而这一切是**编译期从 Events 推导出来的**。

## 第五段:实现里「为什么要几处 cast」

第 107~148 行是运行时的实现(函数体),它调用了 Cordis 的 `ctx.events.dispatch`、`ctx.serial`、`ctx.waterfall`。你会发现里面有几处 `as`(类型断言):

```ts
const fused = <K extends AgentSubjectEvent>(payload: PayloadRest<K>): PayloadOf<K> =>
  ({ ...payload, agent } as PayloadOf<K>)
```

这里的 `{ ...payload, agent }` 在运行时就是「把 agent 塞进 payload 并展开」——而类型上,它需要从 `PayloadRest<K>`(少了 agent)变回 `PayloadOf<K>`(完整 payload)。这个「补上 agent」的变换,TS 无法从 `{ ...payload, agent }` 自动证明类型是 `PayloadOf<K>`,所以作者用了一个**受限的、保形的类型断言** `as PayloadOf<K>`。

代码注释把理由说得很克制也很清楚(第 112~117 行):

> 「派发器拥有注入 agent 的职责;调用者传的是 PayloadRest,所以融合后的记录就是声明的 payload。展开写在前面,这样即使某个 payload 碰巧也带了 agent 字段,也无法覆盖注入的 agent。」

你不需要现在深究每个 `as` 的正当性(那涉及第 16 章的「断言」纪律),只需理解**为什么这里「必须有断言」**:TS 的类型推导有边界,遇到「我(作者)比编译器更确定」的地方,用 `as` 显式越过,而不是靠 `any` 糊弄。这个区别是第 16、18 章要展开的。

## 本章的方法论:怎么读一段高阶类型

回看这一章,它不是要你「背下来」,而是给你一套**读类型代码的步骤**:

1. **先问「它要解决什么问题」** —— 派发器要精确类型化地注入 agent 并派发。
2. **认出零件** —— `keyof`(`[K in ...]`)、`infer`、`extends ? :`、`Omit`、`Awaited`、`never`… 这些你都学过,逐个对号。
3. **看懂数据流** —— 从 `Events` → 筛选事件名 → 拆参数 → 删 agent 键 → 构成方法签名。
4. **别被运行时 cast 吓到** —— 运行时实现和类型推导是两半,`as` 是作者在「类型推导边界」上显式表态。

以后你遇到任何看不懂的类型,都按这四步走,而不是盯着语法发呆。

## 你现在能自己读了

把 [`packages/core/agent/src/dispatch.ts`](../../packages/core/agent/src/dispatch.ts) 从头到尾通读一遍——这次你不会再觉得它是天书,而是一段「要素齐全、意图清晰」的类型代码。读完它,把 `AgentSubjectEvent` 那段(第 28~34 行)合上看,在心里默写出它的推理过程。

如果你能对着一个没读过的人,讲清楚「这段类型如何从 `Events` 筛选出 agent 主题事件名」,那你的 TS 类型水平已经能应付本项目绝大部分源码了。

## 小结

- 高阶类型是「零件」的组合:映射类型 + 条件类型 + infer + keyof + 工具类型。
- 「映射成 never + 索引访问取联合」是筛选键名集合的标准套路。
- 读类型代码四步:定问题 → 认零件 → 看数据流 → 别怕 cast。

## 思考题

1. 用自己的话重写 `AgentSubjectEvent` 的推理过程,重点说明三层 `extends` 各自在判断什么。
2. `PayloadRest` 为什么用 `Omit<PayloadOf<K> & object, 'agent'>`,而不是直接 `Omit<PayloadOf<K>, 'agent'>`?`& object` 在这里起什么作用?
3. 运行时实现里 `{ ...payload, agent } as PayloadOf<K>` 的 `as` 和「直接写 any」相比,差别在哪?为什么项目宁可写 `as` 也不落回 `any`?(提示:想想类型安全「漏」在哪里。)

---

上一章:[第 14 章 · 声明合并与模块扩展](14-declaration-merging.md)
下一章:[第 16 章 · 类型谓词与断言守卫](16-type-predicates-asserts.md)
