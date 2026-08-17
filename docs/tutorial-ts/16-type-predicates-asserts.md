# 第 16 章 · 类型谓词与断言守卫(`is` / `asserts` / `as` / `!`)

## 本节要学的概念

- 类型谓词(type predicate)`x is T`:让一个 `boolean` 函数「顺带收窄类型」。
- 断言守卫(assertion function)`asserts x is T`:断言失败就抛错,成功则收窄。
- 类型断言 `as` 与非空断言 `!`:作者向编译器「具结」,编译器不再验证。

## 一个老问题:函数里的收窄传不出去

第 6 章学过收窄,但有个局限:**收窄只在「判断写在哪」的那个作用域生效**。看这个:

```ts
function isString(x: unknown): boolean {
  return typeof x === 'string'
}

function f(x: unknown) {
  if (isString(x)) {
    x.toUpperCase()   // 报错!x 在这里没有收窄成 string
  }
}
```

问题出在 `isString` 的返回类型是 `boolean`。TS 看到一个「返回 boolean 的函数」,并不知道「返回 true 意味着 x 是 string」这层关系。收窄链条断了。

**类型谓词(type predicate)** 就是补上这层关系——把返回类型从 `boolean` 改成 `x is string`:

```ts
function isString(x: unknown): x is string {
  return typeof x === 'string'
}

function f(x: unknown) {
  if (isString(x)) {
    x.toUpperCase()   // 通过!这里 x 收窄成了 string
  }
}
```

`x is string` 读作「当这个函数返回 true 时,参数 x 是 string」。有了它,TS 在 `if (isString(x))` 里就把 `x` 收窄了。

## 项目真实用例:识别「纯 JSON 对象」

[`packages/core/tools/src/json-schema.ts`](../../packages/core/tools/src/json-schema.ts) 里有一大批这样的守卫函数:

```ts
export function isPlainJsonRecord(value: unknown): value is Record<string, unknown> {
  if (typeof value !== 'object' || value === null || Array.isArray(value)) return false
  // ... 更多判断
}
```

`value is Record<string, unknown>` 是类型谓词:这个函数「既判断,又收窄」——调用方在 `if (isPlainJsonRecord(x))` 分支里,`x` 就被收窄成 `Record<string, unknown>`(第 11 章的 Record)。

这类函数项目里非常多(还有 `isPlainJsonArray`、`isJsonNumber`、`scalarMatches`…),它们处理的是**从外部(模型输出、文件、网络)进来的、类型未知的值**,需要用运行时检查「验证形状 + 收窄类型」。这就是「收窄」在真实边界上的用途:第 18 章讲 `unknown` 时会再遇到。

## 断言守卫(assertion function)

有时候你不用「返回布尔值」,而是「如果不是,就抛错」。这时用**断言守卫(assertion function)**:

```ts
export function assertSupportedJsonSchema(schema: unknown): asserts schema is JsonSchemaNode {
  // 若 schema 不是合法 JSON schema,内部 throw
}
```

返回类型写成 `asserts schema is JsonSchemaNode`(而不是 `boolean`)。语义是:

- **调用之后**,如果函数没有抛错(正常返回),那么 TS 就认为 `schema` 是 `JsonSchemaNode`——**不需要 `if` 包裹**,直接收窄。

```ts
function use(schema: unknown) {
  assertSupportedJsonSchema(schema)   // 不是就抛错;是就继续
  schema.type   // 这里 schema 已收窄成 JsonSchemaNode,可直接访问
}
```

对比一下:

- 类型谓词 `x is T`:配合 `if` 用,返回布尔值,「是」则在分支里收窄。
- 断言守卫 `asserts x is T`:独立调用,失败抛错,「活过来」就在后续收窄。

两者都是「**把运行时检查,翻译成类型收窄**」的手段——把「我检查过这个值是 T」这件运行时事实,变成「编译器也知道它是 T」。

## 类型断言 `as` 与非空断言 `!`:作者单方面「具结」

和第 16 章主题押韵,还有两个「**作者向编译器表态**」的语法,前面各章已经零散见过,这里归拢:

**类型断言 `as`**:「我知道它其实是 T,按 T 处理」。

```ts
const el = document.querySelector('.x') as HTMLDivElement
```

`as` 不引入任何运行时检查,纯粹是**作者说「我确定」,编译器照办**。用错了,类型安全就从那一点「漏」出去(责任回到作者)。

**非空断言 `!`**:「我确定这个值不是 null/undefined,别报『可能空』」。

```ts
const first = list[0]!   // 见第 3 章 noUncheckedIndexedAccess
```

第 15 章 `dispatch.ts` 里 `{ ...payload, agent } as PayloadOf<K>` 用的就是 `as`——作者越过「TS 推不出来」的地方。而 `!` 在第 3 章讲过,是应对 `noUncheckedIndexedAccess` 的「我保证索引有值」。

> 三种东西别混:`is`(返回布尔、配合 if)/`asserts`(失败抛错、自动收窄)是**有运行时检查支撑**的收窄;`as`/`!` 是**零运行时、纯类型层的作者承诺**。前两者「检查过了才收窄」,后两者「作者口头保证」。真正常见的坑是「该用 is/asserts 的地方用了 as」——跳过检查,埋下隐患。

## 你现在能自己读了

回到 [`packages/core/tools/src/json-schema.ts`](../../packages/core/tools/src/json-schema.ts),做两件事:

1. 找到 `isPlainJsonRecord`、`isPlainJsonArray`、`isJsonNumber`(约第 115~180 行),观察它们的返回类型都是 `x is ...` 形式,体会「守卫函数军团」如何把 `unknown` 一层层收窄成确定类型。
2. 找到 `assertSupportedJsonSchema`(第 385 行附近)和 `assertObjectJsonSchema`,`asserts ... is ...` 的返回类型,体会「断言守卫」独立调用即可收窄的用法。

再想想:这些守卫函数,处理的是什么来源的值?为什么不用 `as` 而要用 `is`/`asserts`?(把它们和第 15 章 `dispatch.ts` 里那些 `as` 对照——那边是「同进程内、TS 已保证」的类型,这边是「进程边界外、不可信」的值。)

## 小结

- 类型谓词 `x is T` 让 `boolean` 函数顺带收窄类型,配合 `if` 用。
- 断言守卫 `asserts x is T` 失败抛错、成功收窄,独立调用即可。
- `as`/`!` 是纯类型层的作者断言,运行时无检查,用错则类型安全「漏」出去。

## 思考题

1. `isString(x): x is string` 和 `isString(x): boolean` 对调用方来说,行为上有区别吗?(运行期一样,但编译期差在哪?)
2. 为什么处理「模型输出、文件、网络」这类外部值,要用 `is`/`asserts`,而不是 `as`?
3. `as` 和 `!` 的共同点是什么?「用好它们的纪律」应该是什么?

---

上一章:[第 15 章 · 高级综合:读懂 dispatch.ts](15-reading-dispatch.md)
下一章:[第 17 章 · 名义类型:Branded 品牌类型](17-branded-types.md)
