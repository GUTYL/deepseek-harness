# 第 21 章 · Service 与依赖注入的真实实现

## 本节要学的概念

- Service 的「提供」与「消费」,以及 `inject` 的依赖声明。
- `ctx.plugin`、`ctx.get`、`ctx.<key>` 的区别。
- 看一个真实的 Service 类,把前面所有 TS 知识用起来读它。

## 提供工位:一个 Service 类

第 20 章说了,「提供工位」用 Service 类。回到 [`cordis-tutorial/03-services.md`](../cordis-tutorial/03-services.md) 的最小例子,用你的 TS 知识重读:

```ts
import { Service, type Context } from '@deepseek-ai/cordis'

// ① 类型层:声明合并,给 Context 加一个 greeter 属性(第 14 章)
declare module '@deepseek-ai/cordis' {
  interface Context {
    greeter: GreeterService
  }
}

// ② 运行时:这个类就是一个工位
export class GreeterService extends Service {
  constructor(ctx: Context) {
    super(ctx, 'greeter')   // 注册成名为 'greeter' 的工位(第 3 章的 super)
  }
  greet(who: string) {
    return `Hello, ${who}!`
  }
}

export const name = 'greeter'
export function apply(ctx: Context) {
  ctx.plugin(GreeterService)   // 把它当插件装进工厂
}
```

两半分工,你已完全清楚:

- **类型层** `declare module`(第 14 章):把 `greeter` 加进 `Context` 接口,让 `ctx.greeter` 全项目可类型检查。**运行时无事发生。**
- **运行时** `super(ctx, 'greeter')` + `ctx.plugin(...)`:`greeter` 正式注册进上下文。**类型层不感知。**

两者靠「名字 `greeter` 一致」连起来——这就是第 14 章说过的「插件化的两条腿」。

## 消费工位:`inject` 声明依赖

```ts
export const name = 'consumer'
export const inject = ['greeter']   // 声明:我需要 greeter 工位

export function apply(ctx: Context) {
  console.log(ctx.greeter.greet('world'))
}
```

`inject = ['greeter']` 的含义(第 20 章的表里提过):**「我依赖 greeter,它没就绪前别启动我。」**

由此推出两个重要结论:

1. **加载顺序由依赖决定,不靠文件顺序**:`cordis.yml` 里 consumer 写在前面还是后面无所谓,Cordis 会等它的依赖先就绪,再调它的 `apply`。这是「声明式」取代「手动排序」。
2. **依赖是持续跟踪的,不是一次性检查**:如果 `greeter` 工位在运行中消失(provider 被卸载或热替换),依赖它的 consumer 也会被卸载,等 greeter 回来再重启。这让「换 provider」变得干净:配置文件里把 `dsh-bash-local` 换成一个新的 shell provider,所有 `inject: ['shell']` 的插件都自动重启到新实现上。

## 三种取工位的方式,别混

- `ctx.<key>`(如 `ctx.llm`):当你**声明了** `inject: ['llm']`,可以在 `apply` 里直接 `ctx.llm` 取值。这个属性代理和拓扑有关(根 AGENTS.md 提到它「topology-sensitive」)。
- `ctx.get(name)`:**严格**地从全局服务仓库读「名为 name 的服务」,用于**可选依赖**——你没 `inject` 它、想用就用。
- `ctx.plugin(Service)`:把一个服务**挂载**为插件(提供方用)。

关键区别在**可选依赖**上(见 [`cordis-tutorial/03-services.md`](../cordis-tutorial/03-services.md) 的「Optional dependencies」):

```ts
export function apply(ctx: Context) {
  // 可选依赖:不 inject,用 ctx.get 探测
  const greeter = ctx.get('greeter')   // 没有 provider 时是 undefined,插件照常运行
  console.log(greeter?.greet('maybe') ?? 'no greeter available')
}
```

硬依赖用 `inject`,软依赖用 `ctx.get` + 可选链 `?.` + `??`(这些是 TS 的基本语法,你已经会)。

## 看一个真实的 Service:默认模型配置

现在读一个真实的 Service,检验你的 TS 阅读能力。打开 [`packages/core/agent-default-model/src/index.ts`](../../packages/core/agent-default-model/src/index.ts):

```ts
export class AgentDefaultModelConfig extends Service {
  // ...
  constructor(ctx: Context) {
    super(ctx, 'agentDefaultModel')
  }
  // ...
}
```

对照前面的最小例子,结构完全一致:`extends Service` + `constructor(ctx)` 里 `super(ctx, 'name')` 注册工位。这个类要「提供」的是一个叫 `agentDefaultModel` 的工位(负责「agent 的默认模型配置」)。

再看它的类型层(同文件或同目录):

```ts
declare module '@deepseek-ai/cordis' {
  interface Context {
    agentDefaultModel: AgentDefaultModelConfig   // 类型层声明这个工位存在
  }
}
```

**这就是项目里几百个 Service 的统一套路**,你看到任何一个 `extends Service` 的类,都能预期:

1. 构造函数里 `super(ctx, '某某工位名')`。
2. 同文件或附近有 `declare module` 把该工位名并进 `Context`。
3. 别处在 `inject` 里声明依赖它,或 `ctx.get` 探测它。

## 你现在能自己读了

1. 打开 [`packages/core/agent/src/index.ts`](../../packages/core/agent/src/index.ts),找到 `ctx.agents` 这个工位的注册处(`extends Service` + `super(ctx, 'agents')`),以及对应的 `declare module` 合并。体会「核心能力也是一个普通 Service,一种工位」。

2. 体会项目的命名约定:工位名是**扁平命名空间**里的短名(`tools`、`llm`、`agents`),而包名是 `@deepseek-ai/dsh-*`(见 [`packages/README.md`](../../packages/README.md))。两者是不同层面:包 = 代码单元,工位 = 运行时按名查找的服务。

## 小结

- 提供工位:Service 类 + `super(ctx, name)` 注册 + `declare module` 类型声明。
- 消费工位:`inject` 硬依赖;`ctx.get` 软依赖(可选);`ctx.plugin` 是挂载。
- 依赖由 Cordis 持续跟踪,provider 换掉则消费者自动重启。

## 思考题

1. `inject: ['greeter']` 和「在 apply 里 `ctx.get('greeter')`」的区别是什么?各自适合什么场景?
2. 为什么「依赖由 Cordis 跟踪」能让「换 provider」成为热替换?如果依赖是一次性检查会怎样?
3. 在 [`packages/core/agent/src/index.ts`](../../packages/core/agent/src/index.ts) 里,`ctx.agents` 的类型声明和运行时注册,分别靠什么语法?各出在哪几行?

---

上一章:[第 20 章 · 心智模型:一切皆插件](20-cordis-model.md)
下一章:[第 22 章 · 事件系统与五种投递模式](22-events.md)
