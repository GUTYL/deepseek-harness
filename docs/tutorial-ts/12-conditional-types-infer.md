# 第 12 章 · 条件类型与 `infer`

## 本节要学的概念

- 条件类型(conditional type):`T extends U ? X : Y`,类型层面「如果…就…」。
- `infer`:在条件类型里「捕获」某个子类型,给它起名。
- 二者结合是本项目最深类型代码的原料。

## 条件类型:类型里的 if

TS 的类型也能写「如果 A 就 B,否则 C」,语法借用了三元表达式 `?:`:

```ts
type IsString<T> = T extends string ? true : false

type A = IsString<string>   // true
type B = IsString<number>   // false
```

`T extends string ? true : false` 读作:「如果 T 可以赋给 string,类型就是 `true`;否则是 `false`」。注意 `true`/`false` 在这里是**字面量类型**(第 3 章),不是布尔值——这是类型层面的判断结果。

前面几章你其实已经见过 `extends` 的两种用法,这里有必要区分清楚:

- **泛型约束**(第 9 章):`function f<T extends HasLength>()` —— 要求 T 是某类型,否则报错。
- **条件类型的判断**:`T extends string ? X : Y` —— 不报错,而是根据是否匹配,**选择** X 或 Y。

同一个 `extends`,前者「限制」,后者「分支」。别混。

## `infer`:在条件里抓一个类型出来

条件类型最强大的用法是 `infer`——在做类型匹配的同时,把「匹配到的那部分」捕获成一个新类型变量:

```ts
type Unwrap<T> = T extends Promise<infer R> ? R : never

type A = Unwrap<Promise<string>>   // string
type B = Unwrap<string>            // never(T 不是 Promise)
```

`T extends Promise<infer R>` 的意思是:「T 能不能匹配成 `Promise<某个类型>`?如果能,把『某个类型』捕获为 `R`」。于是 `Unwrap<Promise<string>>` 抓到 `R = string`,整个类型就 `= string`。

`infer` 只能出现在条件类型的 `extends` 子句里,且只能捕获**一个类型位置**上的东西。它的价值:**从复杂类型里「掏」出某个嵌套的子类型**。

## 项目真实用例:拆函数类型

[`packages/core/agent/src/dispatch.ts`](../../packages/core/agent/src/dispatch.ts) 开头就有两条,你现在能读懂了:

```ts
type Params<F> = F extends (...args: infer P) => unknown ? P : never
type Return<F> = F extends (...args: never[]) => infer R ? R : never
```

拆 `Params<F>`:

- 条件:`F extends (...args: infer P) => unknown` —— 「F 能不能匹配成『一个函数类型,参数是 P,返回 unknown』?」
- `infer P` 捕获「参数元组」到 `P`。
- 若匹配,类型结果是 `P`(这个函数的参数元组);否则 `never`。

所以 `Params<(a: string, b: number) => boolean>` = `[string, number]`(参数元组)。

拆 `Return<F>` 同理,`infer R` 捕获返回值:`Return<typeof f>` = `boolean`。这其实就是第 11 章内置 `ReturnType`/`Parameters` 的手写版——**你现在能读懂系统工具类型的实现思路了**。

## 更复杂的真实用例:递归地从 JSON schema 里推类型

这是项目里 `infer` 用得更深的地方,来自 [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts)(节选,略去无关部分):

```ts
type InferObject<S extends { additionalProperties: boolean }, Depth extends unknown[]> =
  S extends { properties: infer P }
    ? S['additionalProperties'] extends true
      ? // 有 properties 且开放:递归推断属性类型,再交叉一个索引签名
        Simplify<{ [K in StringKeyOf<S>]: InferProperties<S, Depth> } & ...>
      : // 有 properties 且封闭:只推断这些属性
        Simplify<{ [K in StringKeyOf<S>]: InferProperties<S, Depth> }>
    : // 无 properties:...
      ...
```

不要求你现在逐行看懂(它还用到了第 11、13、9 章的一堆组合)。但你能认出几个关键模式:

- `S extends { properties: infer P }` —— 用 `infer` 从 schema 里抓出 `properties` 字段的类型。
- `S['additionalProperties'] extends true ? ... : ...` —— 条件类型做分支(开放对象 vs 封闭对象)。
- `Depth extends unknown[]` —— 一个「递归深度计数器」(用数组长度计数,是 TS 里做有限递归的常见手法)。

它整体的目标:**根据工具声明的 JSON schema,自动推算出「工具参数实际是哪种 TS 类型」**。也就是让「参数结构描述」和「参数类型」由同一份 schema 推导,不会各自手写然后对不上。这正是「类型运算」解决真实问题的样子——你不必现在就写这种代码,但读到它时,应该能认出来「哦,这是一段 infer + 条件类型 + 递归的推导」,而不是一片看不懂的天书。

## 你现在能自己读了

回到 [`dispatch.ts`](../../packages/core/agent/src/dispatch.ts),再往下看到这两个(第 15 章会全文精读,这里先预热):

```ts
type PayloadOf<K extends AgentSubjectEvent> = Params<Events[K]> extends [infer Payload, ...unknown[]] ? Payload : never

type Tail<K extends AgentSubjectEvent> = Params<Events[K]> extends [unknown, ...infer R] ? R : never
```

现在你能认出:

- 它们都用了 `infer`。
- `Params<Events[K]>` 先取出事件的参数元组(用第 12 章刚学的 `Params`)。
- `extends [infer Payload, ...unknown[]]` 用**元组模式**(第 8 章)匹配「第一个元素 + 其余」,把**第一个元素**捕获为 `Payload`。
- `Tail` 则用 `[unknown, ...infer R]` 抓「**除了第一个以外的剩余部分**」(rest 展开捕获取 `R`)。

所以 `PayloadOf` 是「事件参数的第一个对象」,`Tail` 是「事件参数去掉第一个之后的剩余」——`infer` 和元组模式一结合,就能对参数列表做「拆头 / 取尾」。这就是本项目把事件系统类型化的核心手法。

## 小结

- 条件类型是「类型层面的 if」:`T extends U ? X : Y`。
- `infer` 在条件类型的 `extends` 子句里捕获子类型并命名。
- `infer` + 元组模式 = 对参数列表做「拆头/取尾」;`infer` + 递归 = 从结构推导类型。

## 思考题

1. `T extends Promise<infer R> ? R : never` 中,如果 `T = Promise<Promise<string>>`,结果是什么?如果是 `T = number[]` 呢?
2. `Params<F>` 用了 `infer P`,而 `Return<F>` 用了 `infer R`;为什么 `Return` 的参数位写的是 `never[]` 而不是 `infer P`?都抓会怎样?
3. 为什么 `Depth extends unknown[]` 这种「用数组长度当计数器」的手法,能实现「有限递归」?说说你的理解。

---

上一章:[第 11 章 · 映射类型与内置工具类型](11-mapped-and-utility-types.md)
下一章:[第 13 章 · 模板字面量类型](13-template-literal-types.md)
