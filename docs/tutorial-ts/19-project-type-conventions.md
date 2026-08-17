# 第 19 章 · 项目的类型规范速览

## 本节要学的概念

- 把前 18 章散落的「项目类型规矩」收拢成一张速查表。
- 理解「这些规范背后都在守什么」——纪律不是教条。

## 为什么单独一章

前 18 章里,我反复引用过根目录 [`AGENTS.md`](../../AGENTS.md) 的某些约定:「封闭联合以 assertNever 结束」「残余 any 要解释」「Branded 只用于跨边界 id」… 这一章把它们**归拢成一张总表**,并解释每条背后的动机。你之后读代码、写代码,都能拿它当「守则」。

## 类型规范速查表

| 规范 | 出处章节 | 背后的动机 |
|---|---|---|
| 全仓 `strict: true` + `noImplicitAny` | 第 2 章 | 类型检查真正生效的前提;不「假装有类型」 |
| 关闭联合以 `assertNever` 收尾;可扩展联合落到文档化 `default` | 第 7 章 | 新增分支时编译期报警,而不是运行期漏掉 |
| 判别联合用字面量 `kind`/`type` 当标签 | 第 7 章 | 唯一确定成员,支撑 switch 收窄与穷尽 |
| 扩展类型用 `interface` + `declare module` 合并 | 第 14 章 | 插件能扩展共享类型词汇表,是「一切皆插件」的类型层 |
| 跨边界 id 用 `Branded<B>` 品牌,配工厂函数 | 第 17 章 | 形状相同但语义不可混的 id,类型上强制区分 |
| 外部边界用 `unknown` + 类型谓词/断言守卫 | 第 16/18 章 | 不可信输入必须收窄后才操作 |
| 残余 `any` 必须解释或钉回 `unknown` | 第 18 章 | 不让 any 泄漏,守住类型安全的「洞」 |
| 只用于类型的位置用 `import type` | 第 2 章 | 类型编译后消失,不引入运行时依赖 |
| 函数导出显式返回类型 + `@param`/`@returns` | 第 4 章 | 免费用编译器校验自己,并给出契约 |

## 两条「总纲」:一切从两个原则推出来

细看这张表,会发现所有规范几乎都从**两条总原则**推出。理解这两条,比背表更有用。

### 原则一:类型安全要「诚实」

「strict 全开」「any 要解释」「unknown 先收窄」,背后是同一个态度:**类型系统要么认真用,要么别假装用了**。关掉 strict、随手 any、跳过收窄,都会让「TS 能救你」变成一句空话。

### 原则二:类型跟着「可扩展」走

「interface 合并」「Map + keyof 联合」「Branded 的节制」,背后是项目的核心架构——**一切皆插件**。插件要能扩展服务、扩展事件、扩展类型词汇表,于是类型系统被设计成「开放的、可合并的」;而真正不该扩展、不该混淆的边界(跨包 id),才用品牌类型「封死」。

**一句话总结本章:这个项目的类型规范,一半在守「诚实」,一半在守「可扩展」。**

## 你现在能自己读了

回到[根 AGENTS.md](../../AGENTS.md) 的「Conventions」小节,逐条读出哪些是「类型规范」(而不是构建/文档/测试规范)。你会发现,本教程 1~18 章其实就是在给这一小节里的每一条,配上「它为什么、它长什么样」的解释。

尤其注意这几条,你现在能完全读懂它们的含义了:

- 「**Switch on discriminant tags.** Closed unions end in `assertNever`; merge-extensible unions fall through a documented default.」
- 「**Opaque cross-boundary ids are branded** (`Branded<B>` from `dsh-brand`), never bare `string`.」
- 「**Trust TypeScript at typed same-process boundaries.** …validate at parser/config, queued, model/tool JSON, durable/file, worker, process, and wire boundaries.」(这条对应第 16/18 章:同进程内的类型信任 TS,跨边界的值要验证。)

## 小结

- 项目的类型规范可归拢成一张表,全部由「诚实」和「可扩展」两条总纲推出。
- 读根 AGENTS.md 的 Conventions 时,把每一条对应到本教程某章,你就真正内化了。

## 思考题

1. 从「可扩展」这条总纲,解释为什么「扩展类型用 interface 合并」而「跨边界 id 用 Branded 封死」——这两者看似相反,实则一致,为什么?
2. 「信任同进程内的类型」和「验证跨边界值」,分别对应本教程哪几章?
3. 如果一个新人给你一段「用 any 写、没开 strict」的代码,你打算怎么用本教程第 16/18 章的知识去帮他收紧?

---

上一章:[第 18 章 · never / unknown / any 的三分法](18-never-unknown-any.md)
下一章:[第 20 章 · 心智模型:一切皆插件(Cordis)](20-cordis-model.md)
