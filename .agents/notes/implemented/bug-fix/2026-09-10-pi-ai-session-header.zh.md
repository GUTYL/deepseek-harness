# Agent Note: pi-ai routes send the Harness session header

Status: implemented

[English](2026-09-10-pi-ai-session-header.md) | 中文

## Problem

OpenCode Go 要求在它认可的某个请求头中携带每个对话的稳定 id。直连 DeepSeek adapter 会从 `GenerateOptions.sessionId` 发送 `x-deepseek-harness-session-id`，但经由 pi-ai adapter 的路由完全不发送任何会话元数据：adapter 只把 `sessionId` 传给 pi-ai，而 pi-ai 的亲和头（`x-session-affinity`、`session_id`、`x-client-request-id`）仅在目录条目显式开启时才发送，且都不是 OpenCode Go 指名的那一个。因此经 pi-ai 路由的对话到达 OpenCode Go 时没有可识别的会话 id，以 `MissingSessionID` 失败。

## Decision

pi-ai adapter 把实时的 `GenerateOptions.sessionId` 以两个模型不可见的传输头并入每个提供方请求：`x-deepseek-harness-session-id`（直连 DeepSeek adapter 已发送的头）与 `x-opencode-session`（OpenCode Go 用于路由与缓存的头）。两者与路由配置的 `headers` 和共享的 Harness 归属信息并列；任一名的同名配置头都无法取代实时 id，保留的归属名仍赢得各自的冲突。pi-ai 自身的会话亲和中转保持不变。

## Alternatives considered

**用静态路由 `headers` 条目打戳。** 部署可以写 `headers: { x-deepseek-harness-session-id: … }`，但常量 id 会把所有对话混为一谈，违背提供方所述的按对话路由与 prompt 缓存目的，也无法跟踪 loop 的身份。

**暴露 pi-ai 的 `sendSessionAffinityHeaders` 与 `sessionAffinityFormat` compat 开关。** 它们发送的是 `x-session-affinity`/`session_id`，不是 OpenCode Go 指名的头，而 compat 门禁刻意withhold pi-ai 目录为具名厂商判定的字段。开启它们仍会漏掉被认可的头。

**新增通用的 `sessionHeader` 配置字段。** 这是第二种命名“adapter 自己能打戳什么”的方式，除当前这一个提供方外没有消费者，而且默认会让请求不带头。

**对 `opencode-go` 提供方做特判。** 该会话头就是直连 adapter 在每个 DeepSeek 请求上都会发送的同一份元数据；按提供方 id 设限会让下一个认可 Harness 会话头的提供方仍然得不到服务。

## Consequences

每个 pi-ai 路由现在都以这两个头携带一个不透明的 Harness 会话 id 作为模型不可见的传输元数据；两个都不用的提供方会忽略未知头。OpenCode Go 会在它指名的头中看到稳定 id，从而按对话路由与缓存，而不是以 `MissingSessionID` 失败。该值跟随 loop 的身份，因此分叉或恢复的对话会保留其请求所携带的身份。

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` 固定了该头在带 session id 时存在、不带时不存在，并会胜过同名的 profile 头；直连 adapter 自己的套件已固定了孪生的那个头。
