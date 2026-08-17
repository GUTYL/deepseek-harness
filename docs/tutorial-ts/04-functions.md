# 第 4 章 · 函数:参数、返回值、可选与默认

## 本节要学的概念

- 函数的参数与返回值类型标注。
- 可选参数 `?`、默认参数、rest 参数。
- 函数类型(function type)与箭头函数类型的写法。

## 给函数标类型

```ts
function greet(who: string): string {
  return "Hello, " + who
}
```

两个注解位置:

- 参数 `who: string` —— 调用者必须传字符串。
- 返回值 `: string`(在 `)` 之后、`{` 之前)—— 这个函数保证返回字符串。

返回值其实也常常能推断:上面即使不写 `: string`,TS 也能从 `return` 推断出返回 `string`。但项目里**几乎所有导出函数都显式写了返回值类型**,原因在项目规则里:

> 每个模块和导出都有简洁的 JSDoc,函数式导出包含 `@param`/`@returns`。

显式写返回类型不只是风格,它有一层实用价值:**编译器会检查「我声明的返回类型」和「我实际 return 的类型」是否一致**——相当于一次免费的自我校验。返回类型写错或漏写,会立刻暴露。

## 参数传错时,TS 替你拦下

```ts
greet(42)   // 编译错误:Argument of type 'number' is not assignable to parameter of type 'string'
```

这就是类型标注的第一个收益:**调用处的参数错误,在编译期就被抓住**,而不是等运行期在函数内部炸掉。

## 可选参数 `?`

有些参数可以不给:

```ts
function resolveHome(configured?: string): string {
  // ...
}
```

`configured?` 的 `?` 表示「这个参数可选」。它的类型含义是 `string | undefined`(第 6 章会正式讲 `|`,现在理解为「要么是字符串,要么没传」)。

在**函数体内部**,`configured` 因此可能是 `undefined`,所以想用它之前要先判断它存不存在。这是「可选参数」和「收窄(narrowing)」这两个概念第一次挂钩——它们其实是一对搭子,收窄会在第 6 章细讲。

## 默认参数

```ts
function resolveHome(configured: string | undefined = undefined): string {
  // ...
}
```

`= undefined` 是**默认参数**——调用者不传时,`configured` 取默认值 `undefined`。这是项目里很常见的写法:它在不改变可读性的前提下,让「可选」这件事变得更明确。项目里 `home-paths` 的真实写法正是这样(见下文)。

## rest 参数:收拢不定个数的参数

```ts
function join(...parts: string[]): string {
  return parts.join("-")
}
join("a", "b", "c")   // "a-b-c"
```

`...parts: string[]` 表示「剩余任意个字符串参数,收拢成一个数组叫 `parts`」。`string[]` 表示「字符串数组」(第 8 章详细讲数组)。

## 函数类型与箭头函数

函数本身也是一种值,所以可以写「函数的类型」:

```ts
// 一种写法:type 别名(第 8 章讲 type)
type GreetFn = (who: string) => string

// 直接注解一个变量
const greet: (who: string) => string = (who) => "Hello, " + who
```

`(who: string) => string` 读作「一个接受 string、返回 string 的函数」。注意区分两个 `=>`:

- 在**类型位置**(`:` 后面)的 `=>` 描述函数的签名。
- 在**值位置**(`=` 右边)的 `=>` 是一个箭头函数。

它们在同一个表达式里可能同时出现,别混淆。

## 项目中一个真实的函数:解析 Harness 目录

打开 [`packages/util/home-paths/src/index.ts`](../../packages/util/home-paths/src/index.ts),找这个函数:

```ts
export function resolveDshHome(
  configured?: string,
  env: Record<string, string | undefined> = process.env,
): string {
  // ...
}
```

现在你能逐段读懂这个签名:

- `configured?: string` —— 可选参数,是字符串或没传。
- `env: Record<string, string | undefined> = process.env` —— 一个默认参数。`Record<string, ...>` 表示「一个键是字符串的映射对象」(第 11 章细讲),`= process.env` 是它的默认值(环境变量)。
- `): string` —— 返回字符串。

你看,一个真实的「解析 home 目录」函数,签名只用到了本章三个知识点:可选参数、默认参数、返回类型。这就是「由浅入深读真实代码」的节奏——复杂的东西其实是简单东西垒起来的。

## 你现在能自己读了

回到 [`packages/llm/llm/src/brand.ts`](../../packages/llm/llm/src/brand.ts),那组 id 工厂函数:

```ts
export function CallId(id: string): CallId {
  return id as CallId
}
```

注意一个微妙点:函数名 `CallId` 和返回类型 `CallId` **同名**。TS 里「类型」和「值」住在两个不同的命名空间(类型空间 / 值空间),所以一个叫 `CallId` 的 type 和一个叫 `CallId` 的函数可以共存而不冲突。这里 `function CallId(...): CallId` 意思是:定义一个运行期函数 `CallId`,它返回类型 `CallId`。这个「类型与值同名」的现象,是 TS 里常见、也常让新手困惑的一点,看到先眼熟即可。

## 小结

- 函数标注两处:参数 `: T` 和返回值 `: T`;显式写返回类型能免费校验自己。
- 可选参数 `?` 的类型是 `T | undefined`,函数体内使用前需收窄。
- 参数错误在编译期就被拦下,而不是运行期才炸。

## 思考题

1. `configured?: string` 和 `configured: string | undefined` 有区别吗?在函数体内,你都需要做什么才能安全使用它?
2. `(who: string) => string` 里的 `=>` 和 `const f = (who) => ...` 里的 `=>`,哪个是「箭头函数」,哪个是「函数类型」的记号?
3. 为什么项目强调「函数式导出要写 `@param` 和 `@returns`」?这和「显式写返回类型」有什么相通之处?

---

上一章:[第 3 章 · 变量与基础类型注解](03-basic-types.md)
下一章:[第 5 章 · 对象与接口](05-interfaces.md)
