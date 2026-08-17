# 第 13 章 · 模板字面量类型

## 本节要学的概念

- 模板字面量类型(template literal type):把字符串模板语法用在「类型」上。
- 用它组合、约束、推导出字符串联合类型。

## 从「字符串模板」到「字符串类型」

JS 里你已经会写模板字符串:

```ts
const greeting = `Hello, ${name}`
```

TS 把这个语法带进了**类型世界**:模板字面量也能当类型,`${}` 里放的可以是类型变量:

```ts
type Greet<T extends string> = `Hello, ${T}`

type A = Greet<'DeepSeek'>   // "Hello, DeepSeek"
type B = Greet<'world'>      // "Hello, world"
```

`Greet<'DeepSeek'>` 的结果是字面量类型 `"Hello, DeepSeek"` —— 编译器真的把字符串「拼接」成了一个精确的字面量类型。

## 联合类型在模板里会「分配」

把联合类型放进 `${}`,会展开成所有组合:

```ts
type Size = 'small' | 'large'
type Color = 'red' | 'blue'

type Shirt = `${Size}-${Color}`
// "small-red" | "small-blue" | "large-red" | "large-blue"
```

`${Size}-${Color}` 是「Size 的每个值 × Color 的每个值」的笛卡尔积。这也是模板字面量类型最强的地方:**用更小的联合「生成」更大的、有规律的字面量联合**。

## 怎么「解析」而不是「生成」:条件类型 + infer

模板字面量类型反向也能用——在条件类型的 `infer` 里「匹配并拆出」字符串的一部分:

```ts
type SplitEvent<T extends string> =
  T extends `${infer Namespace}/${infer Action}` ? { ns: Namespace; action: Action } : never

type A = SplitEvent<'stats/report'>   // { ns: "stats"; action: "report" }
```

`T extends `${infer Namespace}/${infer Action}`` 是一个条件类型(第 12 章),它「看 T 能不能拆成 `某串/某串` 的形式」,能就分别捕获两侧。这就是模板字面量类型在「模式匹配」侧的用法。

## 项目里的真实用例

### 1. 事件的 `namespace/action` 命名

你在本教程反复见到的那些事件名,其实是模板字面量类型在「约定」层面的体现。比如 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts):

```ts
declare module '@deepseek-ai/cordis' {
  interface Events {
    'llm/adapters-updated'(): void
  }
}
```

`'llm/adapters-updated'` 这个键,遵循「`namespace/action`」的命名约定——`llm` 是命名空间(这段事件的归属领域),`adapters-updated` 是动作。这个约定让扁平的事件名空间可读、可归属。事件名本身是字符串字面量类型,而「从事件名解析出命名空间/动作」这类需求,正是模板字面量类型 + `infer` 能表达的(题见思考题 2)。

### 2. 包名到路径的 wildcard 映射

第 2 章让你看过的 [`tsconfig.base.json`](../../tsconfig.base.json) 里,paths 有这样的条目:

```json
"@deepseek-ai/dsh-*": ["./packages/core/*/src", ...]
```

这里的 `*` 是 tsconfig 路径映射的**通配符**,不是 TS 类型语法,但它体现了同一思想:「用模式匹配一串包名,统一映射到磁盘路径」。它和模板字面量类型「用模式描述一串字符串」是同一个心智模型。

### 3. 由 `keyof` 生成的、带前缀的事件/工具名

第 12、15 章你会看到,项目用 `keyof` 从 `XxxMap` 里取出键名联合(那些键天然是 `'a/b'` 这类字符串字面量)。例如 [`packages/core/tools/src/ts-types.ts`](../../packages/core/tools/src/ts-types.ts) 生成了这样一行(它是**生成的 TypeScript 代码文本**,不是运行时逻辑):

```ts
'type ToolName = keyof ToolOutputMap'
```

`keyof ToolOutputMap` 的结果是一个「工具名」的字符串字面量联合——这些工具名正是靠字符串字面量(有些还带前缀)组织起来的。

## 你现在能自己读了

回到 [`dispatch.ts`](../../packages/core/agent/src/dispatch.ts),看这一行:

```ts
export type AgentSubjectEvent = {
  [K in keyof Events]: Events[K] extends (this: Scoped<Agent>, ...args: infer P) => unknown
    ? P extends [infer Payload, ...unknown[]]
      ? Payload extends { agent: Agent } ? K : never
      : never
    : never
}[keyof Events]
```

你现在还不需要完全读懂(第 15 章全文精读)。但你能认出其中几个「熟面孔」:

- `K in keyof Events` 是映射类型(第 11 章)。
- `extends ... ? ... : never` 是条件类型(第 12 章)。
- 结果是 `[keyof Events]` 索引访问——从这个映射类型里取出「所有存活键」的联合(第 10 章)。

而 `K` 和最终结果 `AgentSubjectEvent`,都是**字符串字面量类型的联合**(事件名)。也就是说,这个类型在做的,是「**从所有事件名里,筛选出『满足某个条件』的那些事件名**」——用类型运算对字符串联合进行「过滤」。这正是模板字面量类型、条件类型、映射类型配合发挥的场景。

## 小结

- 模板字面量类型把字符串语法带进类型:`\`Hello, ${T}\``。
- 联合类型放进模板会笛卡尔展开;配合 `infer` 能反向解析/匹配字符串。
- 项目用它组织事件名的 `namespace/action` 约定,并用类型运算筛选字符串联合。

## 思考题

1. `` `${Size}-${Color}` `` 如果 Size 有 2 个值、Color 有 3 个值,结果有几个成员?为什么叫「笛卡尔积」?
2. 既然事件名遵循 `namespace/action`,你能写一个类型,利用模板字面量 + `infer`,从事件名里拆出 `namespace` 吗?(不要求跑通,描述思路即可。)
3. `AgentSubjectEvent` 用 `[keyof Events]` 收尾得到「事件名联合」,为什么不能直接写 `keyof Events`?二者结果一样吗?(提示:中间映射类型把不满足条件的键映射成了 `never`。)

---

上一章:[第 12 章 · 条件类型与 infer](12-conditional-types-infer.md)
下一章:[第 14 章 · 声明合并与模块扩展](14-declaration-merging.md)
