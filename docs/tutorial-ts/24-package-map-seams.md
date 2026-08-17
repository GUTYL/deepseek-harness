# 第 24 章 · 核心包地图与能力接缝(seam)

## 本节要学的概念

- 「能力接缝(seam)= Service Definition / Service Provider / Consumer」三件套。
- 核心包地图:每个包「拥有」什么,对应哪个 `ctx` 键。
- 为什么「换 provider 就换掉整个产品」是 seam 的威力。

## 核心包地图

程序由几十个包组成(完整清单见 [`packages/README.md`](../../packages/README.md))。最核心的几个,以及它们在 `ctx` 上「拥有」的工位,来自 [`architecture.md`](../architecture.md):

| 包 | 拥有 | `ctx` 键 |
|---|---|---|
| `core/session` | 只追加的 `SessionEvent` 日志 + 内存存储 | `ctx.sessions` |
| `core/system-prompt` | 提示词片段 + 工具 schema 的组装 | `ctx.systemPrompt` |
| `core/tools` | 作用域化的工具注册表 + 受守卫的执行管道 | `ctx.tools` |
| `core/agent` | `Agent` 接口、实时注册表、`agent/*` 事件 | `ctx.agents` |
| `core/agent-loop` | 实现该接口的默认驱动 | `ctx.agentLoop` |
| `core/scope` | 每个 agent 的作用域注册原语 | 库,无键 |
| `llm/llm` | 消息与流词汇表 + 适配器 seam | `ctx.llm` |

读这张表,结合第 21 章:

- 每个包「拥有」一个工位,是在 `extends Service` + `super(ctx, key)` + `declare module` 里完成的。
- 注意 `core/agent` vs `core/agent-loop` 的一行之差,这是个关键设计(下面讲)。

## 能力接缝(seam):三件套

上一章末尾埋的「seam」,这里正式展开。一个**能力接缝(seam)** 由三个角色组成(定义见 [`architecture.md`](../architecture.md) 的「Capability seams」和 [`capability-seams.md`](../capability-seams.md)):

1. **Service Definition(服务定义)**:声明「这个能力的接口长什么样」——类型契约。
2. **Service Provider(服务提供者)**:实现这个接口——具体的干活逻辑。
3. **Consumer(消费者)**:使用这个接口——通常是面向模型的工具。

用 `llm` 举例,三个角色分别是:

- **Definition**:`ctx.llm` 的接口(第 23 章的词汇表 + 适配器注册接口)。
- **Provider**:DeepSeek 适配器(把厂商格式翻译进词汇表)。
- **Consumer**:agent-loop(第 25 章,消费 `ctx.llm.stream(...)`)。

用 `shell`(bash 能力)再举例:Definition 是「执行 shell」的接口,Provider 是「本地/远程沙箱里的实现」(它内部还会通过 `ctx.subprocess` 开进程),Consumer 是模型能调用的 `shell` 工具。

**seam 的核心纪律**(根 AGENTS.md):

> **一个能力 seam 包括三个角色,缺一不可;只有当角色各自独立演化时才拆分。** 加入新能力 = 设计好三件套,而不是只写一个孤零零的工具。

## seam 的威力:换一个 provider,换掉整个产品

为什么费心做三件套?因为**provider 可以整体替换**。

架构文档的原话(节选):

> 「seam 是为什么换一个 provider 就换掉整个产品。文件系统和子进程的 provider 共享同一个执行世界,于是把它们指向一个远程沙箱,Bash、PTY、LSP 就跟着一起搬,不需要为每个 provider 写分支。」

关键点:因为消费者依赖的是**抽象的 Definition**,而不是**具体的 Provider**,所以:

```text
消费者(agent-loop / 工具)  →  只认 Definition 接口
                                    ▲
                     Provider A(本地)   Provider B(远程沙箱)  ← 可整体替换
```

第 21 章讲的「`inject` 依赖持续跟踪、provider 换掉则消费者重启」,在这里派上用场:换掉 fs/subprocess 的 provider,所有依赖它们的消费者干净地重启到新实现上。

## 回看:core/agent vs core/agent-loop

现在理解为什么 `core/agent` 和 `core/agent-loop` 要分成两个包:

- `core/agent` 是 **Definition** 的一半——它定义 `Agent` **接口**(能力长什么样),还有 `agent/*` 事件。
- `core/agent-loop` 是 **Provider**——它实现了 `Agent` 接口的默认驱动 `ReactLoopAgent`(第 25 章的主角)。

这样,「主循环」本身也成了一个**可替换的 provider**:你写 UI、hook、工具时,依赖的是 `dsh-agent` 的抽象 `Agent` 接口,而不是 `dsh-agent-loop` 的具体实现。换一个「环路驱动」,只要实现同样的 `Agent` 接口即可。

root `packages/README.md` 明确写了这条依赖纪律:

> 「扩展插件依赖 Service Definition,绝不依赖具体 provider。`dsh-agent-loop` 是可替换的;UI、hook、工具插件用 `dsh-agent`。」

## 你现在能自己读了

1. 通读 [`architecture.md`](../architecture.md) 的「Capability seams」一节(约 8 行),对照本教程之前的讲解,确认每个词都懂了。

2. 打开 [`packages/core/agent/src/types.ts`](../../packages/core/agent/src/types.ts),这是 `Agent` **接口**的定义。先不要细读方法,只看它 `export interface Agent { ... }` 的整体形状——它就是你刚学的「Definition 的角色」:一个纯接口,不含实现。

3. 再打开 [`packages/core/agent-loop/src/agent.ts`](../../packages/core/agent-loop/src/agent.ts) 顶部,看 `export class ReactLoopAgent implements Agent`——`implements Agent` 这五个字,正是「Provider 实现 Definition」的 TS 表达(第 25 章会全文读它)。

## 小结

- 核心包各有其 `ctx` 工位;`core/agent`(定义)vs `core/agent-loop`(实现)是 seam 的典型例子。
- seam = Service Definition / Provider / Consumer 三件套;消费者依赖定义,不依赖具体 provider。
- 「换 provider 就换掉整个产品」:抽象定义让整体替换成为可能。

## 思考题

1. 用自己的话复述 seam 三件套,并给 `llm` 和 `shell` 各举三件套。
2. 为什么消费者「依赖抽象 Definition」是「可替换」的前提?如果消费者直接 import 具体 provider,会怎样?
3. `ReactLoopAgent implements Agent` 里的 `implements`,在 TS 里是什么含义?它和「结构类型(第 5 章)」是什么关系?(提示:implements 是显式声明契约,但 TS 仍是结构校验。)

---

上一章:[第 23 章 · LLM 词汇表:消息与流](23-llm-vocabulary.md)
下一章:[第 25 章 · Agent 主循环:turn/step 状态机](25-agent-loop.md)
