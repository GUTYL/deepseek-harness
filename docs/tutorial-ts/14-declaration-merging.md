# 第 14 章 · 声明合并与模块扩展(declaration merging)

## 本节要学的概念

- 声明合并(declaration merging):多个同名声明会「合并」成一个。
- 模块扩展(module augmentation):用 `declare module` 给**已存在的模块**追加类型。
- 这是本项目「一切皆插件」在类型层的根基。

## 什么是声明合并

在 TS 里,如果你**声明两个同名的 `interface`**,它们不会冲突,而是**合并(merge)**:

```ts
interface User {
  name: string
}

interface User {
  age: number
}

// 结果 User = { name: string; age: number }
```

两个 `User` 合并成一个,属性并集。这就是**声明合并(declaration merging)**。它只对 `interface`(以及 namespace 等)生效;`type` 别名**不能**合并(重复的 `type` 会报错)——这也是为什么项目里「要能被扩展的类型」都用 `interface` 而不写 `type`。

## 模块扩展:给别人的模块加类型

声明合并有个威力巨大的应用场景:**你想给一个「你不拥有」的模块追加类型声明**。比如某个第三方包已经定义了一个接口,你想给它加个新属性。

语法是 `declare module '包名' { ... }`:

```ts
// 假设这个接口来自第三方包 "some-lib",里面有个 User 接口
declare module 'some-lib' {
  interface User {
    email: string   // 我给它的 User 追加一个 email 字段
  }
}
```

`declare module` 告诉 TS:「我要对 `some-lib` 这个模块做**模块扩展(module augmentation)**——往它已有的类型里合并新内容」。于是,在你项目的所有文件里,引 `User` 时都能看到多出来的 `email`,类型信息全项目通用。

## 为什么它对本项目是「灵魂」

现在把「声明合并 + 模块扩展」和本项目的核心概念「**一切皆插件**」接起来,你会看到整个设计的精妙之处。

回顾第 1 章:`dsh` 里的每个能力都是一个**插件**,插件之间通过 `ctx` 这个共享上下文协作。那么问题来了:**插件的作者怎么「声明」自己往 `ctx` 上挂了一个新服务、往事件系统里加了新事件?**

答案就是声明合并。Cordis(`vendor/cordis`)预先声明了一个 `Context` 接口和一个 `Events` 接口,但**它们是空的、开放的,等着别人来合并**。插件作者写:

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    llm: LlmService        // 我要往 ctx 上挂一个叫 llm 的服务
  }
  interface Events {
    'llm/adapters-updated'(): void   // 我要新增一个事件
  }
}
```

合并之后:

- 全项目的 `ctx.llm` 都有了类型(编译器知道 `ctx` 上有 `llm` 这个字段)。
- 全项目的 `ctx.on('llm/adapters-updated', ...)` 都有了类型检查。

**这就是「插件化」在类型层面的实现**:插件作者不需要修改 Cordis 的源码,只要在自己的文件里 `declare module` 合并一下,就把自己的贡献「挂」进了共享类型空间。运行时靠注册(第 21 章讲),类型则靠声明合并——两条腿走路,一条都不能少。

> 这也是第 3 章教程里告诉过你的一句关键事实,现在完全展开:**`declare module` 块运行时什么都不做**。它纯粹是「给共享类型加口子」的编译期手段,和真正把服务注册进 `ctx` 的运行时代码是两回事,靠「名字一致」的约定连起来。

## 项目里到处都在这么干

你可以在整个 `packages/` 下看到几百处 `declare module`。它们合并不止 `Context`/`Events`,还有各种「Map」接口(第 7、10 章的那些可扩展联合):

```ts
declare module '@deepseek-ai/dsh-session/types' {
  interface SessionEventMap {
    'my/new-event': { turn: number }   // 给会话日志新增一种事件
  }
}

declare module '@deepseek-ai/dsh-typert-protocol' {
  interface TypertLookupMap {
    agent: TypertLookup<Agent, AgentId>   // 给 RPC 注册表新增一个可远程调用的服务
  }
}
```

规律是一样的:项目定义了一批「空的、待合并」的 Map/接口(如 `SessionEventMap`、`ContentBlockMap`、`TypertLookupMap`),各插件按需往里面 `declare module` 合并新键,**对应的联合类型(用 `keyof`/索引访问推导,第 10 章)就自动变宽**。

第 7 章的「可扩展联合」、第 10 章的「Map + keyof」,加上本章的「declare module 合并」,至此串成了一个完整的闭环:

```text
插件用 declare module 往 Map 接口加键
        ↓
Map 接口被声明合并,多了新成员
        ↓
keyof 取键、索引访问取成员(第 10 章)
        ↓
对应的联合类型自动变宽
        ↓
所有 switch/类型检查处感知到新情况
```

## 你现在能自己读了

1. 打开 [`packages/core/session/src/types.ts`](../../packages/core/session/src/types.ts),找到 `interface SessionEventMap { ... }`(第 236 行附近)。注意它的 JSDoc 写着「merge-extensible」——这是项目给「待合并接口」的标记。你会看到它已经列了 `turn/start`、`assistant/chunk`、`tool/call` 等事件。

2. 再随便打开一个有 `declare module '@deepseek-ai/dsh-session/types'` 的文件,比如 [`packages/compaction/compaction/src/types.ts`](../../packages/compaction/compaction/src/types.ts),看它怎么往 `SessionEventMap` 里追加 compaction 自己的事件。对照第 1 步,体会「核心定义在 session,扩展在各插件」的分工。

## 小结

- 同名的 `interface` 会声明合并;`type` 不能合并。
- `declare module '包名'` 是模块扩展:给已有模块追加类型,全项目生效。
- 插件靠 `declare module` 合并进 `Context`/`Events`/各种 Map,实现「插件化」的类型层——运行时如何注册是另一条腿(第 21 章)。

## 思考题

1. 为什么「可扩展接口」必须用 `interface` 而不是 `type`?结合「type 不能声明合并」回答。
2. `declare module` 块里写的内容,运行时会执行吗?它和「真正把服务注册进 ctx」的代码,是靠什么对应起来的?
3. 如果一个插件作者忘了写 `declare module` 合并 `Context`,只做了运行时注册,会发生什么(编译期 / 运行期各是什么表现)?

---

上一章:[第 13 章 · 模板字面量类型](13-template-literal-types.md)
下一章:[第 15 章 · 高级综合:完整读懂 dispatch.ts](15-reading-dispatch.md)
