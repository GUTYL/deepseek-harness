# 第 8 章 · 类型别名、数组与元组

## 本节要学的概念

- 类型别名(type alias)用 `type` 给类型起名字。
- 数组类型 `T[]` 与只读数组 `readonly T[]`。
- 元组(tuple):固定长度、各位置类型不同的数组。
- `as const`:把值「冻结」成字面量类型。

## 类型别名 type

第 7 章你已经一直在用 `type X = ...`。这就是**类型别名(type alias)**:给一个(可能很长的)类型起个短名。

```ts
type UserId = string
type Maybe<T> = T | null   // 带泛型的别名,第 9 章细讲
```

`type` 和 `interface`(第 5 章)都能描述类型,但有分工(项目规则也强调):

- `interface` 描述**对象的形状**,且支持「声明合并」(第 14 章)——能被别的文件扩展。
- `type` 用于**联合、字面量、函数类型、别名**等 interface 表达不了的东西。

简单记:**「这是什么对象的形状」用 `interface`;「几种类型之一」「给类型起别名」「函数签名」用 `type`。**

## 数组类型

```ts
const names: string[] = ["Ada", "Grace"]    // 字符串数组
const matrix: number[][] = [[1, 2], [3]]    // 数字的数组的数组(二维)
```

`T[]` 读作「元素类型为 T 的数组」。第 4 章 rest 参数里的 `...parts: string[]` 就是这个。

**只读数组** `readonly T[]`(或 `ReadonlyArray<T>`):元素不能增删改。

```ts
function total(values: readonly number[]): number {
  values.push(1)   // 报错:readonly 数组不能 push
  return values.reduce((a, b) => a + b, 0)
}
```

项目里大量用 `readonly`,含义是「这个数组我只读、不改」——向调用者承诺「我不会动你的数组」。

## 元组:定长、定型的数组

普通数组「每个元素同类型、长度不定」。**元组(tuple)** 相反:长度固定,每个位置有自己的类型。

```ts
type Pair = [string, number]
const p: Pair = ["year", 2026]   // 第一个必须是 string,第二个必须是 number
const bad: Pair = [2026, "year"] // 报错:位置上的类型对不上
```

元组的典型用途是「**打包几个相关的值,且顺序有意义**」。本项目的 `StreamChunk` 里,能看到元组的真实用例(见下)。

## `as const`:把值冻结成字面量

这是项目里极常见、也常让新手一愣的写法。看 [`packages/core/tools/src/json-schema.ts`](../../packages/core/tools/src/json-schema.ts) 里这一行:

```ts
const SCHEMA_TYPES: readonly JsonSchemaType[] = ['object', 'array', 'string', 'number', 'integer', 'boolean', 'null']
```

以及 [`packages/core/tools/src/schema.ts`](../../packages/core/tools/src/schema.ts) 里:

```ts
const ANNOTATION_KEYS = ['description', 'title', 'default', 'examples'] as const
```

看那行末尾的 **`as const`**。它做什么?先看没有 `as const` 会怎样:

```ts
const KEYS = ['a', 'b']        // 类型推断为 string[](任意字符串数组,可增删)
const KEYS2 = ['a', 'b'] as const   // 类型推断为 readonly ['a', 'b'](只读元组,值被锁定)
```

区别:

- 没有 `as const`:`KEYS` 是 `string[]`,TS 只知道「这是个字符串数组」。
- 有 `as const`:`KEYS2` 类型变成 **`readonly ['a', 'b']`**——一个**只读元组**,而且每个元素都被推断成字面量 `'a'`、`'b'`,不是泛泛的 `string`。

`as const` 做了两件事:**① 把 `T[]` 升级成元组(定长);② 把元素类型升级成字面量;③ 整体设为只读。**

为什么项目要这样写?拿 `SCHEMA_TYPES` 举例:下面这段代码(还是同一个文件)用它来校验:

```ts
if (typeof type !== 'string' || !(SCHEMA_TYPES as readonly unknown[]).includes(type)) {
  // type 不是合法的 schema 类型
}
```

只有当 `SCHEMA_TYPES` 的元素是精确的字面量(`'object'`、`'array'`…),后续「`type` 是否在这个合法集合里」的类型关系才精确。用 `as const` 锁定,TS 才能在别处对 `type` 做精确收窄。

> 一句话:`as const` 表示「这个值我很确定它就是这个字面量集合,以后也不会变,请按精确的字面量类型对待它」。

## 你在前面其实已经见过元组了

第 6 章学过 `AgentCancelCause`,它是「对象成员的联合」。而 `StreamChunk`(见 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts))是「**对象成员的联合**」,但它在项目里也大量出现。这里我们先巩固元组概念,用一个人为的简单例子:

```ts
type Event = ['start', number] | ['text', string] | ['end']
```

这是一个**由元组组成的判别联合**——每个成员是定长数组,第一个元素是判别标签(`'start'`/`'text'`/`'end'`)。这是「用元组的位置来表达类型」的经典形态。真实项目里,「按位置打包」更多出现在函数参数、`Promise` 的并发结果等地方,你读到会认出来。

## 你现在能自己读了

在项目里搜 `as const`(用编辑器的全局搜索),随便点开几处,观察它们右侧的数组、对象长什么样,思考「作者为什么想把这些值锁成字面量」。

再到 [`packages/core/tools/src/py-types.ts`](../../packages/core/tools/src/py-types.ts) 里找这一行:

```ts
const TYPING_ORDER = ['Any', 'Literal', 'NotRequired', 'Protocol', 'TypedDict'] as const
```

现在你能说出:`TYPING_ORDER` 被 `as const` 锁成了一个只读元组 `readonly ['Any', 'Literal', 'NotRequired', 'Protocol', 'TypedDict']`,每个元素都是精确字面量。

## 小结

- `type` 给类型起别名、表达联合/函数类型/元组;`interface` 描述对象形状。
- `T[]` 是数组,`readonly T[]` 是只读数组;元组是定长定型数组。
- `as const` 把值锁成只读的、字面量化的元组/对象类型。

## 思考题

1. `string[]` 和 `readonly ['a', 'b']` 有什么区别?一个值能同时属于两者吗?
2. `const KEYS = ['a', 'b']`(无断言)和 `const KEYS = ['a', 'b'] as const`,各自的类型分别是什么?
3. 为什么「合法的 schema 类型集合」这种常量,要用 `as const` 锁住,而不是直接写 `const SCHEMA_TYPES = [...]`?

---

上一章:[第 7 章 · 判别联合与穷尽检查](07-discriminated-unions.md)
下一章:[第 9 章 · 泛型](09-generics.md)
