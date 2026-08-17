# 第 3 章 · 变量与基础类型注解

## 本节要学的概念

- 基础类型(primitive types):`string` / `number` / `boolean` / `null` / `undefined`。
- 字面量类型(literal types)与 `const` 的推断(inference)。
- 类型推断:什么时候你能省略注解,什么时候必须写。

## 基础类型注解

```ts
const name: string = "DeepSeek"
const year: number = 2026
const done: boolean = false
const nothing: null = null
const missing: undefined = undefined
```

`:` 后面的 `string`、`number`、`boolean` 就是**基础类型**。规则简单:一个值属于哪种基础类型,就标哪个。

## 一个常见误区:不需要处处写注解

新手常以为「TS 就是给所有变量都手动标类型」。其实不然。看这个:

```ts
const name = "DeepSeek"   // TS 自动推断 name 是 string
```

TS 编译器看着右边的 `"DeepSeek"`,自己就知道 `name` 是字符串——这叫**类型推断(type inference)**。所以大部分时候,**你什么都不用写**,TS 自动帮你补上类型。

什么时候才需要手动写 `: string`?两条经验法则:

1. **右边的值让 TS 无法确定类型时**(比如空数组 `const list = []` 推断不出里面装什么)。
2. **你想「锁定」一个更窄的类型时**(下一节讲)。其余时候,交给推断即可。

## 字面量类型:这就是类型系统进阶的起点

看两个声明:

```ts
const a = "DeepSeek"   // 类型是什么?
let b = "DeepSeek"     // 类型是什么?
```

直觉都说「都是 `string`」。但 TS 更精确:

- `a` 用 `const` 声明,值永远不会变,TS 推断它的类型是 **`"DeepSeek"`** 这个**字面量类型(literal type)**——「只能是 DeepSeek 这唯一一个字符串」。
- `b` 用 `let` 声明,以后可以重新赋值成任何字符串,TS 推断它是 **`string`**。

所以 `const` 会得到「一个精确的字面量类型」,`let` 会得到「一个宽泛的基础类型」。字面量类型的写法就是**字面值本身**当类型名:

```ts
const a: "DeepSeek" = "DeepSeek"   // 合法:唯一允许的值就是 "DeepSeek"
const b: 42 = 42                   // 数字字面量类型:只能是 42
const c: true = true               // 布尔字面量类型
```

**为什么这个特性是「进阶的起点」?** 因为本项目的核心模式——判别联合(discriminated union,第 7 章)——就建立在这之上:用 `type: "text"` 这样的字面量,当作区分「这是哪种东西」的标签。先记住「字面量能当类型」这个事实,第 7 章会看到它大放异彩。

## 项目中一个真实的字面量推断:命名的 id 工厂

打开 [`packages/llm/llm/src/brand.ts`](../../packages/llm/llm/src/brand.ts),你会看到这样一组函数:

```ts
export type CallId = Branded<'CallId'>

export function CallId(id: string): CallId {
  return id as CallId
}
```

这里的 `<'CallId'>`(尖括号里的单引号字符串)就是一个**字符串字面量类型**——它描述的精确值是 `"CallId"` 这串文本。至于 `Branded<...>` 和 `as` 是干什么的,分别属于第 17 章和第 16 章,现在不用深究。

你现在能读懂的只是:类型系统允许「`'CallId'` 这串字面文本」被当成一个类型来用。这再次印证**字面量能当类型**。

## `noUncheckedIndexedAccess`:项目为什么那么多 `!`

第 2 章让你猜的开关,现在揭晓。看这段 JS 常犯的错:

```ts
const names: string[] = ["Ada", "Grace"]
const first = names[100]   // 数组只有 2 个元素,访问第 100 个
first.toUpperCase()        // first 实际是 undefined,运行到这里才崩
```

默认情况下,TS 会假装 `names[100]` 返回 `string`(骗自己「数组访问一定有值」)。而本项目开了:

```json
"noUncheckedIndexedAccess": true
```

打开后,TS 诚实多了:`names[100]` 的类型变成 `string | undefined`——「要么是字符串,要么是你要的东西不存在」。于是 `first.toUpperCase()` 会被编译器拦下:「`first` 可能是 `undefined`,你不能直接调方法」。

这就是为什么你在项目源码里,会看到很多**非空断言(non-null assertion)**符号 `!`:

```ts
const first = names[100]!
```

`!` 的意思是「我(作者)向你(编译器)保证:这里不可能是 `undefined`,别管了」。这是作者和编译器之间的一个「具结」。

> 这个「`!`」符号在第 16 章会和 `as`、`is` 一起归入「类型断言」系统详细讲。现在先眼熟:**它表达的是「作者对类型的额外承诺」,且这个承诺不会被编译器验证——用错了就是作者的责任**。你在项目里看到它,就知道「这里作者认为数组索引一定有值」。

## 你现在能自己读了

回到 [`packages/llm/llm/src/brand.ts`](../../packages/llm/llm/src/brand.ts),通读一遍。现在你应该能认出:

- 每个 `export type XxxId = Branded<'XxxId'>` 里的 `'XxxId'` 是字面量类型。
- 每个工厂函数 `function XxxId(id: string): XxxId` 的 `: string`(参数)和 `: XxxId`(返回值)是类型标注。

再打开 [`packages/core/agent-loop/src/constants.ts`](../../packages/core/agent-loop/src/constants.ts):

```ts
export const DEFAULT_MAX_PARALLEL_TOOL_CALLS = 10
```

因为这个值用 `const` 声明,TS 推断它的类型是精确的字面量类型 `10`,而不是宽泛的 `number`(这正是 `const` 推断的体现)。

## 小结

- 基础类型是 `string`/`number`/`boolean` 等;大部分类型靠推断,不必手写。
- `const` 得到字面量类型(如 `"DeepSeek"`、`10`),`let`/`var` 得到宽类型(如 `string`、`number`)。
- `noUncheckedIndexedAccess` 让数组索引访问返回 `T | undefined`,逼作者诚实;`!` 是作者对编译器「保证非空」的断言。

## 思考题

1. `const x = 42` 和 `let y = 42`,它们的类型分别是 `42` 还是 `number`?为什么?
2. 字面量类型 `"text"` 和基础类型 `string` 是什么关系?(提示:一个值属于前者,是否必然属于后者?)
3. 为什么打开 `noUncheckedIndexedAccess` 后,代码里会「凭空」多出很多 `!`?这个开关的代价和收益分别是什么?

---

上一章:[第 2 章 · 环境与最小 TypeScript 程序](02-setup-and-tsc.md)
下一章:[第 4 章 · 函数](04-functions.md)
