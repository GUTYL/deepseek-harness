# 第 11 章 · 映射类型与内置工具类型

## 本节要学的概念

- 映射类型(mapped type):`{ [K in ...]: ... }` 从一个集合「批量生成」属性。
- 常见的几个内置工具类型(utility types):`Partial` / `Pick` / `Omit` / `Record` / `ReturnType` / `Parameters` / `Awaited`。

## 映射类型:批量变换一个类型

第 10 章学了「取键」「取成员」。**映射类型(mapped type)** 是把这两者合成一个「变换」:

```ts
interface User {
  name: string
  age: number
}

type ReadonlyUser = {
  readonly [K in keyof User]: User[K]
}
// 结果 = { readonly name: string; readonly age: number }
```

拆解 `{ readonly [K in keyof User]: User[K] }`:

- `[K in keyof User]` — `K` 遍历 `User` 的每个键(`"name"`、`"age"`)。
- `: User[K]` — 每个键对应的值类型,用索引访问(第 10 章)取出来。
- 前面加 `readonly` — 给每个生成的属性统一加修饰符。

结果就是「把 User 的每个属性都变成只读」的新类型。**映射类型 = 遍历键 + 对每个键产出属性**,本质是一次「对类型结构的批量变换」。

## 项目真实用例:`Simplify<T>`

第 9 章让你记过这个名字,现在读它的定义。来自 [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts):

```ts
type Simplify<T> = { [K in keyof T]: T[K] } & {}
```

这是一个不折不扣的映射类型:

- `{ [K in keyof T]: T[K] }` — 遍历 T 的每个键,产出「键同名、值类型相同」的属性。
- `& {}` — 交叉上一个空对象类型。

效果是「**把 T 的属性平铺展开成一个干净的对象类型**」。为什么要这么做?因为 TS 有时会把一个复杂映射类型显示成一坨难读的 `{ x: ... } & { y: ... }` 或保留一堆中间别名,`Simplify<T>` 让它「压平」成一眼能读的样子。这是**纯类型层面的可读性技巧**——运行时无事发生(第 2 章的「类型编译后消失」)。

## 内置工具类型:TS 已经帮你写好的映射类型

上面这些「批量变换」太常用了,TS 把它们做成了内置的**工具类型(utility types)**。项目里出现的几个,逐一认识:

### `Partial<T>`:全变可选

```ts
type PartialUser = Partial<User>   // { name?: string; age?: number }
```

### `Pick<T, K>`:挑几个键

```ts
type NameOnly = Pick<User, 'name'>   // { name: string }
```

### `Omit<T, K>`:删掉几个键

```ts
type NoAge = Omit<User, 'age'>   // { name: string }
```

`Omit` 在项目里是常客。我们在第 15 章会精读的 `dispatch.ts` 里就有:

```ts
type PayloadRest<K extends AgentSubjectEvent> = Omit<PayloadOf<K> & object, 'agent'>
```

你能读懂它的意图(细节到第 15 章):「取 `PayloadOf<K>` 这个对象类型,**删掉 `agent` 键**」——因为 `agent` 由派发器自动注入,调用者不该再传。

### `Record<K, V>`:造一个「键→值」映射

```ts
type Env = Record<string, string | undefined>
// { [k: string]: string | undefined }
```

第 4 章 `resolveDshHome` 的 `env: Record<string, string | undefined>` 就是这个:一个「任意字符串键 → (字符串或 undefined)值」的映射。

### `ReturnType<F>` 与 `Parameters<F>`:拆函数类型

```ts
function f(a: string, b: number): boolean { return true }

type R = ReturnType<typeof f>    // boolean
type P = Parameters<typeof f>    // [string, number](元组)
```

第 10 章的 `ReturnType<typeof setTimeout>` 用的就是它。

### `Awaited<T>`:解开 Promise

```ts
type R = Awaited<Promise<string>>   // string
```

第 15 章的 `dispatch.ts` 里 `serial(...): Promise<Awaited<Return<Events[K]>>>` 会用上它。

> **工具类型不是魔法**,它们都是「映射类型 + 类型运算」组合出来的(源码在 TS 标准库里,`keyof`/`in`/条件类型搭成)。你学了前几章,已经具备读懂它们实现的能力。这里只需记住「这些现成工具存在、各自做什么」,需要时查即可。

## 你现在能自己读了

回到 [`packages/core/agent/src/dispatch.ts`](../../packages/core/agent/src/dispatch.ts),只看这一行(它出自第 15 章要精读的文件):

```ts
type PayloadRest<K extends AgentSubjectEvent> = Omit<PayloadOf<K> & object, 'agent'>
```

现在你能拆出几个零件:

- 这是一个泛型(第 9 章),`K` 约束到 `AgentSubjectEvent`。
- `PayloadOf<K> & object` 是交叉类型(第 9 章)。
- 外层 `Omit<..., 'agent'>` 是内置工具类型:删掉 `agent` 键。

你还没法完全理解 `PayloadOf`、`AgentSubjectEvent` 是什么(`infer` 条件类型的产物,第 12 章),但**你已经能读出「这是在删掉 agent 键」这层意图**——这就是逐章积累的价值:复杂类型拆到底,都是你学过的零件。

## 小结

- 映射类型 `{ [K in 集合]: 变换 }` 批量产属性,是「类型变换」的基础。
- 内置工具类型(`Partial`/`Pick`/`Omit`/`Record`/`ReturnType`/`Parameters`/`Awaited`)都是常用的映射组合,记住用途即可。
- `Simplify<T> = { [K in keyof T]: T[K] } & {}` 用于把复杂类型「压平」,纯类型层技巧。

## 思考题

1. 用自己的话描述 `{ readonly [K in keyof User]: User[K] }` 的执行过程:K 依次是什么?每个属性怎么来?
2. `Pick` 和 `Omit` 是「互补」的吗?`Omit<User, 'age'>` 能不能用 `Pick` 表达出来?
3. 为什么项目要写一个 `Simplify<T>`,而不是直接用系统的 `{ [K in keyof T]: T[K] }`?提示:结合「类型显示的易读性」和「复用」。

---

上一章:[第 10 章 · 类型运算](10-keyof-typeof-indexed.md)
下一章:[第 12 章 · 条件类型与 infer](12-conditional-types-infer.md)
