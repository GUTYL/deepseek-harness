# 第 9 章 · 泛型(generics):写会复用的代码

## 本节要学的概念

- 泛型(generics):用「类型参数」让一个函数/接口/类型能适配多种类型,同时保持类型安全。
- 泛型约束(constraint)`extends`。
- `const` 类型参数(`<const T>`)。

## 一个动机:不想每种类型都写一遍

看这两个函数,几乎一模一样,只是参数类型不同:

```ts
function firstString(list: string[]): string | undefined { return list[0] }
function firstNumber(list: number[]): number | undefined { return list[0] }
```

显然很蠢。但你又不想牺牲类型安全——如果只写一个 `first(list)` 不带类型,返回的就是 `any`,编译器就帮不上忙了。

**泛型(generics)** 解决的就是这个:写一份代码,「类型」让它成为一个「可以按需填入的占位符」。

```ts
function first<T>(list: T[]): T | undefined {
  return list[0]
}

first<string>(["a", "b"])   // T 被填成 string,返回 string | undefined
first<number>([1, 2])       // T 被填成 number,返回 number | undefined
first(["a", "b"])           // 也可以不写 <string>,TS 自动推断 T = string
```

`<T>` 就是**类型参数(type parameter)**。调用时,TS 根据传入的实参自动把它「填」成具体类型。一份实现,适配所有元素类型,同时返回类型依然精确——这就是泛型的价值。

## 泛型接口

`interface` 同样能带类型参数:

```ts
interface Box<T> {
  value: T
}

const stringBox: Box<string> = { value: "hi" }
const numBox: Box<number> = { value: 42 }
```

`Box<string>` 和 `Box<number>` 是两个不同的类型,由填入的 `T` 决定。

## 泛型约束:`T extends ...`

有时不能接受任意类型,而要「至少有某个能力的类型」:

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b
}

longest("ab", "cde")          // 合法,string 有 length
longest([1, 2], [3])          // 合法,数组有 length
longest(42, 7)                // 报错:number 没有 length
```

`<T extends { length: number }>` 是**约束(constraint)**:`T` 必须是「拥有 `length: number` 属性」的类型。约束之后,你在函数体内就能安全访问 `a.length` 了——因为 T 保证有 length。约束之外的类型(如 `number`)被拒之门外。

## 项目真实用例一:`Branded<B extends string>`

回到第 3、4 章反复出现的品牌类型,现在你能读懂它的完整定义。打开 [`packages/util/brand/src/index.ts`](../../packages/util/brand/src/index.ts):

```ts
declare const BRAND: unique symbol

export type Branded<B extends string> = string & { readonly [BRAND]: B }
```

逐步拆:

- `Branded<B extends string>` 是一个带约束的泛型:`B` 必须是字符串字面量类型(比如 `'CallId'`)。
- `string & { readonly [BRAND]: B }` 是一个**交叉类型(intersection,`&`)**:「既是 string,又有一个只读的、名为 BRAND 的属性」。

交叉类型 `A & B` 表示「同时是 A 又是 B」。所以 `Branded<'CallId'>` 是「`string`,但额外背着一个 `[BRAND]: 'CallId'` 的标记」。这就是第 17 章会展开的「品牌类型」——现在你只需知道**它是个带约束的泛型 + 交叉类型**。

## 项目真实用例二:`const` 类型参数

看 [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts) 里的函数签名:

```ts
export function defineTool<const S extends ParameterSchemaSpec, const O extends ValueSchemaSpec>(
  options: DefineToolOptions<S, O>,
): ...
```

两个新东西:

1. **多个类型参数**:`<const S ..., const O ...>`,两个独立的类型参数,分别约束到不同的界。
2. **`const` 类型参数**:类型参数名前面的 `const` 关键字。它的效果和第 8 章 `as const` 类似——**告诉 TS:调用者传入的字面量对象,请按「字面量类型」来推断,不要放宽成普通类型**。

为什么要 `<const S ...>`?因为 `defineTool` 接收一个描述工具参数结构的对象,作者想要「精确到字面量」的推断(好让后面的类型运算精确到 `required`/`enum` 这些细节)。`const` 类型参数省去了让每个调用者自己写 `as const` 的麻烦——它内建了「按 const 推断」。

> 你现在不必读懂 `ParameterSchemaSpec`、`ValueSchemaSpec` 是什么(它们是把 JSON schema 编码进类型的高阶类型,第 12 章会碰到 `infer` 那部分)。重点是**认出「泛型 + 约束 + const 类型参数」这个组合**,以及它想达成什么:「精确推断入参的结构」。

## 你现在能自己读了

1. 通读 [`packages/util/brand/src/index.ts`](../../packages/util/brand/src/index.ts) 全文(它只有 27 行),现在你应该能逐行读懂 `Branded<B extends string>` 的定义;最后一行 `string & { readonly [BRAND]: B }` 是「交叉类型」,你能说出「交叉」的两半分别是什么。

2. 在 [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts) 顶部附近,找到 `type Simplify<T> = { [K in keyof T]: T[K] } & {}`。`[K in keyof T]` 是什么你到第 10、11 章才学,但你能认出它**也是一个泛型**(`<T>`),以及末尾的 `& {}` 是交叉类型。先记住这个名字 `Simplify`,后面会再见到它。

## 小结

- 泛型用类型参数 `<T>` 让一份代码适配多种类型,且不丢类型安全。
- `<T extends 界>` 是约束:限制 T 必须具备某能力,并允许在函数体内使用该能力。
- `const` 类型参数让调用处按字面量精确推断,免去逐个 `as const`。

## 思考题

1. 泛型和「写死类型」相比,好处是什么?和「用 any」相比,好处又是什么?
2. `Branded<'CallId'>` 与普通 `string` 的区别在哪?为什么约束是 `<B extends string>` 而不是 `<B>`?
3. 一个泛型函数的类型参数,是调用时显式写 `first<string>(...)` 好,还是省略让 TS 推断好?各有什么取舍?

---

上一章:[第 8 章 · 类型别名、数组与元组](08-type-aliases-arrays-tuples.md)
下一章:[第 10 章 · 类型运算:keyof / typeof / 索引访问](10-keyof-typeof-indexed.md)
