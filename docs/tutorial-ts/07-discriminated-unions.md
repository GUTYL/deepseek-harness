# 第 7 章 · 判别联合(discriminated union)与穷尽检查

## 本节要学的概念

- 判别联合(discriminated union):用共同的字面量「标签」字段区分联合的各成员。
- 穷尽检查(exhaustiveness checking)与 `never`:让编译器替你保证「所有情况都处理到了」。
- 项目的两种联合风格:封闭(closed)vs 可扩展(merge-extensible)。

## 什么是判别联合

第 6 章你已经见过 `AgentCancelCause`。再看一遍它:

```ts
export type AgentCancelCause =
  | { readonly kind: 'user' }
  | { readonly kind: 'parent' }
  | { readonly kind: 'hook'; readonly reason: string }
  | { readonly kind: 'disposed' }
```

注意一个关键点:**每个成员都有一个共同的 `kind` 字段,而且它的值是一个互不相同的字面量**(`'user'`/`'parent'`/`'hook'`/`'disposed'`)。

这个 `kind` 字段就是「**判别标签(discriminant)**」。因为它每个成员的值都唯一,所以「看 `kind` 是什么」就能唯一确定「这个值是哪个成员」。由这种方式组织起来的联合,叫**判别联合(discriminated union)**。

## 判别联合的意义:switch 自动收窄

判别联合最优雅的用法是 `switch`:

```ts
function describe(cause: AgentCancelCause): string {
  switch (cause.kind) {
    case 'user': return '用户取消'
    case 'parent': return '父级取消'
    case 'hook': return 'hook 取消: ' + cause.reason
    case 'disposed': return '已销毁'
  }
}
```

每进一个 `case`,TS 自动把 `cause` 收窄成对应的成员——所以 `case 'hook'` 里才能安全访问 `cause.reason`(它只存在于 hook 成员)。

## 穷尽检查:让编译器保证「没漏 case」

上面这个 `describe` 有个隐患:如果将来有人给 `AgentCancelCause` **新增了第五种取消原因**,而忘了在 `describe` 里加对应的 `case`,程序会怎样?

答案是 `switch` 结束后 `describe` 没有一个 `return`,返回 `undefined`——一个 bug **静默地**溜进来,而且不报错。

**穷尽检查(exhaustiveness checking)** 就是为了抓这种 bug。辅助手段是一个叫 `assertNever` 的小工具(本项目到处用它):

```ts
function assertNever(value: never, message: string): never {
  throw new Error(message)
}
```

关键在参数类型 **`never`**(第 18 章细讲)。它的作用这里就能看懂:配合判别联合,逼编译器检查你有没有漏 case:

```ts
function describe(cause: AgentCancelCause): string {
  switch (cause.kind) {
    case 'user': return '用户取消'
    case 'parent': return '父级取消'
    case 'hook': return 'hook 取消: ' + cause.reason
    case 'disposed': return '已销毁'
    default:
      return assertNever(cause, '未知取消原因')
  }
}
```

现在看 `default` 分支里发生了什么推理:

- `switch` 已经分支了全部四种 `kind`,所以走到 `default` 时,`cause` 的 `kind` 从类型看**已经是 `never`**(「没有任何可能是这四种之外的值」)。
- 于是 `assertNever(cause, ...)` 里,`cause` 能赋给 `never` 形参,类型检查通过。

**关键在你漏了一个 case 的那一刻**:假设有人新增了 `{ kind: 'timeout' }` 这个成员,却没加对应 `case`。那么走到 `default` 时,`cause` 的 `kind` 就不是 `never` 了(它可能是 `'timeout'`),`assertNever(cause, ...)` 会**编译报错**——「类型 `{ kind: 'timeout' }` 不能赋给 `never`」。

**这就是穷尽检查的威力:你忘了处理新情况,编译器在编译期就替你喊出来,而不是让一个返回 `undefined` 的 bug 溜进运行期。**

> 项目规则明确写着:「开关联合以 `assertNever` 结束」(closed unions end in assertNever)。你在源码里看到 `assertNever`,就知道:**这是一个需要穷尽处理的判别联合,漏一个 case 编译器会拦下。**

## 两种风格:封闭 vs 可扩展

判别联合在本项目有两种组织方式,分清它们很重要:

**1. 封闭联合(closed union)** ——直接 `type X = A | B | C`:

```ts
export type AgentCancelCause =
  | { readonly kind: 'user' }
  | { readonly kind: 'parent' }
  | { readonly kind: 'hook'; readonly reason: string }
  | { readonly kind: 'disposed' }
```

作者把成员写死在类型里,「今后就这三种」。配合 `assertNever` 做穷尽。

**2. 可扩展联合(merge-extensible union)** ——用一个「Map」接口,再用 `keyof` 推导出联合(第 10 章正式讲 `keyof`):

```ts
export interface TurnEndReasonMap {
  completed: { kind: 'completed' }
  aborted: { kind: 'aborted'; reason: TurnEndCancelCause }
  blocked: { kind: 'blocked' }
  error: { kind: 'error'; error: LlmFailure }
  'max-tokens': { kind: 'max-tokens' }
  interrupted: { kind: 'interrupted' }
}

export type TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]
```

风格 1 是「就在这里定死」;风格 2 是「**别的插件可以通过声明合并往 `TurnEndReasonMap` 里加新键,`TurnEndReason` 就自动变宽**」。这是本项目「一切皆插件」哲学在类型层面的体现——插件不但能加运行时功能,还能**扩展类型词汇表**。

它们各有对应的处理纪律(项目规则的原话):

- **封闭联合**:`switch` 以 `assertNever` 收尾(穷尽)。
- **可扩展联合**:`switch` 要**落到一个文档化的 `default`**(因为你不能穷尽——别人随时可能加新成员)。

你现在读代码时,遇到 `switch`,可以问一句:「这是一个封闭联合(应该有 `assertNever`)还是可扩展联合(应该有 `default`)」?

## 你现在能自己读了

打开 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts),读这两个联合(它们都是「可扩展」风格):

```ts
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
export type ContentBlockType = keyof ContentBlockMap
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

现在你能读懂它的**意图**:`ContentBlock` 是「文本块 / 推理块 / 图片块 / 工具调用块 / 工具结果块」五种之一,靠 `type` 字段(每个成员的第一个字段,见第 5 章)判别。虽然 `keyof` 和 `[ContentBlockType]` 的精确含义要等第 10 章,但「这是用 Map + 索引类型构造的可扩展判别联合」这个结论你已经能说出来了。

同文件里的 `FinishReasonMap` / `FinishReason` 是同一模式,模型为什么停止(stop / tool-calls / max-tokens / aborted / error)就靠它表示。

## 小结

- 判别联合 = 各成员共享一个值唯一的 `kind` 字面量标签;`switch` 自动收窄。
- 穷尽检查:配合 `never` 形参的 `assertNever`,漏掉新 case 会在编译期报错。
- 封闭联合用 `assertNever` 穷尽;可扩展联合(Map + `keyof`)用文档化 `default`。

## 思考题

1. 判别联合的「判别标签」为什么必须是**互不相同的字面量**?如果两个成员都是 `kind: 'x'`,收窄还能唯一吗?
2. 在 `describe(cause)` 的 `switch` 里,如果漏掉 `case 'disposed'`,`assertNever(cause, ...)` 在哪一行、报什么错?用自己的话解释推理过程。
3. 「可扩展联合」为什么不能靠 `assertNever` 做穷尽?它和「一切皆插件」的哲学是什么关系?

---

上一章:[第 6 章 · 联合类型与收窄](06-unions-and-narrowing.md)
下一章:[第 8 章 · 类型别名、数组与元组](08-type-aliases-arrays-tuples.md)
