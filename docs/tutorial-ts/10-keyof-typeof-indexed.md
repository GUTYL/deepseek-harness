# 第 10 章 · 类型运算:`keyof` / `typeof` / 索引访问

## 本节要学的概念

- `keyof`:取出一个类型的「所有键名」组成的联合。
- `typeof`(类型位置):从一个「值」取出它的「类型」。
- 索引访问 `T[K]`:取类型 `T` 里键 `K` 对应的那个成员类型。
- 三者组合,就是从「已有类型」里「算」出新类型——类型运算的基础。

## 从一个类型里「取键」:`keyof`

```ts
interface User {
  name: string
  age: number
}

type UserKey = keyof User   // "name" | "age"
```

`keyof User` 得到一个**联合类型** `"name" | "age"`——就是 User 的所有键名。这就是第 7 章 `ContentBlockType = keyof ContentBlockMap` 里用到的那个 `keyof`。

## 从「值」取「类型」:typeof(类型位置)

第 6 章学过 `typeof x === 'string'` 是**运行期**的类型判断。但 `typeof` 出现在 **`: 后面的类型位置**时,是另一回事——它从一个「值」推导出「类型」:

```ts
const fruit = { name: "apple", color: "red" }

type Fruit = typeof fruit
// Fruit = { name: string; color: string }
```

`typeof fruit` 不是「判断 fruit 是什么类型」,而是「把 fruit 这个值对应的类型**取出来**当类型用」。于是你可以在不重复手写那个对象类型的情况下,拿到它的精确类型。

`keyof` 和 `typeof` 经常连用:

```ts
type FruitKey = keyof typeof fruit   // "name" | "color"
```

先 `typeof fruit` 拿到类型 `{ name: string; color: string }`,再 `keyof` 取键,得到 `"name" | "color"`。

## 索引访问:T[K]

拿到一个类型后,可以用 `T[K]` 取「键 K 对应成员的类型」:

```ts
interface User { name: string; age: number }

type Name = User["name"]   // string
type All = User["name" | "age"]   // string | number(取多个键)
```

`User["name"]` 读作「User 里 name 成员的类型」。把 `K` 换成联合,就一次取多个成员,结果是各成员类型的联合。

## 三者组合:第 7 章的谜底

现在回头看第 7 章那句一直留到最后的话,你能完全读懂它了。打开 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts):

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

逐步拆开:

1. `ContentBlockType = keyof ContentBlockMap` —— 取 `ContentBlockMap` 的所有键,得到联合 `'text' | 'reasoning' | 'image' | 'tool-call' | 'tool-result'`。
2. `ContentBlock = ContentBlockMap[ContentBlockType]` —— 用这个联合去**索引访问** `ContentBlockMap`,得到各成员类型的联合 = `TextBlock | ReasoningBlock | ImageBlock | ToolCallBlock | ToolResultBlock`。

**这就是「可扩展联合」的完整配方**:一个 Map 接口(`ContentBlockMap`)+ 一个 `keyof`(得到所有键)+ 一个索引访问(得到所有成员的联合)。这三个运算符组合,把「一个接口」变成了「一个联合类型」。

它的「可扩展」体现在:任何插件只要 `declare module` 往 `ContentBlockMap` 里加一个新键(第 14 章),`keyof ContentBlockMap` 就自动多出一个键,`ContentBlock` 联合就自动多出一个成员——**给接口加键,联合类型自动变宽,所有 switch 它的地方都会收到「你该处理新情况了」的信号**。

## 项目的其他真实用法

`typeof`(类型位置)还常见于「引用一个现成值/模块的类型而不重复」:

```ts
// 取自 packages/util/timeout/src/index.ts 的真实写法
let timer: ReturnType<typeof setTimeout> | undefined
```

`ReturnType<...>` 是第 11 章的内置工具类型,这里先看 `typeof setTimeout`:`setTimeout` 是 JavaScript 内置的一个**函数值**,`typeof setTimeout` 取出它的**函数类型**,再交给 `ReturnType` 取返回类型。结论:`timer` 的类型是「setTimeout 返回值」——不写死成 `number`,因为不同环境的 `setTimeout` 返回类型可能不同,用 `typeof` 跟随真实定义。

再看 [`packages/core/tools/src/testing.ts`](../../packages/core/tools/src/testing.ts):

```ts
const CONTENT_VALUE_SCHEMA = { type: 'array', items: { type: 'json' } } as const

export type ContentToolFixtureOptions<S extends ParameterSchemaSpec> = Omit<
  DefineToolOptions<S, typeof CONTENT_VALUE_SCHEMA>,
  ...
>
```

这里的 `typeof CONTENT_VALUE_SCHEMA` 配合第 8 章的 `as const`:先 `as const` 把对象锁成精确的字面量类型,再 `typeof` 取出这个精确类型,传给 `DefineToolOptions<S, ...>`。一环扣一环:**`as const`(锁值)→ `typeof`(取类型)→ 泛型(填入)**。

## 你现在能自己读了

回到 [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts),找到第 7 章见过的：

```ts
export interface TurnEndReasonMap {
  completed: { kind: 'completed' }
  aborted: { kind: 'aborted'; reason: TurnEndCancelCause }
  // ...
}

export type TurnEndReason = TurnEndReasonMap[keyof TurnEndReasonMap]
```

现在你能完整解释 `TurnEndReasonMap[keyof TurnEndReasonMap]` 的每一个符号:先取所有键(联合),再索引访问(取各成员),得到「回合结束原因」的联合类型。和第 7 章「可扩展联合」的讲解对照,你就彻底懂了这套配方。

## 小结

- `keyof T` 取键名联合;`typeof 值`(类型位置)取值对应的类型;`T[K]` 索引访问取成员类型。
- 三者组合是「类型运算」的基础:从已有类型算出新类型。
- Map 接口 + `keyof` + 索引访问 = 可扩展联合,是「一切皆插件」在类型层的体现。

## 思考题

1. `keyof User` 的结果为什么是联合类型而不是数组?它和普通值的 `Object.keys()` 有何异同?
2. `typeof` 在「`typeof x === 'string'`」和「`type T = typeof x`」两个位置,分别是什么语义?为什么同一个关键字能做两件事?
3. 为什么 `ContentBlockType = keyof ContentBlockMap` 配合索引访问,能让「往 Map 加键 → 联合自动变宽」成立?请用「类型是惰性计算的引用」来解释。

---

上一章:[第 9 章 · 泛型](09-generics.md)
下一章:[第 11 章 · 映射类型与内置工具类型](11-mapped-and-utility-types.md)
