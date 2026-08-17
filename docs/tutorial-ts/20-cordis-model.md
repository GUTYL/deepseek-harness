# 第 20 章 · 心智模型:一切皆插件(Cordis)

## 本节要学的概念

- Cordis 框架的核心:插件、上下文、服务、事件、可逆注册。
- 为什么「一切皆插件」是这个项目所有设计的原点。
- 建立一张能装下后面 6 章的「心智地图」。

## 先建立直觉:一个共享工厂

现在你已学完 TS,可以真正读懂项目了。我们回到第 1 章埋下的那句「一切皆插件」,把它展开成一张能指导读代码的地图。

想象一个**共享工厂(上下文 Context)**。工厂里有若干**工位(服务 Service)**:

- `llm` 工位:和语言模型对话。
- `tools` 工位:登记和执行工具(跑 bash、读写文件…)。
- `sessions` 工位:记录发生过的所有事(日志)。
- `agents` 工位:管理正在运行的智能体。

每个工位由某个**插件(plugin)**提供。插件之间**不直接 import 对方**,而是通过 `ctx.llm`、`ctx.tools` 这样的**工位名**找到彼此。

工厂里还有一套**广播系统(事件 Event)**:插件可以「喊一声」,其他不认识它的插件也能听见并响应。

最重要的是:**插件装进工厂,是可撤销的**——谁开了工位、谁挂了监听,插件卸载时全自动清理。

用技术语言复述一遍(即 Cordis 的五个核心概念,详见 [`cordis-primer.md`](../cordis-primer.md)):

| 概念 | 一句话 |
|---|---|
| 插件(plugin) | 一个函数/对象/类,通过 `apply(ctx)` 挂进上下文 |
| 上下文(Context) | 所有插件的共享仓库,`ctx.xxx` 取值 |
| 注入(inject) | 声明「我需要哪些工位」,加载顺序由依赖决定,不靠文件顺序 |
| 事件(Events) | 通过声明合并新增事件名,再 `emit`/`waterfall` 等派发 |
| 可逆效果(effect) | 所有注册走 `ctx.effect()`/`ctx.on()`,卸载时自动撤销 |

现在,把这五个概念和你学过的 TS 特性一一挂钩——你会发现「框架设计」和「类型设计」是同一套东西的两面:

- 「插件往 `ctx` 挂服务」的**类型侧**,是第 14 章的 `declare module` 合并 `Context`。
- 「插件新增事件」的**类型侧**,是 `declare module` 合并 `Events`,以及第 15 章 `dispatch.ts` 从 `Events` 推导派发器。
- 「工位按名查找、可替换」对应「依赖抽象接口而非具体实现」(第 24 章会讲 Service Definition / Provider / Consumer)。

## 三种插件形态

一个抽象,三种写法(详见 [`cordis-tutorial/01-first-plugin.md`](../cordis-tutorial/01-first-plugin.md)):

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// 1. 函数插件:最常用
export function apply(ctx: Context) {}

// 2. 对象插件:带 apply 方法的对象
export const myPlugin = {
  name: 'my-plugin',
  apply(ctx: Context) {},
}

// 3. 类插件:Service 子类,用来「提供工位」
export class MyService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'myService')
  }
}
```

用到「提供工位」时才需要类形态;普通插件用函数即可。

## 一个最小插件的样子

```ts
import type { Context } from '@deepseek-ai/cordis'

export const name = 'hello'
export function apply(ctx: Context) {
  console.log('hello from my first plugin')
}
```

配套一个 `cordis.yml`(插件清单),Loader 按它把插件一个个装进工厂。`import type { Context }` 是第 2 章学过的「只在类型位置引用」,编译后不产生真实依赖。

## 一条贯穿全程的主线

从这一章往后,你会反复看到同一个结构:**「插件通过注册(运行时) + 声明合并(类型)」把能力挂进共享上下文**。

记住这张图,后面 6 章都在填充它的细节:

```text
                 插件 A          插件 B          插件 C
                  │              │              │
      注册(运行时)+声明合并(类型) │              │
                  ▼              ▼              ▼
              ┌───────────────────────────────┐
              │        共享 Context            │
              │  服务: llm / tools / agents … │
              │  事件: agent/* / tools/* …    │
              └───────────────────────────────┘
```

## 你现在能自己读了

1. 通读 [`cordis-primer.md`](../cordis-primer.md) 全文(约 44 行)。它现在对你已经是「熟悉词汇的回放」而非天书——尤其「Dispatch Modes」那张表,和第 22 章会详细展开的派发模式一一对应。

2. 如果愿意动手,过一遍 [`cordis-tutorial/`](../cordis-tutorial/index.md) 的前 4 章(它们可运行、无需 API key)。用本教程的 TS 知识去读那些示例的 `declare module` 和 `inject`,你会发现自己已经能一眼看懂。

## 小结

- Cordis = 插件 + Context + 服务 + 事件 + 可逆注册;五个概念对应你学过的五块 TS 特性。
- 一个抽象三种写法;需要提供工位才用 Service 类。
- 「注册(运行时)+ 声明合并(类型)」是理解后续所有章节的主线。

## 思考题

1. 用自己的话说:插件之间「不 import 对方,靠 `ctx.xxx` 找到彼此」,带来什么好处?(提示:可替换性。)
2. `impl type` 和 `declare module` 在「插件化」里分别扮演什么角色?一个管运行时,一个管什么?
3. 「工位按名查找」为什么让「换一个 provider 就换掉整个产品」成为可能?

---

上一章:[第 19 章 · 项目的类型规范速览](19-project-type-conventions.md)
下一章:[第 21 章 · Service 与依赖注入的真实实现](21-services.md)
