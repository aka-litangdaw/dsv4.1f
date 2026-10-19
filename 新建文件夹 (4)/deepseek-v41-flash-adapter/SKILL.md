---
name: deepseek-v41-flash-adapter
description: Adapts evidence-grounded research, scientific-data, coding, and tool workflows to DeepSeek-V4.1-Flash-1017, exposed by the official DeepSeek API as deepseek-flash. Use when the selected model/provider is DeepSeek V4.1 Flash 1017 or when debugging its thinking, tool-call, JSON, vision, context-cache, or Responses API behavior.
---

# DeepSeek V4.1 Flash 1017 Adapter

## Scope and identity

This is a **runtime and workflow adapter**, not a weight-training recipe. It should be used together with `evidence-grounded-answer-loop` when the task needs evidence control.

The official DeepSeek API currently exposes the model with the request name `deepseek-flash`; the official model-version field identifies it as **DeepSeek-V4.1-Flash-1017**. Legacy names such as `deepseek-v4-flash` and `deepseek-v4-flash-vision-exp` are routed to the current Flash model according to the official documentation. If another provider exposes `deepseek-v4.1-flash-1017` directly, record the provider, exact model ID, response `model`, and `system_fingerprint` before assuming that its behavior and parameter support are identical to the official API.

## Verified runtime profile

Use these as API constraints, not as claims about every third-party deployment:

- Context length: **1M tokens**.
- Maximum output: **384K tokens**.
- Thinking and non-thinking modes are available; thinking is enabled by default in the official API and the default effort is `high`.
- Supported thinking efforts: `low`, `high`, and `max`.
- `deepseek-flash` supports JSON Output, function tool calls, Responses API, Anthropic-compatible API, and vision input.
- The official model accepts images in JPEG, PNG, GIF, and WebP formats. Image limits and token cost still apply.
- Official context caching is automatic; inspect usage fields rather than assuming a cache hit.

Do not copy these properties to a provider-specific endpoint without checking that provider’s model metadata and API documentation.

## Mode selection

Choose the mode from the task, not from the input length alone:

### Evidence analysis without tools

- Use thinking mode for complex literature synthesis, condition matching, causal-boundary review, and multi-step scientific reasoning.
- Prefer `reasoning_effort=high` for normal strict analysis; use `max` only when the acceptance rule is difficult enough to justify extra latency and output.
- Do not treat the returned `reasoning_content` as evidence. Cite the source, data, or test result instead.
- If the request has no `tools` parameter, previous reasoning content does not need to be replayed for the next turn according to the official thinking-mode guide.

### Tool-using agent work

- Thinking mode supports tool calls, but every subsequent request carrying `tools` must replay the complete prior assistant message, including `reasoning_content` and `tool_calls`; otherwise the API can return HTTP 400.
- Use `tool_choice=auto` in thinking mode unless the provider confirms support for a stronger choice. The official Chat Completions reference states that `required` and named tool choices are not supported in thinking mode.
- If a workflow must force a tool call, use non-thinking mode for that sub-step or use a supported strict tool schema; do not silently assume `required` works in thinking mode.
- Validate tool arguments in the host before execution. Do not let the model’s generated JSON directly authorize a destructive action.
- Preserve the tool result and the exact tool-call message in the evidence record.

### Structured extraction

For claim ledgers, metadata extraction, and machine-consumed reports:

- Prefer non-thinking mode plus JSON Output when a short, schema-bound response is more valuable than extended reasoning.
- Set `response_format` to `{"type":"json_object"}`.
- Explicitly tell the model to output JSON and include a small example schema; the official guide warns that omitting this instruction can produce whitespace or an apparently stuck request.
- Set a sufficient `max_tokens` so the JSON is not truncated. Validate the parsed object and check `finish_reason` before accepting it.
- Treat empty content or `finish_reason=length` as `inconclusive`, not as a successful extraction.

## Evidence-grounded workflow adaptations

### Stable prompt prefix and cache-aware turns

The API’s disk cache is prefix-based and best-effort. Keep the following prefix stable across a multi-turn task:

1. system/skill instructions;
2. evidence status definitions;
3. source and task metadata;
4. the unchanged evidence block.

Append the new question, tool result, or review request after that stable prefix. Do not rewrite the same long evidence block in a different order on every turn. When available, record `prompt_cache_hit_tokens` and `prompt_cache_miss_tokens` from usage; cache status is an optimization signal, not evidence of answer correctness.

### Long-context evidence handling

The 1M-token context does not justify passing an entire Zotero library, raw instrument dump, or repository into one request. Use staged retrieval:

```text
inventory → targeted extraction → claim/evidence cards → synthesis → final audit
```

Keep a compact evidence card stable between turns. For long files, pass the relevant range, page, table, or time window and retain the path and locator. Use the model’s large context for aligned evidence, not for unfiltered volume.

### Scientific data

For UV–Vis, electrochemical, or instrument time series, ask the model to separate:

- file/header facts;
- filename-only hints;
- derived statistics and formulas;
- protocol-confirmed conditions;
- missing metadata;
- conclusions that are only conditional.

Never let the model infer voltage, sample identity, units, or protocol from a filename alone. For plots and images, provide the image as a user input or tool output, not as a system/assistant image block.

### Code and repository work

- Keep the project root, exact command, exit status, changed files, and unrun checks in the task contract.
- Use function tools with narrow JSON Schemas. In strict tool mode, every object property must be required and objects must set `additionalProperties` to `false`; validate the schema before sending it.
- Separate `passed`, `failed`, `blocked`, `not-run`, and `inconclusive`. A malformed tool call, stale path, missing dependency, or unavailable credential is not automatically a code defect.
- For patching, prefer the host’s patch executor. If using the Responses API, the official compatibility guide supports the `apply_patch` custom tool but ignores other custom tool names; do not assume generic MCP, computer-use, code-interpreter, or file-search tools are native to this model API.
- After a tool call, re-read the affected state before claiming success. The model’s final prose is not an independent verification.

### Vision and files

- Use `detail=original` or `high` for small labels, plot legends, axes, microscopy details, and dense tables; use `low` only for coarse scene or layout checks.
- Inline or external images are limited by the documented request/image sizes. Use the Files API `file_id` when an image is reused or would exceed inline limits.
- The official Files API is image-oriented. Do not assume arbitrary PDFs, CSVs, or binary instrument files can be uploaded and parsed through it; extract or convert them locally first, then provide text, a table, or an image.
- Images are accepted only in user/developer-style input locations supported by the endpoint. Do not place images in system or assistant messages.

## Endpoint-specific constraints

### Chat Completions

- Official base URL: `https://api.deepseek.com`.
- Model parameter: `deepseek-flash` for the official API.
- With thinking plus tools, replay the full assistant message including `reasoning_content` on every tool sub-turn.
- With JSON Output, explicitly mention JSON in the prompt and check for truncation or empty content.
- Function tool arguments may be invalid or contain extra fields unless strict mode is used; validate regardless.

### Responses API

The official Responses API is stateless for this model:

- `previous_response_id` and `conversation` are not supported;
- `store`, `background`, `metadata`, and prompt-cache control parameters are not supported;
- automatic context caching is managed by the provider;
- function tools are supported;
- the `apply_patch` custom tool is supported for Codex compatibility;
- server-side `web_search` is supported by `deepseek-v4-pro`, not by `deepseek-flash`;
- file inputs are not general-purpose; images use `input_image`.

Therefore, explicitly resend the relevant conversation/evidence state when using Responses API. Do not assume that a response ID gives durable conversation memory.

## Model-specific smoke checks

Run these checks after changing provider, endpoint, model alias, OpenCode version, or tool adapter. Record the actual returned `model`, `system_fingerprint`, `finish_reason`, usage, and any error:

1. **Direct answer:** a short factual question with an explicit uncertainty instruction.
2. **Evidence comparison:** two values with one missing condition; verify that the model refuses an unqualified ranking.
3. **JSON extraction:** a small claim card using JSON Output; parse it and check schema and finish reason.
4. **Tool round trip:** one harmless function call; in thinking mode verify that the complete `reasoning_content` is replayed.
5. **Code/check status:** a harmless command that returns a known nonzero status; verify it is reported as a check result plus cause, not automatically as a code defect.
6. **Vision, when needed:** one chart or screenshot with axes/legend; verify that visual observations are separated from metadata not present in the image.

Do not call a provider adapter validated until the affected checks pass on the actual endpoint. Vendor benchmark claims are background evidence, not a substitute for this smoke suite.

## Model-specific evidence ledger

When recording a DeepSeek-specific behavior, use:

```text
provider | endpoint/API family | exact model ID | model returned by API |
system_fingerprint | date | mode/effort | tools/schema | task sample |
observed behavior | metric | limitation | next recheck trigger
```

Treat one observation as a pilot finding. Promote it to a reusable adapter rule only after repeated observations on the same model/provider/version and at least one task outside the original example.

## Open-weight boundary

This adapter does not require model weights and does not imply that DeepSeek-V4.1-Flash-1017 weights are available. The official sources consulted here document hosted API aliases and capabilities; they do not establish an official open-weight checkpoint for this exact dated version. Use skill/prompt/tool/RAG adaptation for the hosted API. Weight-level LoRA, SFT, or PPO work requires a separately verified checkpoint, tokenizer, license, training stack, and compute budget.

## Official sources consulted

- API quick start: https://api-docs.deepseek.com/
- Models and pricing: https://api-docs.deepseek.com/quick_start/pricing
- Chat Completions reference: https://api-docs.deepseek.com/api/create-chat-completion
- Thinking mode: https://api-docs.deepseek.com/guides/thinking_mode
- Tool Calls: https://api-docs.deepseek.com/guides/tool_calls
- JSON Output: https://api-docs.deepseek.com/guides/json_mode
- Vision: https://api-docs.deepseek.com/guides/vision
- Files API: https://api-docs.deepseek.com/guides/files_api
- Context caching: https://api-docs.deepseek.com/guides/kv_cache
- Responses API: https://api-docs.deepseek.com/guides/responses_api
- V4.1-Flash-1017 changelog: https://api-docs.deepseek.com/updates

These sources were checked on 2026-10-19. Prices, aliases, limits, and provider behavior can change; recheck the official documentation before changing runtime configuration.
