# 第 23 章 · LLM 词汇表:消息与流

## 本节要学的概念

- 「提供者中立」的类型词汇表:把各家模型厂商的差异,收编成一套统一的类型。
- 内容块(ContentBlock)、流块(StreamChunk)、生成请求(GenerateOptions)如何用判别联合表达。
- 这套词汇表是「能力接缝(seam)」在 LLM 上的体现。

## 为什么需要「词汇表」

第 21、22 章讲了 Services 和 Events 的「运行机制」。这一章看一个具体的**数据模型**——`dsh-llm` 定义的那套「消息和流」类型。

问题背景:agent 要调用模型,但不同厂商(DeepSeek、OpenAI…)的接口长得不一样。如果 agent-loop 直接对着某家厂商的格式写,换一家就得重写。

解法:**在中间定一套「厂商无关」的类型词汇表**,厂商差异交给「适配器(adapter)」去翻译。agent-loop 只认这套统一词汇表,不认厂商。这就是 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts) 文件开头的注释所说的:「canonical provider-neutral message and streaming vocabulary for the loop, session log, and plugins. Adapters alone translate provider wire messages.」

这套词汇表,全部用你学过的 TS 特性写成。

## 内容块:可扩展的判别联合

第 5、7、10 章你反复见过它,现在完整地把它读作「一个模型消息由什么块组成」:

```ts
export interface ContentBlockMap {
  'text': TextBlock
  'reasoning': ReasoningBlock
  'image': ImageBlock
  'tool-call': ToolCallBlock
  'tool-result': ToolResultBlock
}
export type ContentBlockType = keyof ContentBlockMap
export type ContentBlock = ContentBlockMap[ContentBlockType]
```

- `ContentBlock`(第 7 章的可扩展联合)= 文本块 / 推理块 / 图片块 / 工具调用块 / 工具结果块。
- 每个块用 `type` 字段判别(第 7 章的判别标签)。
- 「可扩展」:插件能往 `ContentBlockMap` 加新键(第 14 章)。

具体到单个块,读一个(第 5 章见过):

```ts
export interface ToolCallBlock {
  type: 'tool-call'
  id: CallId            // 品牌类型(第 17 章)
  name: string
  arguments: string     // 模型产出的原始 JSON 字符串
}
```

注意 `id: CallId` 是第 17 章的品牌类型——工具调用的关联 id 不能和会话 id 混用,所以打了品牌。

## 流:模型输出是「流」出来的

模型不会一次性吐完答案,而是一块一块流式输出。`StreamChunk` 用一个判别联合描述这条流里的每一种事件(第 8 章见过它的摘要):

```ts
export type StreamChunk =
  | { type: 'block-start'; index: number; blockType: ContentBlockType }
  | { type: 'text-delta'; index: number; text: string }
  | { type: 'reasoning-delta'; index: number; text: string }
  | { type: 'tool-call-delta'; index: number; id: CallId; name?: string; argumentsDelta: string }
  | { type: 'block-end'; index: number; block: ContentBlock }
  | { type: 'usage'; usage: TokenUsage }
  | { type: 'finish'; reason: FinishReason; replayState?: ReplayEnvelope }
```

读这段时,把每个 `type` 当判别标签:

- `block-start` / `block-end`:一个内容块开始/结束(中间夹着增量)。
- `*-delta`:块的增量(文本增量、推理增量、工具调用参数增量)。
- `usage`:token 用量。
- `finish`:结束,`reason` 是为什么结束(`FinishReason`,又是第 7 章的可扩展联合)。

配套的类型也都用判别联合表达,比如「为什么停止」:

```ts
export interface FinishReasonMap {
  'stop': { kind: 'stop' }
  'tool-calls': { kind: 'tool-calls' }
  'max-tokens': { kind: 'max-tokens' }
  'aborted': { kind: 'aborted'; failure: LlmFailure }
  'error': { kind: 'error'; failure: LlmFailure }
}
export type FinishReason = FinishReasonMap[keyof FinishReasonMap]
```

`FiniteReason` 是第 10 章「Map + keyof」配方、第 7 章「可扩展判别联合」的又一实例。`LlmFailure`(第 5 章)是「可序列化的失败事实」。

## 生成请求:一次完整调用的装配

`GenerateOptions`(第 5 章有过几段)是「一次完整的模型请求」的接口:

```ts
export interface GenerateOptions {
  provider: string
  model: string
  reasoningEffort?: ReasoningEffortId   // 品牌类型(第 17 章)
  messages: Message[]                   // 有序对话消息(模型视角)
  system?: string
  tools?: ToolSchema[]                  // 工具 schema
  temperature?: number
  maxTokens?: number
  stop?: string[]
  signal?: AbortSignal
  sessionId?: Branded<'SessionId'>      // 品牌类型
  purpose?: 'compaction' | 'session-title'
}
```

逐类看,几乎每个字段都呼应前几章:

- `provider`/`model` 是**路由**:选哪个适配器、哪个模型(第 24 章的 seam 路由)。
- `reasoningEffort?`、`sessionId?` 是**品牌类型**(第 17 章)——「推理强度 id」「会话 id」都是跨边界、易混淆的 id。
- `messages: Message[]` 是第 8 章的数组。
- `purpose?` 是第 6 章的联合字面量(`'compaction' | 'session-title'`)。

这正是一个「参数全部类型化」的接口:把「调模型」这件大事,精确描述成一份可检查的类型。

## 把「词汇表」和「seam」接起来

第 24 章会正式讲「能力接缝(seam) = Service Definition / Provider / Consumer 三件套」。这里先埋一句:`dsh-llm` 的这套词汇表,就是 **「调用大模型」这个 seam 的「服务定义」层的一半**——它定义了「消息长什么样、流长什么样、请求长什么样」的**类型契约**,而真正的 DeepSeek 适配器(provider)负责把厂商格式翻译进来,agent-loop(consumer)消费它。

于是「换模型厂商」=「换一个适配器」,词汇表和 agent-loop **都不动**。这就是前面反复强调的「可替换性」在数据模型上的落地。

## 你现在能自己读了

1. 打开 [`packages/llm/llm/src/types.ts`](../../packages/llm/llm/src/types.ts) 全文通读一遍。现在你能认出:哪些是判别联合(`ContentBlock`/`StreamChunk`/`FinishReason`)、哪些是品牌类型(`CallId`/`ReasoningEffortId`)、哪些是普通接口(`GenerateOptions`/`LlmFailure`)。

2. 重点看 `StreamChunk` 那段 JSDoc 注释,理解适配器的职责:「Adapters emit usage before the terminal finish and nothing afterward; tool arguments remain raw JSON strings.」——它精确说明了「谁负责什么」。

## 小结

- LLM 词汇表是「厂商无关」的统一类型,厂商差异交给适配器翻译。
- `ContentBlock`/`StreamChunk`/`FinishReason` 都是可扩展判别联合(第 7/10 章配方)。
- `GenerateOptions` 是一次完整请求的精确类型,字段大量用品牌类型和联合字面量。

## 思考题

1. 为什么「工具调用的参数」在 `ToolCallBlock` 里是 `arguments: string`(原始 JSON 字符串),而不是解析好的对象?提示:想想模型输出的是什么、谁负责解析。
2. `StreamChunk` 用判别联合表达「流里的各种事件」,和一个「统一的、带可选字段的接口」相比,好处是什么?(提示:判别联合表达的是「互斥的一种」,不会是「可能同时有几种」。)
3. 为什么 agent-loop 只依赖这套词汇表、不依赖具体厂商?这给「换模型」带来什么具体好处?

---

上一章:[第 22 章 · 事件系统与五种投递模式](22-events.md)
下一章:[第 24 章 · 核心包地图与能力接缝](24-package-map-seams.md)
