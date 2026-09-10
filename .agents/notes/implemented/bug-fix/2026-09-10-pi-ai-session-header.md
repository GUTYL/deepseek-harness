# Agent Note: pi-ai routes send the Harness session header

Status: implemented

English | [中文](2026-09-10-pi-ai-session-header.zh.md)

## Problem

OpenCode Go requires a stable per-conversation id in a header it recognizes. The direct DeepSeek adapter sends `x-deepseek-harness-session-id` from `GenerateOptions.sessionId`, but a route served through the pi-ai adapter sent no session metadata at all: the adapter passed `sessionId` into pi-ai, whose affinity headers (`x-session-affinity`, `session_id`, `x-client-request-id`) are emitted only when a catalog entry opts in and are not the header OpenCode Go names. A conversation routed through pi-ai therefore reached OpenCode Go without a recognized session id and failed with `MissingSessionID`.

## Decision

The pi-ai adapter merges the live `GenerateOptions.sessionId` into every provider request under two model-hidden transport headers: `x-deepseek-harness-session-id`, which the direct DeepSeek adapter already sends, and `x-opencode-session`, the name OpenCode Go routes and caches on. Both sit beside the route's configured `headers` and the shared Harness attribution; a configured header of either name cannot displace the live id, and a reserved attribution name still wins its collision. pi-ai's own session-affinity pass-through is unchanged.

## Alternatives considered

**Stamp it with a static route `headers` entry.** A deployment could write `headers: { x-deepseek-harness-session-id: … }`, but a constant id conflates every conversation, defeating the routing and prompt-cache purpose the provider states, and it cannot track the loop's identity.

**Expose pi-ai's `sendSessionAffinityHeaders` and `sessionAffinityFormat` compat switches.** They emit `x-session-affinity`/`session_id`, not the header OpenCode Go names, and the compat gate deliberately withholds the fields pi-ai's catalog determines for a named vendor. Enabling them would still miss the recognized header.

**Add a generic `sessionHeader` config field.** A second way to name what the adapter can stamp itself, with no current consumer beyond this one provider, and it would leave the request headerless by default.

**Special-case the `opencode-go` provider.** The session header is the same metadata the direct adapter sends on every DeepSeek request; gating it by provider id would leave the next provider that recognizes the Harness session header unserved.

## Consequences

Every pi-ai route now carries an opaque Harness session id under both headers as model-hidden transport metadata; a provider that uses neither ignores unknown headers. OpenCode Go sees a stable id in the header it names and routes and caches per conversation instead of failing `MissingSessionID`. The value follows the loop's identity, so a forked or resumed conversation keeps the identity its requests carry.

## Testing

`packages/llm/llm-pi-ai/tests/adapter.spec.ts` pins the header present with a session id, absent without one, and winning over a same-named profile header; the direct adapter's own suite already pins the twin header.
