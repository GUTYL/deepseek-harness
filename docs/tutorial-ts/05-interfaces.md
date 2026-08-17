# 第 5 章 · 对象与接口(interface)

## 本节要学的概念

- `interface` 定义对象的形状(shape)。
- 可选属性与 `readonly`。
- 结构类型(structural typing):TS 靠「形状」匹配,不靠名字。

## 用 interface 描述一个对象

现实里的数据很少是单个字符串或数字,而是一个个**有结构的对象**。比如一条「文本消息块」:

```ts
interface TextBlock {
  type: 'text'
  text: string
}
```

这就是一条 `interface`(接口)声明。它读作:「一个 `TextBlock` 对象,有一个 `type` 属性(字面量类型 `'text'`),和一个 `text` 属性(string)」。

然后你就能用它:

```ts
const block: TextBlock = {
  type: 'text',
  text: 'hello',
}
```

少一个属性、多一个属性、类型对不上,TS 都会在编译期报错——这就是 `interface` 的核心价值:**锁住对象的形状**。

## 这是项目里最常用的声明方式

翻遍本项目,`interface` 是无处不在的主角。比如 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts) 里的失败信息:

```ts
export interface LlmFailure {
  readonly message: string
  readonly code: string
  readonly status?: number
}
```

逐字段看,又学两个新东西:

- `readonly` —— **只读属性**:一旦创建,不能再给这个字段重新赋值(防止误改)。
- `status?: number` —— **可选属性**:`?` 表示「这个字段可有可无」。读它的类型时是 `number | undefined`。

注意 `readonly` 修饰的是「能不能改」,`?` 修饰的是「存不存在」——两者正交,可以同时用(`readonly status?: number` 表示「可能有、有也不能改」)。

## 结构类型:为什么 TS 不看你起的名字

TS 判断「一个对象符不符合某个接口」,**只看形状,不看它有没有声明实现这个接口**。这叫**结构类型(structural typing)**,是 TS 和 Java/C# 这类「名义类型(nominal typing)」语言最根本的区别。

```ts
interface Point { x: number; y: number }

const p = { x: 1, y: 2, z: 3 }   // 这对象有个多余的 z,也没声明实现 Point

const q: Point = p   // 合法!因为它「形状上」满足 Point(x 和 y 都是 number)
```

多出来的 `z` 不影响:只要它**至少有** `x` 和 `y` 且类型对,它就「是」一个 `Point`。这个特性对理解本项目至关重要——后面讲服务、事件、声明合并时,你会发现项目大量依赖「形状即可匹配」这件事。

> 和它相对的「名义类型」是「名字对不上就不行」,本项目在需要名义类型的地方(Token id 之类)会借 `Branded` 手工造出来——那正是第 17 章的主题。两种类型哲学,本项目都会用到,这个对照很值得记住。

## interface 也能描述函数和方法

除了「数据对象」,interface 还能描述「带方法的对象」或「函数本身」。本项目 `dispatch.ts` 里的派发器就是带方法的接口(第 15 章会精读):

```ts
export interface AgentEventDispatch {
  emit<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>): void
  serial<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>): Promise<Awaited<Return<Events[K]>>>
  waterfall<K extends AgentSubjectEvent>(name: K, payload: PayloadRest<K>, ...rest: Tail<K>): Return<Events[K]>
}
```

现在你当然读不懂 `<K extends ...>`、`PayloadRest<K>`、`Awaited<...>` 这些是什么——它们分别属于第 9、11、12、13 章。但你要能认出**结构**:这是一个「带三个方法的对象」的接口,每个方法都是「函数类型」(第 4 章刚学过 `(参数序列) => 返回`)。把它当「一个对象的形状说明书」来读,方向就对。

## 你现在能自己读了

打开 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts),找这几条 interface,逐一读出它们的形状:

```ts
export interface TextBlock {
  type: 'text'
  text: string
}

export interface ToolCallBlock {
  type: 'tool-call'
  id: CallId
  name: string
  arguments: string
}
```

现在你至少能读出:`TextBlock` 有 `type`(必须是字面量 `'text'`)和 `text`(string);`ToolCallBlock` 有 `type`(字面量 `'tool-call'`)、`id`(类型是 `CallId`,第 3 章介绍过这个品牌类型)、`name`(string)、`arguments`(string)。

注意每个块的第一个字段都是 `type`,而且值是**互不相同的字面量** `'text'` / `'tool-call'`。这正是第 7 章「判别联合」的伏笔——先记住这个「形状」。

## 小结

- `interface` 锁住对象的形状:属性、类型、可选 `?`、只读 `readonly`。
- TS 是**结构类型**:只看形状是否匹配,不看名字是否声明。
- `readonly` 管「能否改」,`?` 管「是否存在」,可以叠加。

## 思考题

1. 为什么说「TS 是结构类型的」?用自己的话解释 `const q: Point = p`(p 多了个 z)为什么合法。
2. `readonly status?: number` 同时用了两个修饰符,它表达的完整含义是什么?
3. 在 [`types.ts`](../../packages/llm/llm/src/types.ts) 里,`TextBlock.type` 是 `'text'` 而不是 `string`,作者为什么故意这样写?(提示:想想「想锁住什么」。)

---

上一章:[第 4 章 · 函数](04-functions.md)
下一章:[第 6 章 · 联合类型与收窄](06-unions-and-narrowing.md)
