# 第 6 章 · 联合类型与收窄(narrowing)

## 本节要学的概念

- 联合类型(union type)用 `|` 表示「这个值可能是几种类型之一」。
- 收窄(narrowing):从「可能是几种」缩小到「确定是哪种」,然后才能安全操作。
- 收窄的几种常见手段:`typeof`、`===`、`in`、真值判断。

## 联合类型:一个值可能是几种

看一个真实的例子,来自 [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts):

```ts
export type AgentCancelCause =
  | { readonly kind: 'user' }
  | { readonly kind: 'parent' }
  | { readonly kind: 'hook'; readonly reason: string }
  | { readonly kind: 'disposed' }
```

`AgentCancelCause` 是一个**联合类型(union type)**:它的值是**四种对象之一**(用户取消 / 父级取消 / hook 取消 / 已销毁)。竖线 `|` 读作「或」,把四个「成员(member)」并成一个类型。

更简单的联合也能是基础类型:

```ts
type MaybeString = string | null
type Result = string | number
```

类型 `string | null` 表示「要么是字符串,要么是 null」。

## 问题来了:联合类型怎么安全地用?

联合类型意味着「我不知道它到底是哪一个」。如果直接对一个 `string | number` 做「只有 string 才能做」的操作,TS 会拦你:

```ts
function length(x: string | number) {
  return x.toUpperCase()   // 报错:number 没有 toUpperCase
}
```

因为「这个值可能是 number」,而 number 没有 `.toUpperCase()`,TS 出于安全,拒绝你在这个「还不确定」的位置调用它。

**解法就是收窄(narrowing)**:先判断「它到底是哪个成员」,判断之后,TS 在你写判断的那个分支里,把类型「收窄」成确定的那个。

## 收窄手段一:`typeof`

```ts
function length(x: string | number): number {
  if (typeof x === 'string') {
    return x.length   // 在这一行,x 被收窄成 string
  }
  return String(x).length   // 走到这里,x 一定是 number
}
```

`typeof x === 'string'` 是 JS 自带的判断,TS 理解它:在这个 `if` 块里,`x` 的类型从 `string | number` **收窄**成了 `string`。

## 收窄手段二:`===` 与字面量判断

第 3 章学的字面量类型,这里派上用场。回到 `AgentCancelCause`:

```ts
function describe(cause: AgentCancelCause): string {
  if (cause.kind === 'user') {
    return '用户取消'          // 这里 cause 收窄成 { kind: 'user' }
  }
  if (cause.kind === 'hook') {
    return 'hook 取消: ' + cause.reason   // 这里 cause 收窄成 { kind: 'hook'; reason: string }
  }
  return '其他原因'
}
```

用 `cause.kind === 'user'` 判断字面量,TS 就把 `cause` 收窄成那个具体的成员。注意第二个 `if` 里能访问 `cause.reason`——因为 `reason` 只存在于 `{ kind: 'hook' }` 这个成员上,**只有收窄之后才允许访问**。

## 收窄手段三:真值判断

可选参数(第 4 章)的默认收窄方式:

```ts
function title(name: string | undefined): string {
  if (name) {
    return name.toUpperCase()   // 这里 name 收窄成 string
  }
  return '(未命名)'
}
```

`if (name)` 把 `undefined`(以及 `null`、空串、`0` 等「假值」)排除掉,收窄成 `string`。

## 收窄手段四:`in` 判断「有没有某个属性」

```ts
function area(shape: { kind: 'circle'; r: number } | { kind: 'rect'; w: number; h: number }) {
  if ('r' in shape) {
    return Math.PI * shape.r ** 2   // 收窄成 circle
  }
  return shape.w * shape.h          // 收窄成 rect
}
```

`'r' in shape` 判断「这个对象有没有 `r` 属性」,以此区分两种形状。

## 一个真实写照:todo 状态

回到 [`types.ts`](../../packages/core/session/src/types.ts),`TodoItem` 的 status 字段:

```ts
export interface TodoItem {
  content: string
  status: 'pending' | 'in_progress' | 'completed'
}
```

`status` 的类型是一个**由三个字面量组成的联合类型**:它的值只能是这三个字符串之一,写别的(`status: "done"`)会编译报错。这就是用「联合 + 字面量」把「合法取值」锁死的手法。

## 你现在能自己读了

回到 [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts) 的 `AgentCancelCause`,试着自己写(在心里)一个 `describe(cause)` 函数,用 `if / else if` 把四种 `kind` 都处理一遍,体会「每个分支里类型如何被收窄」。

再看 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts) 里的这两个字段,说出它们的联合成员:

```ts
finishedReason?: FinishReason
inputModalities?: readonly ModelModality[]
```

你先不用懂 `FinishReason`、`ModelModality` 是什么类型(它们是第 7 章的主角),只要注意 `?:` 让这些字段本身成了 `T | undefined`——**可选属性本质就是和 `undefined` 的联合**。

## 小结

- 联合类型 `A | B` 表示「可能是 A 或 B」;操作前必须先收窄。
- 收窄手段:`typeof`、`===` 字面量、真值判断、`in`。
- 收窄是「类型跟着你的判断走」——TS 理解你的 `if` 分支,把类型一步步缩小。

## 思考题

1. 为什么「直接对 `string | number` 调 `.toUpperCase()`」会被拦,而「写个 `typeof` 判断后再调」就合法?两者对 TS 来说区别是什么?
2. `AgentCancelCause` 的第二个成员是 `{ kind: 'parent' }`,它没有 `reason` 字段。如果你没先收窄就直接 `cause.reason`,会发生什么?
3. `status: 'pending' | 'in_progress' | 'completed'` 这种「联合字面量」和直接写 `status: string` 相比,好在哪?

---

上一章:[第 5 章 · 对象与接口](05-interfaces.md)
下一章:[第 7 章 · 判别联合与穷尽检查](07-discriminated-unions.md)
