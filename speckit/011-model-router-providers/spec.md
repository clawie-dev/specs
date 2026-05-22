# 011 — Model Router & Provider Abstraction

**Status:** Draft
**Constitutional anchor:** Principle X (Pluggable)
**Depends on:** 003, 004, 006, 007
**Required by:** 008, 016

## Purpose

The control plane routes every model call through a router that abstracts providers, applies fallback policy, enforces budget, applies rate-limit policy, and records cost. No provider SDK is imported directly by an agent.

## Functional requirements

| ID | Requirement |
|---|---|
| 011-FR-001 | Provider adapters MUST implement a stable interface: `chat`, `complete`, `embed`, `stream`, `count_tokens`. |
| 011-FR-002 | Built-in providers in v1: Anthropic, OpenAI, Gemini, OpenRouter, Ollama (local), llama.cpp (local). Others installable as plugins. |
| 011-FR-003 | Each agent's `MODEL.yaml` MUST declare a primary provider/model + ordered fallback list. |
| 011-FR-004 | The router MUST attempt primary, then fall back on: rate-limit, timeout, 5xx, content-policy-rejection. Fall back is NOT automatic on cost-cap; that aborts. |
| 011-FR-005 | The router MUST emit cost entries to the ledger before returning a completion to the agent. |
| 011-FR-006 | The router MUST honor budgets per spec 007; over-budget calls are rejected with `cause=budget_exceeded`. |
| 011-FR-007 | The router MUST honor permissions: if a model is denied for an agent (e.g., "no Claude" or "only local"), the call fails fast. |
| 011-FR-008 | Prompt caching support: when the provider supports it, the router MUST manage cache control headers automatically. |
| 011-FR-009 | Streaming MUST be supported and surfaced via the WebSocket API. |
| 011-FR-010 | The router MUST gracefully degrade: a degraded provider (slow but responsive) is preferred over a hard fail of the chain. |
| 011-FR-011 | Credentials MUST come from the credential broker (spec 012), never from the agent container env. |
| 011-FR-012 | Model registry MUST be configurable: each provider has a list of available models with pricing metadata; updates as PRs. |

## Non-functional requirements

| ID | Requirement |
|---|---|
| 011-NFR-001 | Router overhead MUST be < 50ms per call (excluding model latency). |
| 011-NFR-002 | Fallback decision MUST be made within provider's first-error window (~5s). |

## User stories

- **011-US-001 [P0]** — As an operator, my Claude key rate-limits; the next call falls back to OpenAI, audit reflects both.
- **011-US-002 [P0]** — As an agent, I never see API keys; I call `model.chat({...})` and credentials are injected by the broker.
- **011-US-003 [P1]** — As an operator, I configure a team to "local models only", and the router refuses paid calls.
- **011-US-004 [P1]** — As an operator, I add a custom provider plugin (e.g., Mistral, Together) without modifying core.

## Acceptance criteria

- Fallback triggers on rate-limit/timeout/5xx within seconds.
- Cost ledger sees every call within 10s.
- Local-only mode strictly enforced.
- Plugin provider verifiable end-to-end with a test.

## Edge cases & failure modes

- All providers down → task fails with `cause=provider_error`, marked recoverable; will retry on next claim.
- Provider returns malformed output → router validates shape; on failure, fall back.
- Token-count estimation drift → reconciled at provider response; ledger adjusts.

## Related specs

`007-budget-governance`, `006-observability-audit-cost`, `008-agent-definition`, `012-credential-broker`.
