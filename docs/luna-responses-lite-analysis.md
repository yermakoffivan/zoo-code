# GPT-5.6 Luna Responses Lite Compatibility Analysis

## Status

This document records the findings and implementation plan for porting [OpenCode PR #36685](https://github.com/anomalyco/opencode/pull/36685) to Zoo Code. The upstream change was merged as commit [`67aa9cee23f9aac19477e714cce7db937a0e4eb9`](https://github.com/anomalyco/opencode/commit/67aa9cee23f9aac19477e714cce7db937a0e4eb9).

No implementation is included in this document.

## Target model clarification

The request was initially described as “5.5 Luna.” The exact model targeted by the upstream PR and by Zoo Code's existing model catalog is **`gpt-5.6-luna`**. This appears to be a naming mix-up rather than a scope change: GPT-5.5 is a separate catalog entry, while Luna belongs to the GPT-5.6 family.

The port must therefore use an exact model-ID check for `gpt-5.6-luna`. It must not alter requests for `gpt-5.5`, `gpt-5.6-sol`, `gpt-5.6-terra`, other Codex OAuth models, API-key providers, or possible names and aliases containing “Luna.”

## Executive summary

OpenCode PR #36685 is a transport compatibility fix for `gpt-5.6-luna` when accessed through ChatGPT OAuth. Luna expects OpenAI's Responses Lite request contract rather than the otherwise shared Codex Responses request shape. The upstream fix adapts the request body and supplies compatibility/session headers.

This is **not** a model catalog change. Zoo Code already catalogs `gpt-5.6-luna` in [`packages/types/src/providers/openai-codex.ts`](../packages/types/src/providers/openai-codex.ts), alongside Sol and Terra. The Zoo Code port should be isolated to the ChatGPT OAuth transport in [`src/api/providers/openai-codex.ts`](../src/api/providers/openai-codex.ts).

The recommended implementation is to:

1. Detect only the exact model ID `gpt-5.6-luna`.
2. Resolve one effective session ID per request: metadata `taskId` when present, otherwise the handler's session UUID.
3. Transform Luna's body once, before choosing the OpenAI SDK or manual SSE transport.
4. Reuse the same transformed body and effective session ID in both streaming paths.
5. Apply the same Luna contract to `completePrompt()`, using the handler session UUID because that path has no task metadata.
6. Preserve all existing request behavior for every non-Luna model.

## Upstream findings

### Purpose of PR #36685

The upstream PR addresses ChatGPT OAuth transport compatibility for `gpt-5.6-luna` by:

- translating the normal Responses request into the Responses Lite body contract;
- adding the session and compatibility headers expected by that contract; and
- retaining the original reasoning configuration while selecting all-turn context.

The change is narrowly transport-specific. It does not add Luna to a catalog, rename a model, introduce an alias, or change general GPT-5.5 behavior.

### Upstream portions that apply to Zoo Code

The HTTP request-body conversion, effective-session handling, and HTTP headers apply to Zoo Code's Codex OAuth provider.

### Upstream portion that does not apply

The upstream final diff also changed WebSocket metadata. Zoo Code's OpenAI Codex provider has no WebSocket request path: streaming uses the OpenAI SDK over HTTP, with a manual HTTP SSE fallback, and `completePrompt()` uses HTTP JSON. No WebSocket metadata should be added as part of this port.

### Upstream test coverage

The final upstream diff added no tests. Zoo Code should not reproduce that gap. The port should include focused unit coverage for the Luna-only body transformation, headers, session selection, transport reuse, `completePrompt()`, and non-Luna invariants.

## Current Zoo Code implementation

### Model catalog

Zoo Code already defines `gpt-5.6-luna` in [`packages/types/src/providers/openai-codex.ts`](../packages/types/src/providers/openai-codex.ts), with its context window, output limit, supported tools, image support, prompt-cache support, reasoning efforts, and other model capabilities. No catalog edit is required.

The same catalog contains distinct `gpt-5.6-sol`, `gpt-5.6-terra`, and `gpt-5.5` entries. This reinforces the need for exact matching rather than family-, prefix-, or substring-based detection.

### OAuth request paths

All relevant request paths are in [`src/api/providers/openai-codex.ts`](../src/api/providers/openai-codex.ts):

1. `createMessage()` delegates to the Responses API flow.
2. The handler formats the conversation and builds the normal Codex Responses body.
3. `executeRequest()` first attempts OpenAI SDK HTTP streaming.
4. If the SDK path fails or does not return an async iterable, `makeCodexRequest()` performs a manual HTTP SSE request with `fetch()`.
5. `completePrompt()` independently builds and sends a non-streaming HTTP JSON request.

The handler creates a UUID once for its lifetime and currently prefers metadata `taskId` in streaming request headers. This existing behavior provides the two inputs needed for the effective session ID.

### Existing headers

The provider already sends general Codex headers such as:

- `originator: zoo-code`;
- the existing `session_id` header;
- the Zoo Code `User-Agent`; and
- `ChatGPT-Account-Id` when available.

These headers must remain intact. Luna's Responses Lite headers are additive and model-gated; the implementation must not replace or remove the general Codex headers.

## Required Luna body transformation

Apply the following transformation only when `model.id === "gpt-5.6-luna"`.

### Preconditions and validation

Before transforming, validate the fields consumed by the Responses Lite adapter:

- `input` must be an array;
- `tools`, when present, must be an array; and
- `instructions`, when present, must be a string.

Do not silently reinterpret malformed values. Validation should fail before dispatch rather than sending a partially transformed request.

### Ordered transformation

Given the original body, retain references to the original `input`, `tools`, and `instructions`, then construct the Luna body in this order:

1. Recursively traverse the original input and remove **only** the `detail` property from every object whose `type` is `input_image`.
    - The traversal must work at any nesting depth, including image items inside message `content` arrays.
    - Preserve every other property on the image item.
    - Preserve every non-image value and property.
    - Prefer a non-mutating traversal so the normal body remains safe to inspect and reuse in tests.
2. Start a new input array with a developer `additional_tools` item containing the original `tools` array, or an empty array when tools were absent.
3. If the original `instructions` value is non-empty, append a developer message containing those instructions as `input_text`.
    - An empty instruction string does not add this developer message.
4. Append the recursively transformed original input after the developer items.
5. Delete the top-level `tools` and `instructions` fields.
6. Force `tool_choice` to `auto`.
7. Force `parallel_tool_calls` to `false`.
8. Set `prompt_cache_key` to the effective session ID.
9. Preserve the existing reasoning `effort` and `summary` fields.
10. Add `context: all_turns` to the reasoning configuration.

Semantically, the result is:

```text
input = [
  developer additional_tools(original tools or []),
  developer input_text(original non-empty instructions), // optional
  ...recursively transformed original input,
]

top-level tools        = removed
top-level instructions = removed
tool_choice             = auto
parallel_tool_calls     = false
prompt_cache_key        = effective session ID
reasoning               = existing effort/summary + context: all_turns
```

The port should follow the upstream Responses Lite representation for the two developer items exactly. It should not invent a Zoo Code-specific encoding.

### Reasoning edge case

Zoo Code currently includes a reasoning object only when a reasoning effort is active. The Luna adapter still needs to preserve any existing effort and summary while adding `context: all_turns`. The implementation should define the resulting reasoning object deliberately when no effort is present, rather than accidentally dropping the required context or introducing unrelated defaults.

## Effective session ID and headers

### Session selection

Resolve the effective session ID once per request:

```text
effectiveSessionId = metadata.taskId when available, otherwise handler session UUID
```

For `completePrompt()`, use the handler session UUID because that method does not receive request metadata.

The same exact value must be used for:

- the Luna body field `prompt_cache_key`;
- `session-id`;
- `x-session-affinity`; and
- the existing general Codex session header.

Using one resolved value prevents cache affinity and server session routing from diverging between body and headers.

### Luna-only headers

Add these headers only for `gpt-5.6-luna`:

| Header                                   | Value                |
| ---------------------------------------- | -------------------- |
| `session-id`                             | Effective session ID |
| `x-session-affinity`                     | Effective session ID |
| `version`                                | `0.144.0`            |
| `x-openai-internal-codex-responses-lite` | `true`               |

Retain the existing general Codex headers, including the current `session_id` spelling. The Luna-specific `session-id` is an additional compatibility header, not a rename of the existing header.

## Implementation plan

All production changes belong in [`src/api/providers/openai-codex.ts`](../src/api/providers/openai-codex.ts).

### 1. Add an exact Luna predicate

Introduce a small predicate or constant based on the exact model ID. Do not use `includes("luna")`, a GPT-5.6 family check, capability flags, or catalog descriptions.

### 2. Add a dedicated Responses Lite body adapter

Create a focused helper that accepts the already-built normal request body and effective session ID. It should:

- return the original request shape unchanged for non-Luna callers, or be invoked only after the exact-model guard;
- validate `input`, `tools`, and `instructions`;
- recursively remove only `detail` from `input_image` items;
- construct the ordered developer items and transformed input;
- remove top-level tools and instructions;
- force the required tool behavior;
- add the cache key; and
- preserve reasoning effort/summary while adding all-turn context.

Keeping this logic in one helper makes the contract directly unit-testable and avoids slightly different SDK, SSE, and completion variants.

### 3. Centralize effective-session resolution

In the streaming flow, resolve `metadata?.taskId || this.sessionId` before transport dispatch. Pass that resolved value—not separate optional `taskId` values—to body transformation and header construction.

For `completePrompt()`, resolve to `this.sessionId` and pass it through the same helpers.

### 4. Centralize header construction

Build the general Codex headers once from the model and effective session ID, then conditionally merge the four Luna-only headers. Continue adding authorization in the transport-appropriate place and `ChatGPT-Account-Id` when available.

The SDK request options and manual SSE request must receive equivalent Luna compatibility headers.

### 5. Transform before selecting SDK or fallback

Apply the Luna body adapter before calling `executeRequest()` or, at minimum, at the start of `executeRequest()` before its SDK attempt. The transformed body must be the single body object reused by:

- the OpenAI SDK HTTP streaming call; and
- `makeCodexRequest()` when the SDK attempt falls back to manual SSE.

Do not transform independently inside both transports. A shared transformation prevents duplicated developer items or mismatches when the SDK fails after receiving the body.

### 6. Cover `completePrompt()`

After `completePrompt()` builds its non-streaming body, apply the same Luna transformation with the handler session UUID and add the same Luna-only headers. Keep `stream: false`; Responses Lite adaptation must not accidentally convert this path to streaming.

### 7. Preserve non-Luna behavior

For every model other than the exact Luna ID:

- preserve the current top-level `tools` and `instructions`;
- preserve existing tool-choice and parallel-tool settings;
- do not add `prompt_cache_key` as part of this port;
- do not add the four Luna-only headers; and
- preserve the existing reasoning shape and request paths.

## Test plan

Add focused tests in [`src/api/providers/__tests__/openai-codex.spec.ts`](../src/api/providers/__tests__/openai-codex.spec.ts) and/or [`src/api/providers/__tests__/openai-codex-native-tool-calls.spec.ts`](../src/api/providers/__tests__/openai-codex-native-tool-calls.spec.ts). The former is a natural home for body-shape tests; the latter already covers SDK, fetch fallback, and completion headers.

### Test matrix

| Area             | Case                                                  | Expected result                                                                                     |
| ---------------- | ----------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Model gate       | Exact `gpt-5.6-luna`                                  | Responses Lite body and headers are applied                                                         |
| Model gate       | `gpt-5.5`                                             | Request remains unchanged by the Luna adapter                                                       |
| Model gate       | `gpt-5.6-sol` and `gpt-5.6-terra`                     | Request remains unchanged by the Luna adapter                                                       |
| Model gate       | Similar/alias-like Luna string                        | No Luna behavior unless it is the exact catalog ID                                                  |
| Body validation  | `input` is not an array                               | Fails before dispatch                                                                               |
| Body validation  | Present `tools` is not an array                       | Fails before dispatch                                                                               |
| Body validation  | Present `instructions` is not a string                | Fails before dispatch                                                                               |
| Tool wrapper     | Tools are present                                     | First developer `additional_tools` item contains the original tools                                 |
| Tool wrapper     | Tools are absent                                      | First developer `additional_tools` item contains `[]`                                               |
| Instructions     | Non-empty instructions                                | Developer `input_text` message appears before original input                                        |
| Instructions     | Empty instructions                                    | No instructions developer message is added                                                          |
| Input order      | Tools, instructions, and conversation are present     | Developer tool item, optional instruction message, then original transformed input                  |
| Images           | Nested `input_image` contains `detail`                | Only `detail` is removed                                                                            |
| Images           | Multiple nested images                                | `detail` is removed recursively from every image                                                    |
| Images           | A non-image object contains `detail`                  | Its `detail` property is preserved                                                                  |
| Tool controls    | Original values differ                                | Result forces `tool_choice: auto` and `parallel_tool_calls: false`                                  |
| Top-level fields | Tools and instructions existed                        | Both fields are absent after transformation                                                         |
| Reasoning        | Effort and summary exist                              | Both are preserved and `context: all_turns` is added                                                |
| Session          | Metadata contains `taskId`                            | Cache key and all session/affinity headers use that task ID                                         |
| Session fallback | Metadata omits `taskId`                               | Cache key and headers use the handler UUID                                                          |
| SDK transport    | Luna streaming succeeds through SDK                   | SDK receives the transformed body and Luna headers                                                  |
| SSE fallback     | SDK fails and fetch fallback runs                     | Fetch receives the same transformed body and Luna headers, with no duplicate transformation         |
| Completion       | Luna `completePrompt()`                               | Non-streaming body is transformed and handler UUID is used consistently                             |
| General headers  | Luna and non-Luna                                     | Existing originator, user agent, account ID, authorization behavior, and `session_id` are preserved |
| Regression       | Existing non-Luna native tool calls and stream events | Existing tests continue to pass                                                                     |

Where handler UUID assertions are needed, compare values captured from the body and headers rather than asserting a specific generated UUID. The important invariant is equality across all session-bearing fields.

## Risks and mitigations

### Over-broad model detection

**Risk:** Applying Responses Lite behavior to GPT-5.5, Sol, Terra, aliases, or unrelated providers could break otherwise valid requests.

**Mitigation:** Gate on exact equality with `gpt-5.6-luna` inside the Codex OAuth provider and add explicit negative tests.

### Divergent SDK and fallback requests

**Risk:** Transforming separately in each transport can produce different bodies or prepend the developer items twice during fallback.

**Mitigation:** Transform once before transport selection and pass the same result to both paths.

### Session mismatch

**Risk:** Using `taskId` for one header and the handler UUID for the cache key or affinity header can fragment server-side session/cache behavior.

**Mitigation:** Resolve one effective session ID and reuse it everywhere.

### Accidental data removal during recursive traversal

**Risk:** A broad sanitizer could remove `detail` from unrelated objects or strip other image properties.

**Mitigation:** Remove the property only when an object has `type: input_image`; retain all other keys and test nested non-image controls.

### Mutation of the normal request body

**Risk:** In-place mutation can make fallback behavior and test diagnostics difficult to reason about.

**Mitigation:** Use a non-mutating recursive transformation and construct a new Luna body.

### Reasoning-shape regression

**Risk:** Replacing the reasoning object could lose selected effort or automatic summary behavior.

**Mitigation:** Merge `context: all_turns` into the existing reasoning configuration and assert all three values.

### Header spelling

**Risk:** Treating `session-id` as a replacement for the existing `session_id` would alter general Codex behavior.

**Mitigation:** Preserve `session_id` and add the hyphenated Luna header alongside it.

### SDK type mismatch

**Risk:** Responses Lite fields or developer items may not be represented by the installed OpenAI SDK's public request types even though the HTTP endpoint accepts them.

**Mitigation:** Keep any narrow type escape localized to the adapter/dispatch boundary, validate the resulting shape in unit tests, and avoid weakening types across the provider.

## Non-goals

This port must not:

- add, remove, rename, or otherwise edit model catalog entries;
- change GPT-5.5 behavior;
- change `gpt-5.6-sol` or `gpt-5.6-terra` behavior;
- apply Luna behavior by alias, substring, model family, or capability;
- modify API-key-based OpenAI providers or any non-Codex provider;
- add a WebSocket transport or WebSocket metadata;
- alter response event parsing, token accounting, or native tool-call parsing unless a focused test proves a separate compatibility need;
- change OAuth token refresh behavior;
- create a changeset; or
- introduce unrelated refactoring.

## Verification commands

Run the focused provider tests from the [`src`](../src) workspace, which owns the relevant Vitest dependency:

```sh
cd src && npx vitest run api/providers/__tests__/openai-codex.spec.ts api/providers/__tests__/openai-codex-native-tool-calls.spec.ts
```

Then run type checking and linting from the same workspace:

```sh
cd src && pnpm check-types
cd src && pnpm lint
```

## Acceptance criteria

The port is complete when all of the following are true:

- only exact `gpt-5.6-luna` ChatGPT OAuth requests use Responses Lite;
- the Luna body follows the ordered transformation documented above;
- nested image `detail` fields alone are removed;
- reasoning effort and summary are preserved with all-turn context;
- one effective session ID is shared by the cache key and all relevant headers;
- SDK streaming, SSE fallback, and `completePrompt()` are covered;
- existing Codex headers and every non-Luna request remain unchanged;
- the focused tests, type check, and lint commands pass; and
- no WebSocket, catalog, alias, API-key-provider, changeset, or unrelated changes are included.
