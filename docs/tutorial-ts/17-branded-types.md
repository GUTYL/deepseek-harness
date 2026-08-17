# 第 17 章 · 名义类型:`Branded<B>` 品牌类型

## 本节要学的概念

- 结构类型(第 5 章)的短板:两个「形状相同」的字符串类型会互相混用。
- 品牌类型(branded type):给类型打上编译期「标签」,让形状相同的类型**不可互换**。
- `unique symbol`、交叉类型、`as` 断言如何共同实现它。

## 回顾:结构类型的「漏洞」

第 5 章说过,TS 是**结构类型(structural typing)**:只看形状,不看名字。这带来了一个具体问题——考虑两种 id:

```ts
type SessionId = string
type CallId = string
```

`SessionId` 和 `CallId` **形状都是 string**,所以结构上「一模一样」。于是下面这种张冠李戴,TS **完全不报错**:

```ts
function logCall(id: CallId) { ... }

const sid: SessionId = "abc"
logCall(sid)   // 通过!但语义上是把「会话 id」当「调用 id」传,是 bug
```

这两种 id 在业务上**绝对不能混用**(会话 id 不该被当成工具调用 id),但类型系统——因为它们的结构都是 string——**看不出任何区别**。

**品牌类型(branded type)就是用来强行制造这个区别的**:给类型贴一个「只有编译器认识、运行时不存在」的标签,让两个结构相同的类型**名义上(按名字/标签)不再相同**。

## 实现:一个类型层面的「标签」

第 9 章你已经见过定义,这里完整再读。打开 [`packages/util/brand/src/index.ts`](../../packages/util/brand/src/index.ts):

```ts
declare const BRAND: unique symbol

export type Branded<B extends string> = string & { readonly [BRAND]: B }
```

三段分别是什么,现在都能说清了:

1. **`declare const BRAND: unique symbol`** —— 声明一个**独一无二**的符号。`unique symbol` 保证它不会和任何其他符号相等——每个 `unique symbol` 都是「全世界仅此一个」的键。
2. **`string & { readonly [BRAND]: B }`** —— 交叉类型(第 9 章):「既是 string,又多背一个属性:键为 `BRAND` 这个独一符号、值为 `B`」。`B` 是泛型,传入 `'CallId'` 之类(第 3 章的字面量)。
3. 因为 `BRAND` 是 unique symbol,两个不同的品牌 `Branded<'CallId'>` 和 `Branded<'SessionId'>`,它们的「标签属性」类型不同(`'CallId'` vs `'SessionId'`),于是它们**在类型上不再兼容**——即使运行时都是普通 string。

关键一词:**`declare`**。`declare const BRAND` 表示「这个符号**只存在于类型层,运行时不存在**」。所以「打品牌」这个操作是**零运行时成本**的——编译后所有 `& { readonly [BRAND]: ... }` 都被擦除,剩下的就是普通 string(第 2 章的「类型编译后消失」)。

## 项目怎么用它:一个 id 一个工厂

第 3、4 章你见过 `brand.ts` 里的用法。再看 [`packages/llm/llm/src/brand.ts`](../../packages/llm/llm/src/brand.ts) 的完整模式:

```ts
import type { Branded } from '@deepseek-ai/dsh-brand'

export type CallId = Branded<'CallId'>

export function CallId(id: string): CallId {
  return id as CallId
}
```

要点(第 4 章讲过「类型与值同名」):

- `type CallId = Branded<'CallId'>` 定义**类型**(品牌化的字符串)。
- `function CallId(id: string): CallId` 是**工厂函数**,把普通 string「包装」成品牌类型。函数体里 `return id as CallId` 用 `as`(第 16 章)完成这个「贴标签」。

这就强制了「**id 必须在产生它的地方通过工厂创建**」——你想得到一个 `CallId`,不能随便拿个字符串就用,得走 `CallId(s)` 这个工厂。类型系统由此建立了「名义上的身份」:一个 `SessionId` 值,类型上就是过不了 `logCall(id: CallId)` 这一关。

项目注释把这条政策说得很清楚(见 brand.ts 顶部 JSDoc):

> 「一个包对它自己拥有的 id 打品牌——`CallId` 归 dsh-llm、共享的 `SessionId` 归 dsh-session、`JobId` 归 dsh-jobs。品牌用于**跨包边界、且可能混淆**的 id;不是每个字符串都该打品牌。」

所以品牌类型的使用是**有节制的**:只在「跨边界 + 易混淆」的 id 上才值得那一点类型成本,而不是到处滥贴。

## 结构类型与名义类型:项目两头都用

第 5 章结尾留过一个对照,现在闭环了:

- **默认是结构类型**(第 5 章):对象靠形状匹配,这带来了「声明合并」「插件化」的便利(第 14 章)。
- **需要时用品牌类型「造」出名义类型**:在 id 这种「形状相同但语义绝不可混」的地方,用 `Branded` 手工引入名义区分。

一套语言里,两种类型哲学按需选用——这正是本项目的类型设计观:大部分地方交给灵活的结构类型,关键的、易混淆的边界用品牌类型收紧。

## 你现在能自己读了

1. 通读 [`packages/util/brand/src/index.ts`](../../packages/util/brand/src/index.ts) 全文(27 行),现在你应该能逐句解释:为什么 `BRAND` 要 `unique symbol`、为什么用 `&` 交叉、为什么 `declare`、为什么 `B extends string`。

2. 在 [`packages/llm/llm/src/brand.ts`](../../packages/llm/llm/src/brand.ts) 里,这个包打包了 4 个 id(`MessageId`/`CallId`/`ProviderRequestId`/`ReasoningEffortId`)。想想:为什么它们值得打品牌?它们跨了哪些边界、可能和什么混淆?

## 小结

- 结构类型的短板:形状相同的类型会互混;品牌类型用「类型层标签」制造名义区分。
- `Branded<B> = string & { readonly [unique symbol]: B }`:交叉类型 + unique symbol + declare,零运行时成本。
- 打品牌要节制:只在跨边界、易混淆的 id 上使用,配套工厂函数。

## 思考题

1. `Branded<'CallId'>` 和 `string` 在运行时有什么区别?在编译期呢?
2. 为什么 `BRAND` 必须声明成 `unique symbol`,而不是一个普通字符串键 `{ brand: B }`?(提示:普通键可能意外撞名,unique symbol 保证唯一。)
3. 工厂函数 `CallId(id)` 体里的 `as CallId`,属于第 16 章说的哪一类(有运行时检查的收窄,还是纯类型层作者断言)?为什么这里用 `as` 是安全的?

---

上一章:[第 16 章 · 类型谓词与断言守卫](16-type-predicates-asserts.md)
下一章:[第 18 章 · never / unknown / any 的三分法](18-never-unknown-any.md)
