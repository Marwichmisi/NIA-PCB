# Meta Model API — capability and integration notes

**Context:** Meta Global AI Developer Hackathon, Oct 26 – Nov 4, 2026. Target project: a web agent that generates PCB cards.
**Research date:** 2026-10-27.
**Source policy:** Meta-owned docs and pages first (`dev.meta.ai`, `ai.meta.com`), plus official inference-partner model pages (Baseten, Fireworks, Modal) and Meta's Hugging Face model repos. Third-party pages are labelled as such. No model ID, endpoint, price or capability below is inferred.

> **Naming note.** The hackathon guide calls all of these "models", but the docs are stricter: **Muse Code is not a model**, it is Meta's terminal/CI coding agent, built for Muse Spark and powered by Model API. Its default model is `muse-spark-1.2`. Treat "Muse Code" as a tool, not as an ID you can send to `api.meta.ai`.
> Source: <https://dev.meta.ai/docs/muse-code>, <https://dev.meta.ai/docs/overview>

---

## 1. Meta Model API — the integration surface

### 1.1 Base URL and auth

| Item | Value | Source |
|---|---|---|
| Base URL | `https://api.meta.ai/v1` | <https://dev.meta.ai/docs/api-reference> |
| Auth header | `Authorization: Bearer $MODEL_API_KEY` | <https://dev.meta.ai/docs/api-reference> |
| Env var | `MODEL_API_KEY` | <https://dev.meta.ai/docs/api-reference> |
| Key management | Model API dashboard at `https://dev.meta.ai/` | <https://dev.meta.ai/docs/authentication> |
| Error shape | OpenAI-style `{ "error": { "message", "type", "param", "code" } }` | <https://dev.meta.ai/docs/api-reference> |
| Health check | `GET /v1/status` — unauthenticated | <https://dev.meta.ai/docs/api-reference> |

There is **no Meta-proprietary SDK**. The documented clients are the standard OpenAI SDK, the Anthropic SDK, and any OpenAI-compatible client (LangChain, LlamaIndex, Vercel AI SDK, LiteLLM). One caveat the docs call out explicitly: neither SDK reads `MODEL_API_KEY` automatically — the OpenAI SDK looks for `OPENAI_API_KEY`, the Anthropic SDK for `ANTHROPIC_API_KEY`. Pass the key explicitly.
Source: <https://dev.meta.ai/docs/sdks>

### 1.2 Three wire formats, same models, same price

| Format | Endpoint | Best for | Reasoning carried across turns |
|---|---|---|---|
| **Responses** (recommended) | `POST /v1/responses` | Agents, multi-step tool loops, full feature set | Yes — encrypted replay or `previous_response_id` |
| **Chat Completions** | `POST /v1/chat/completions` | Dropping into existing OpenAI code | No |
| **Messages** | `POST /v1/messages` | Anthropic-format tools (Claude Code, Claude Agent SDK) | Yes — Anthropic `thinking` replay |

Source: <https://dev.meta.ai/docs/protocols>

For the Anthropic SDK the client base URL is the bare host `https://api.meta.ai` (the SDK appends `/v1/messages`) and the key goes in `auth_token`, not `api_key`. A response's `content` starts with a `redacted_thinking` block carrying encrypted reasoning, so read the `text` blocks rather than indexing `content[0]`.
Source: <https://dev.meta.ai/docs/sdks>

### 1.3 OpenAI compatibility, precisely

- `client.responses.create()` and `client.chat.completions.create()` both work through the OpenAI SDK.
- `client.models.list()` works (`GET /v1/models`), returning API-hosted models only — self-hosted Muse Glimmer is not listed.
- `client.files.create()` works (Files API).
- LiteLLM reaches it through the OpenAI provider with the `openai/` prefix: `model="openai/muse-spark-1.3"`.

Sources: <https://dev.meta.ai/docs/sdks>, <https://dev.meta.ai/docs/models>

Not everything is OpenAI-identical. Documented divergences:

- `tool_choice` accepts **only** `"auto"`; `"none"`, `"required"` and named-function choices return HTTP 400.
- `logprobs: true` returns HTTP 400 (not supported).
- `truncation: "auto"` is rejected; only the default `"disabled"` is accepted. You must compact history yourself.
- `functions` / `function_call` are deprecated; use `tools` / `tool_choice`.

Sources: <https://dev.meta.ai/docs/tool-calling>, <https://dev.meta.ai/docs/reasoning>, <https://dev.meta.ai/docs/protocols/responses>

### 1.4 Streaming

- Set `stream: true` and consume server-sent events. Typed events include `response.created`, `response.in_progress`, `response.output_item.added/done`, `response.content_part.added/done`, `response.output_text.delta`, `response.completed`. Tool arguments stream as `response.function_call_arguments.delta/done`.
- Streaming is not optional for large work: **non-streaming requests are subject to a server-side time limit and return HTTP 504 (`gateway_timeout`) if they run too long**; streaming is exempt because it returns output incrementally.
- `background: true` runs the response asynchronously; then poll, stream-retrieve with `GET /v1/responses/{id}?stream=true`, or cancel. You cannot combine `background: true` with `stream: true` on creation (HTTP 400), and a background creation body is capped at 1 MiB.
- Muse Image has its own SSE shape (`image_generation.completed`, `image_edit.completed`) when `stream: true`.

Sources: <https://dev.meta.ai/docs/protocols/responses>, <https://dev.meta.ai/docs/image-generation>

### 1.5 Tool / function calling

- Tools live in `tools`, type `function` (JSON Schema parameters) or `custom` (freeform text input). Custom tools are Responses-only; sending one to Chat Completions returns 400.
- The model never executes developer tools; you do, and you return `function_call_output` keyed by `call_id`.
- `parallel_tool_calls` defaults to **`true`** on both endpoints.
- `strict` defaults to **`false`**. When `false`, the schema is accepted as-is; when `true`, the server validates it against the strict subset (`additionalProperties: false`, every property required) and returns 400 on violation.
- **Regardless of `strict`, generated arguments are not guaranteed to validate.** The docs say to parse and validate arguments in your own code before use.
- Function names must match `^[a-zA-Z0-9_.-]+$` and contain **at most one dot**. Multi-segment MCP-style names need a namespace tool.
- `max_tool_calls` caps **built-in** tool calls only (e.g. `web_search`), not your function tools.
- `tool_search` + `defer_loading: true` lets the model load tool schemas on demand — relevant if the PCB agent exposes many tools (ERC, DRC, BOM, footprint lookup, etc.). Responses and Messages only, not Chat Completions.

Source: <https://dev.meta.ai/docs/tool-calling>, <https://dev.meta.ai/docs/tool-search>

### 1.6 Structured outputs

- Constrained **decoding**, not post-processing: output is guaranteed to conform to the schema.
- Chat Completions uses `response_format: {type: "json_schema", json_schema: {name, schema}}`; the Responses API uses `text.format`. The two are not interchangeable — a parameter from the other endpoint does not configure structured output, even when accepted for compatibility.
- Recursive schemas are rejected (HTTP 400).
- Schema limits: nesting depth 10, 5,000 total properties, 120,000 total string characters, 1,000 enum values by default (a single string enum > 250 values is also capped at 15,000 chars), 200,000 expanded nodes after `$ref` inlining.

Source: <https://dev.meta.ai/docs/structured-output>

### 1.7 Reasoning control

`reasoning_effort` (top-level on Chat Completions, `reasoning.effort` on Responses) accepts `none | minimal | low | medium | high | xhigh | max`.

- `"none"` is **not supported** by Muse Wonder or Muse Spark — HTTP 400.
- `"max"` is supported by `muse-wonder-1.0` and `muse-spark-1.3`, but **not** by `muse-spark-1.3-contributor`.
- Reasoning tokens are billed as output tokens and count against `max_tokens` / `max_output_tokens`. Budget accordingly.
- Raw chain of thought is never returned as text. On Responses it can be carried across turns as encrypted items (`include: ["reasoning.encrypted_content"]`, `store: false`), or via `previous_response_id`. Those two cannot be combined in one request.

Source: <https://dev.meta.ai/docs/reasoning>

### 1.8 Files, PDFs and images

| Path | Limit |
|---|---|
| Inline (`file_data` / `file_url` in body) | 50 MB |
| Files API upload (`POST /v1/files`) | 1 GiB |
| Team storage across all keys | 100 GiB |
| Images per request | 50 (enforced, 400 above) |
| PDF text | first 100 pages |
| PDF page-images | first 50 pages |

PDFs are converted **in parallel to text and per-page raster images**. A PDF ≤ 50 pages is fully covered; longer ones lose text past page 100 and visuals past page 50. That matters for 80-page datasheets.
Source: <https://dev.meta.ai/docs/file-handling>, <https://dev.meta.ai/docs/image-understanding>

Vision cost scales with resolution, roughly 1,300–1,500 tokens for a ~1280 px image; a 1×1 image is ~3 tokens. Use `POST /v1/responses/input_tokens` for the authoritative count — it renders the request and runs the model's tokenizer, including media and tool definitions.
Sources: <https://dev.meta.ai/docs/image-understanding>, <https://dev.meta.ai/docs/token-counting>

### 1.9 Search grounding and tool search

- `web_search` is a built-in, server-side tool on text models: `tools=[{"type": "web_search"}]`. It returns `web_search_call` items and `url_citation` annotations on `output_text`.
- Pricing: **$2.50 per 1,000 search queries**, on top of token cost.
- Muse Image does **not** use this tool — its web/image search is built in, always on, and included in the per-image price. Passing `web_search` to Muse Image returns an error.

Sources: <https://dev.meta.ai/docs/search-grounding>, <https://dev.meta.ai/docs/pricing-rate-limits>, <https://dev.meta.ai/docs/image-generation>

### 1.10 Minimal calls

Every snippet below is the documented shape, condensed.

**Text model with a tool** (Responses API; flat tool schema):

```python
import json
import os
from openai import OpenAI

client = OpenAI(base_url="https://api.meta.ai/v1", api_key=os.environ["MODEL_API_KEY"])


def lookup_footprint(mpn: str) -> dict:
    return {"mpn": mpn, "package": "<from your library>"}  # replace with your lookup

tools = [{
    "type": "function",
    "name": "lookup_footprint",
    "description": "Return the footprint and land pattern for a part number.",
    "parameters": {
        "type": "object",
        "properties": {"mpn": {"type": "string"}},
        "required": ["mpn"],
        "additionalProperties": False,
    },
}]

r = client.responses.create(
    model="muse-spark-1.3",
    reasoning={"effort": "high"},
    input="Find the footprint for STM32G431CBT6.",
    tools=tools,
    parallel_tool_calls=True,
)

for item in r.output:
    if item.type == "function_call":
        args = json.loads(item.arguments)          # arguments is a JSON string
        result = lookup_footprint(**args)          # your tool; validate args first
        # follow up with previous_response_id=r.id and a function_call_output item
```

**Image generation** (Responses API; reuses the client above; image comes back base64 on `image_generation_call`):

```python
r = client.responses.create(
    model="muse-image-1.0",
    input="a PCB business card, front side, dark green solder mask, gold ENIG pads",
)
b64 = next(i.result for i in r.output if i.type == "image_generation_call")
```

**Segmentation** (streaming is recommended for video):

```python
with client.responses.stream(
    model="sam-3.1",
    input=[{"type": "message", "role": "user", "content": [
        {"type": "input_text", "text": "connector"},
        {"type": "input_image", "image_url": "https://example.com/board.jpg"},
    ]}],
) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="")
```

**Transcription** (documented curl form; `request` is a JSON part):

```bash
curl --fail-with-body -N \
  'https://api.meta.ai/v1/asr/transcribe?sessionId=demo' \
  -H "Authorization: Bearer $MODEL_API_KEY" \
  -H 'Accept: text/event-stream' \
  -F 'request={"mode":"ENDPOINTING","model":"muse-voice-transcribe-1.0","audioEncoding":"WAV"};type=application/json' \
  -F 'audio=@meeting_audio.wav'
```

Sources: <https://dev.meta.ai/docs/sdks>, <https://dev.meta.ai/docs/tool-calling>, <https://dev.meta.ai/docs/image-generation>, <https://dev.meta.ai/docs/sam/overview>, <https://dev.meta.ai/docs/speech-to-text>


---

## 2. Model-by-model

### 2.1 Summary table

| Model ID | Input | Output | Context | Tool use | Structured output | Price | Status |
|---|---|---|---|---|---|---|---|
| `muse-wonder-1.0` | text, image, video, audio, PDF | text | 1,048,576 | Yes | Yes | $4.00 in / $0.20 cached / $20.00 out per 1M | Preview; not in South Korea |
| `muse-spark-1.3` | text, image, video, audio*, PDF | text | 1,048,576 | Yes | Yes | $1.25 / $0.15 / $4.25 per 1M | Available |
| `muse-spark-1.2` | text, image, video, audio, PDF | text | 1,048,576 | Yes | Yes | same as 1.3 | Available |
| `muse-spark-1.1` | text, image, video, audio, PDF | text | 1,048,576 | Yes | Yes | same as 1.3 | Standard only, not in Muse Code |
| `muse-spark-1.3-contributor` | same | text | 1,048,576 | Yes | Yes | $0.10 / $0.002 / $0.20 per 1M | Contributor mode; no `max` effort |
| `muse-image-1.0` | text, image | image | n/a | No function tools; internal search/shell | n/a | $0.01 per generated image | Available |
| `muse-voice-transcribe-1.0` | audio | text | n/a | n/a | n/a | $0.18 per audio hour | Available |
| `sam-3.1` | text prompt + image/video | boxes + masks (text grammar) | n/a | n/a | n/a | $2.50/1k images; $0.20/1k frames (min 50) | Available |
| `muse-spark-open` (self/providers) | text, image, video, audio** | text | 1,048,576 | Yes | Not documented for open weights | Provider-dependent | Open weights |
| `muse-glimmer` (self/providers) | text, image | text | 131,072 (128K) | Yes | Not documented | Provider-dependent | Open weights, Apache 2.0 |

\* Audio understanding on Muse Spark 1.3 is explicitly "not fully supported" and quality may degrade; the docs say use 1.2 for audio.
\*\* Modality support depends on the provider — Fireworks documents text + images only; Baseten and Modal document all four.
Sources: <https://dev.meta.ai/docs/models>, <https://dev.meta.ai/docs/pricing-rate-limits>

Benchmark tables (SWE-Bench, Terminal-Bench, MCP Atlas, OSWorld, etc.) are published on the model pages at <https://dev.meta.ai/models/muse-wonder>, <https://dev.meta.ai/models/muse-spark>, and <https://dev.meta.ai/models/muse-glimmer>. They are not reproduced here because the HTML tables are not reliably attributable cell-by-cell; use them only for relative sizing, not as procurement evidence.

### 2.2 Muse Wonder — `muse-wonder-1.0`

Meta's most capable model, positioned for complex software engineering and long-horizon agentic work: multi-file changes, sub-agent delegation, parallel tool calls, error recovery, computer use. Text/image/video/audio/PDF in, text out, 1M context, reasoning up to `max`.

Important availability caveat: **Muse Wonder is in preview**, and the higher-capability variant **Muse Wonder Full is not available on Model API at all** — it is Muse Code subscription-only (Power Usage and Maximum Usage plans). Setting `reasoning_effort: "max"` on Model API does not select Muse Wonder Full. Muse Wonder is also not available in South Korea, on either surface.

Sources: <https://dev.meta.ai/docs/models>, <https://dev.meta.ai/models/muse-wonder>

### 2.3 Muse Spark series — `muse-spark-1.3` / `1.2` / `1.1`

Meta's agentic/coding model line, the one the docs default to. Same modalities and 1M context as Wonder, cheaper per token, tuned for high-volume, latency-sensitive sub-agent work and tight tool loops.

Three versions differ by capability, not by interface: 1.3 is the latest, tuned for agentic workflows (multi-step tool, browser, long-horizon) with improved coding over 1.2; 1.2 is previous; 1.1 is original. All support tool calling, structured output, search grounding, image and video/audio understanding.

Contributor mode is a real architectural choice: `muse-spark-1.3-contributor` runs the same model at **$0.10 input / $0.002 cached / $0.20 output per 1M**, in exchange for Meta being allowed to train on your prompts, files and outputs. For a hackathon with original PCB designs, that is a data-governance decision, not just a discount.

Sources: <https://dev.meta.ai/docs/models>, <https://dev.meta.ai/docs/pricing-rate-limits>

### 2.4 Muse Image — `muse-image-1.0`

One model for generation and editing. Text + optional reference images in, image out. It is explicitly agentic: before rendering it can pull visual references from the web, look up current facts, and run code to build layouts/charts. That grounding is built in and included in the price.

Two surfaces:

1. **Responses API** (`POST /v1/responses`, `model: "muse-image-1.0"`) for conversational, multi-turn editing with `previous_response_id` or replayed `image_generation_call` items.
2. **OpenAI-compatible Images API**: `POST /v1/images/generations` and `POST /v1/images/edits`.

Controls (Responses): `enable_image_search`, `enable_web_search`, `enable_shell` (all default `true`); `reasoning_strength: "high" | "low"`; `size` as a `"WxH"` string that sets **aspect ratio, not exact pixel dimensions** — the generator's own resolution is used; `output_format: "webp" | "png" | "jpeg"` (webp default).

Price is flat **$0.01 per generated image**, including `n > 1` (billed per returned image), and failed/safety-filtered images are not billed. Aspect ratios beyond 16:1 return 400.

Also available outside Meta: **fal, Runway, OpenRouter** (per the model page).
Sources: <https://dev.meta.ai/docs/image-generation>, <https://dev.meta.ai/models/muse-image>

### 2.5 Muse Voice Transcribe — `muse-voice-transcribe-1.0`

Speech-to-text only. Audio in, transcript out. Two endpoints, same model:

| Endpoint | Use |
|---|---|
| `wss://api.meta.ai/v1/asr/realtime` | Live audio (voice agents, dictation, captions). Auth in the handshake frame; the `Authorization` header is ignored. |
| `POST https://api.meta.ai/v1/asr/transcribe` | An existing recording, multipart upload, Bearer auth. |

Capabilities: 25 languages with code-switching and `languageBias`; keyword/vocabulary biasing; endpointing (`PUSH_TO_TALK`, `ENDPOINTING`, `DIARIZATION` modes); speaker labels for 20+ speakers; partial transcripts; turn-level timestamps.

Documented gaps: **no word-level timestamps**, no confidence scores, no sound-event detection, no emotion detection, no transcript reformatting. It does not synthesize speech and there is no speech-to-speech API.

Pricing: **$0.18 per hour of audio**, metered in whole seconds, streaming and file priced the same.
Sources: <https://dev.meta.ai/docs/speech-to-text>, <https://dev.meta.ai/docs/models>, <https://dev.meta.ai/docs/pricing-rate-limits>

### 2.6 SAM 3.1 — `sam-3.1`

See §5 for the deep dive. In one line: promptable concept segmentation over images and video. Short noun phrase (or box/point prompts) in, boxes + pixel masks out, streamed as a text grammar on the Responses API.
Sources: <https://dev.meta.ai/docs/sam/overview>, <https://dev.meta.ai/docs/models>

### 2.7 Muse Spark Open — `muse-spark-open` (not an API model)

816B total / 42B active sparse MoE, multimodal (text, image, audio, video in; text out), 1,048,576-token context. It is **not served on `api.meta.ai`** — you self-host or call a provider.
Sources: <https://dev.meta.ai/docs/muse-spark-open>, <https://dev.meta.ai/docs/models>

### 2.8 Muse Glimmer — `muse-glimmer` (not an API model)

30B (29.6B) dense multimodal model distilled from Muse Spark, text + image in, text out, 131,072-token context, Apache 2.0. Built for local/edge agents.
Sources: <https://dev.meta.ai/docs/muse-glimmer>, <https://huggingface.co/meta-models/Muse-Glimmer-30B>

### 2.9 Muse Code — a coding agent, not a model

Terminal + CI coding agent: plans, edits, runs commands, with approvals and an OS sandbox from the first run. Install via `curl -fsSL https://dev.meta.ai/install.sh | sh`; default model `muse-spark-1.2`; `muse --model muse-spark-1.3` overrides. It is "the same models, same auth" as Model API, but it is a product, not an endpoint.
Source: <https://dev.meta.ai/docs/muse-code>

---

## 3. Which model for which job

Recommendations below are engineering judgements grounded in the documented capabilities above, not Meta claims.

| Job | Primary | Why | Fallback / notes |
|---|---|---|---|
| **(a) Agent planning + tool use** | `muse-wonder-1.0`, `reasoning_effort: "max"` | Docs position Wonder for long-horizon, sub-agent orchestration, parallel tools, error recovery | `muse-spark-1.3` at `high`/`xhigh` for sub-agents and tight loops. Cost ratio is ~3.2× input and ~4.7× output, so route by difficulty. |
| **(b) Code generation (ato / Python / TS)** | `muse-spark-1.3` | 1M context, agentic/coding tuning, structured output for tool args, best cost/latency | `muse-wonder-1.0` when a change spans many files and must be verified. There is **no documentation for "ato"** — the model emits text, so any language is possible in principle, but Meta documents no ato support. |
| **(c) Image generation** | `muse-image-1.0` | $0.01/image, reference-image conditioning, multi-turn refinement, built-in search | fal / Runway / OpenRouter for the same model outside Meta. Use Responses for iteration, `/v1/images/generations` for one-shots. |
| **(d) Image / datasheet / schematic understanding** | `muse-spark-1.3` | Native image + PDF input, 50 images/request, 1M context, perception grounding with normalized 0–1000 coordinates | `muse-wonder-1.0` for harder visual reasoning. Use SAM 3.1 only to get masks/boxes, not to interpret. |
| **(e) Audio transcription** | `muse-voice-transcribe-1.0` | $0.18/h, streaming + diarization + 25 languages | If you need audio *reasoning* (not just text), use `muse-spark-1.2` — audio on 1.3 is degraded per docs. |

**Cost sanity check for a 10-day build.** At $150 of credits: ~15,000 Muse Image generations, or ~833 hours of transcription, or ~120M Muse Spark input tokens at list price. Muse Wonder at $20/1M output is the budget risk — reserve it for the planner, not the inner loop. (Arithmetic derived from the published prices at <https://dev.meta.ai/docs/pricing-rate-limits>.)

### 3.1 Suggested agent topology for the PCB card generator

This is a design proposal; every capability it relies on is documented above.

1. **Planner** — `muse-wonder-1.0`, `reasoning_effort: "high"` or `"max"`, Responses API with `previous_response_id`. Produces a structured plan (JSON schema) of card sections, required inputs, and tool calls.
2. **Tool layer** — your own functions (component lookup, footprint/land-pattern check, BOM roll-up, image render, ERC/DRC wrapper). Keep names to one dot max, validate arguments yourself even with `strict: true`, and let `parallel_tool_calls` stay on.
3. **Datasheet reader** — `muse-spark-1.3` with `input_file` PDF, chunked to respect the 100-page text / 50-page image ceiling; ask for structured output (pin table, absolute max ratings, package) rather than prose.
4. **Schematic locator** — `muse-spark-1.3` perception grounding on the rendered schematic image, requesting boxes in the 0–1000 grid; convert to pixels, crop, then optionally feed the crop to `sam-3.1` for a mask if you need the exact region.
5. **Visual generation** — `muse-image-1.0` with reference images (previous cards, house style) to keep the card series visually consistent; `reasoning_strength: "high"` only when typography/layout accuracy matters.
6. **Long jobs** — `background: true` + polling, because non-streaming requests hit a server-side timeout (504) and background bodies are capped at 1 MiB, so pass files by `file_id`.

---

## 4. Open weights: Muse Spark Open and Muse Glimmer

### 4.1 Muse Spark Open

| Property | Value | Source |
|---|---|---|
| Architecture | Sparse MoE transformer with perception encoders, 816B total / 42B active | <https://dev.meta.ai/docs/muse-spark-open> |
| Modalities | text, image, audio, video → text | same |
| Context | 1,048,576 tokens | same |
| Languages | 100+ | same |
| BF16 checkpoint | ~1.65 TB weights; 8 × B300 (TP8/EP8); budget ~2 TB VRAM | same, plus <https://dev.meta.ai/docs/muse-spark-open/get-the-model> |
| NVFP4 checkpoint | ~537 GB weights; 4 × B300 (TP4/EP4) | same |
| HF repos | `meta-models/Muse-Spark-Open-816B-A42B`, `meta-models/Muse-Spark-Open-816B-A42B-NVFP4` | <https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B> |
| License | **Muse Model License** + Acceptable Use Policy | <https://dev.meta.ai/docs/muse-spark-open> |
| Access | Gated: you must sign in and accept the licence before download | <https://dev.meta.ai/docs/muse-spark-open/get-the-model> |

**Is it truly open?** It is open-weight, not open-source: weights are published, but under the Muse Model License rather than an OSI licence. I could not read the licence text anonymously — the HF repo returns HTTP 401 without authentication, so the specific grant/restriction clauses are **not verified** here. Do not assume Apache-2.0-style rights.

**Serving.** Self-host runtimes documented: SGLang, vLLM, TensorRT-LLM, NVIDIA Dynamo, llama.cpp, MLX, plus Hugging Face Transformers (`MuseSparkForConditionalGeneration`). Hardware guidance covers NVIDIA Blackwell (B300/GB300), AMD MI355X/MI350X (8 GPUs), Intel Xeon 6 (INT4, SGLang), and Apple Mac Studio M5 Ultra (NVFP4, MLX).
Sources: <https://dev.meta.ai/docs/muse-spark-open/deploy>, <https://dev.meta.ai/docs/muse-spark-open/hardware-platforms>

**Hosted providers documented by Meta:** Together AI (`meta-models/muse-spark-open`), Fireworks (`accounts/fireworks/models/muse-spark-open`), Baseten (`meta-models/muse-spark-open`), Groq (approval required), Nebius Token Factory, plus Modal and OpenRouter.
Source: <https://dev.meta.ai/docs/muse-spark-open/deploy#hosted-providers>

| Provider | Serverless? | Model string | Price (per 1M: in / cached / out) | Notes |
|---|---|---|---|---|
| Baseten | Yes (Model APIs) + dedicated | `meta-models/muse-spark-open` | $1.25 / $0.15 / $4.25 | Full 1,048,576 context; text, image, video, audio; OpenAI-compatible at `https://inference.baseten.co/v1`; video ≤ 200 MB by URL, ~96 frames sampled |
| Fireworks | Yes, US-only capacity | `accounts/fireworks/models/muse-spark-open` | $1.25 / $0.15 / $4.25 | **Text + images only**; 1,040k context; tool calling; `reasoning_effort`; also an Anthropic-compatible Messages API |
| Modal | Yes (Shared Endpoint) | see Modal library | $1.25 / $0.15 / $4.25 | NVFP4; 1M context; text/image/video/audio |
| Groq | Yes, approval required | `meta-models/muse-spark-open` | not published on Meta docs | **131,072-token context** on Groq, not 1M |

Sources: <https://www.baseten.co/library/muse-spark-open>, <https://fireworks.ai/models/fireworks/muse-spark-open>, <https://modal.com/library/meta/muse-spark-open>, <https://dev.meta.ai/docs/muse-spark-open/deploy>

Note the provider divergence: Baseten and Modal advertise all four input modalities; Fireworks advertises text + images. Meta's own docs describe the checkpoint as text/image/audio/video. Assume multimodal parity only where the provider says so.

### 4.2 Muse Glimmer

| Property | Value | Source |
|---|---|---|
| Size | 29.6B total (30B class), dense causal transformer + ~1.8B ViT-G/14 perception encoder | model card, <https://huggingface.co/meta-models/Muse-Glimmer-30B> |
| Modalities | text + image → text | same |
| Context | 131,072 (128K) default; longer contexts supported | <https://dev.meta.ai/docs/muse-glimmer> |
| License | **Apache 2.0** | <https://dev.meta.ai/docs/muse-glimmer> |
| BF16 weights | ~60 GB; ~72 GB VRAM to serve | <https://dev.meta.ai/docs/muse-glimmer/deploy> |
| GGUF Q4_K_M | ~17 GB, fits under 24 GB VRAM (text); +1.4 GB vision projector | <https://dev.meta.ai/docs/muse-glimmer/get-the-model> |
| Reasoning | Selectable strength, `low` → `xhigh` | Fireworks model page |
| Tool calling | Yes (vLLM recipe uses `--tool-call-parser muse_glimmer`) | <https://dev.meta.ai/docs/muse-glimmer/deploy> |
| Access | Ungated on Hugging Face (no auth needed) | observed + docs |

**Serving.** vLLM (`vllm/vllm-openai:muse-glimmer` image), SGLang, llama.cpp, Ollama, LM Studio, ExecuTorch, Hugging Face Transformers, Unsloth. Validated hardware in the docs: B300, B200, H200, RTX PRO 6000 96 GB, DGX Spark, RTX 5090 32 GB (Q4_K_M/NVFP4), Apple silicon M5 Pro 48 GB+.
Source: <https://dev.meta.ai/docs/muse-glimmer/deploy>

**Hosted:**

| Provider | Status | Model string | Notes |
|---|---|---|---|
| Baseten | Dedicated deployment, not serverless Model APIs | `meta-models/Muse-Glimmer-30B` | BF16 on a single H100 via OpenAI-compatible vLLM; paired with the 2.56B DFlash assistant; no per-token price published on the page |
| Fireworks | **On-demand only** — serverless deprecated 2026-09-25 | `accounts/fireworks/models/muse-glimmer-30b` | 131K context, function calling, LoRA fine-tuning supported; no price shown on model page |
| Modal | Dedicated endpoints | `muse-glimmer-30b-nvfp4` / `-bf16` | NVFP4 build is RadixArk's mixed NVFP4/MXFP8 quantisation, **text-only**, 128K, tool calling |

Sources: <https://www.baseten.co/library/muse-glimmer-30b>, <https://fireworks.ai/models/fireworks/muse-glimmer-30b>, <https://modal.com/library/meta/muse-glimmer-30b-nvfp4>

**Hackathon-relevant caveat:** Meta's Glimmer overview lists Fireworks as a launch partner, but Fireworks' own page says serverless was retired on 25 September 2026 and only on-demand remains. For a 10-day build, Baseten's single-H100 dedicated deployment or a local GGUF on a 24–32 GB GPU is the lower-friction path.

---

## 5. SAM 3.1 — what it actually is

### 5.1 Definition

SAM 3.1 is the current version of Meta's Segment Anything Model family, available both as open weights and on Model API as `sam-3.1`. It performs **promptable concept segmentation**: given a short noun phrase ("yellow school bus") plus an image or video, it finds **every** instance of that concept and returns a box and a pixel-accurate mask for each. It also accepts visual prompts — boxes and points — to segment specific objects, and tracks them through video.

It is a *segmentation* model, not a vision-language model. It returns geometry, not descriptions.
Sources: <https://dev.meta.ai/docs/sam/overview>, <https://ai.meta.com/blog/segment-anything-model-3/>

### 5.2 Inputs and outputs

| | Detail |
|---|---|
| Input | One user message: `input_text` short noun phrase, plus one `input_image` or `input_video`. Box/point prompts go in the same `input_text` part, written in the token grammar (never mixed with a noun phrase). |
| Image formats | Any format accepted by image understanding (jpeg/png/gif/webp/x-icon) |
| Video formats | MP4 or MOV, H.264 / H.265 / MPEG-4. **VP9/WebM rejected** |
| Output | `output_text` as a special-token grammar: `<0f>` frame marker, object ordinal, then `<\|box;x1=…;y1=…;x2=…;y2=…;w=…;h=…\|>`, then `<\|mask;x=…;y=…;data=…\|>` |
| Coordinates | Source pixels, with the media's displayed `w`/`h` (EXIF/rotation applied) |
| Mask encodings | `lossless` (default, `~` marker) or `one_bit` (`!` marker); box/point prompts always return `one_bit` |
| Confidence | Opt-in via `metadata.include_confidence: "true"` — a `c` field, 0–1, per box and mask per frame |
| Streaming | Recommended for video; non-streaming returns the same grammar in one response |
| Client libraries | `@meta-sam/parser` (TS), `meta_sam_parser` (Python), plus optional graphics/video/react packages |

Sources: <https://dev.meta.ai/docs/sam/overview>, <https://dev.meta.ai/docs/sam/segmenting>, <https://dev.meta.ai/docs/sam/reading-segmentation>, <https://dev.meta.ai/docs/sam/client-libraries>

### 5.3 Limits

- Up to **15,000 frames** per request (~10 minutes).
- Video tracks up to **16 objects** per frame; box/point prompts name up to **16 objects**.
- One concept per request. To segment several object types, send several requests.
- Zero matches is valid — the stream completes with empty `output_text`.
- Single image inference ~30 ms on an H200 with 100+ objects (SAM 3 figure); SAM 3.1 reaches 32 fps video tracking on a single H100.

Sources: <https://dev.meta.ai/docs/sam/segmenting>, <https://ai.meta.com/blog/segment-anything-model-3/>

### 5.4 Can it read schematics or datasheets?

**Partly, and not the way you want.** Honest reading of the primary sources:

- It segments **concrete visual concepts named by a short noun phrase**. It does not read text, interpret pin numbers, or answer questions. The docs explicitly redirect interpretation to Muse Spark: "For a caption, an answer about a scene, or coordinates as text, use a Muse Spark model… The two pair well: let Muse Spark interpret a scene and decide what to segment, then hand the concept to SAM 3.1 for the masks." (<https://dev.meta.ai/docs/media-segmentation>)
- The SAM 3 blog states the model **struggles to generalise to fine-grained out-of-domain concepts in a zero-shot manner**, especially niche technical/medical vocabulary, and **does not support longer, complex phrases** such as "the second-to-last book from the right on the top shelf". (<https://ai.meta.com/blog/segment-anything-model-3/>)

Practical consequences for PCB work:

| Task | Works? | Better approach |
|---|---|---|
| Crop a connector, IC package, or logo region from a board photo | Yes, with a plain noun phrase | `sam-3.1` + `one_bit` masks, then crop |
| Count repeated features in a photo (connectors, test points) | Plausible, if visually consistent | Prompt "connector", inspect `c` confidence, verify |
| Locate "pin 1 of U3" on a schematic | No — that is a referring expression, not a concept | `muse-spark-1.3` perception grounding (0–1000 boxes) + crop |
| Read a pin table from a PDF datasheet | No — SAM has no OCR/QA | `muse-spark-1.3` with PDF `input_file` + structured output |
| Trace nets across a schematic | No | Code/geometry layer, not SAM |
| Segment a symbol in a rasterised schematic for a card thumbnail | Yes | Box prompt on the symbol's coordinates, or noun phrase + crop |

A sound pipeline is: Muse Spark reads and localises → you crop in code → SAM 3.1 masks only when you need a clean alpha cut-out. Do not ask SAM to name what it segmented.

---

## 6. Rate limits, quotas and hackathon credits

### 6.1 Rate limits (per team, not per key)

| Model | RPM | TPM | Other |
|---|---|---|---|
| `muse-wonder-1.0` | 3,000 | 4,000,000 | — |
| `muse-spark-1.3` | 3,000 | 4,000,000 | — |
| `muse-spark-1.2` | 3,000 | 4,000,000 | — |
| `muse-spark-1.1` | 3,000 | 4,000,000 | — |
| `muse-spark-1.3-contributor` | 150 | 3,000,000 | — |
| `muse-spark-1.2-contributor` | 150 | 3,000,000 | — |
| `sam-3.1` | 3,000 | none | billed per image/frame |
| `muse-image-1.0` | 150 | none | billed per image |
| `muse-voice-transcribe-1.0` | none | none | 256 concurrent streams; 16,000 streams/hour |

Additional: background response creation is capped at **600 submissions/minute per team**. Successful token-based responses carry `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-tokens`, `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests`. 429 responses include `Retry-After`; the docs recommend exponential backoff with jitter starting at 500 ms.

Source: <https://dev.meta.ai/docs/pricing-rate-limits>

### 6.2 Quotas and spend

- Files storage: 100 GiB per team across all keys; files do not expire.
- Background request body: 1 MiB.
- No documented free tier, trial credit or default spend cap on the Model API pages I read. Spend-limit alerts exist as a billing feature (listed in the Help Center), not as a documented quota.
  Source: <https://dev.meta.ai/help>

### 6.3 Hackathon credits

Meta's event page confirms the dates, the $1M prize pool, the two tracks, and the model lineup, but **does not state a credit amount**.
Source: <https://dev.meta.ai/events/global-hackathon>

The **official co-organizer page** (Cerebral Valley, linked from Meta's event page) does state the credits:

| Item | Amount |
|---|---|
| Meta Model API credits for selected participants | **$150**, valid for a 10-day development window |
| Baseten | $100 credits + 2M TPM rate limits (select participants, open-weight models) |
| Modal | $100 credits (select participants, open-weight models via Endpoints) |
| Fireworks | $25 credits (select participants, open-weight models) |

Source: <https://cerebralvalley.ai/e/meta-ai-developer-hackathon>

Caveat: this is the co-organizer's page, not Meta's docs. It is the page Meta links to for applications and rules, so it is the best available primary-ish source, but treat credit delivery and eligibility as "confirmed at acceptance", not as a guaranteed quota.

---

## 7. Unverified / not found

Everything in this section was looked for and **not** confirmed in a primary source. Do not build on it.

1. **`$150` credit amount on a Meta-owned domain.** Only the Cerebral Valley co-organizer page states it. Meta's event page omits it. Terms live behind the "Credit Terms" PDF linked from that page, which I did not parse.
2. **Muse Model License terms.** The HF repo `meta-models/Muse-Spark-Open-816B-A42B` returns HTTP 401 anonymously (gated). I verified only the licence *name* and link, not commercial-use thresholds, redistribution rules, or output-ownership clauses.
3. **Muse Image maximum `n`, maximum reference images per turn, and maximum output resolution.** The docs describe `n`, `size` as an aspect ratio, and 16:1 rejection, but publish no upper bounds for `n` or reference-image count.
4. **Muse Image function/tool calling.** It is described as agentic with internal search and shell, but there is no documented `tools` function-calling interface for `muse-image-1.0`.
5. **Structured output for open-weight models.** Not documented for Muse Spark Open or Muse Glimmer. Fireworks mentions JSON mode generically for Spark Open; that is a provider feature, not a Meta guarantee.
6. **Muse Glimmer serverless availability.** Fireworks' page says serverless was deprecated 2026-09-25; Baseten exposes only a dedicated deployment; Modal exposes dedicated endpoints. I found no current serverless Glimmer offering among the three. Meta's Glimmer page lists Fireworks and Together as launch partners, but the deploy guide documents only Together.
7. **Muse Glimmer and Muse Spark Open per-token prices on Baseten/Modal/Fireworks.** Baseten and Modal publish Spark Open prices; neither publishes a Glimmer per-token price. Fireworks publishes Spark Open prices but not Glimmer's.
8. **`ato` language support.** No Meta documentation mentions ato. The models emit text and are documented for Python/TypeScript examples, so ato is plausible but unverified.
9. **Groq Muse Spark Open pricing and full context.** Meta's docs say Groq requires org approval and give a 131,072-token context; no price is published on the Meta page.
10. **Whether `muse-wonder-1.0` preview status has an expiry or SLA.** Docs say "preview" with no date.

---

## 8. Source index

**Meta documentation**
- <https://dev.meta.ai/docs/overview>
- <https://dev.meta.ai/docs/models>
- <https://dev.meta.ai/docs/pricing-rate-limits>
- <https://dev.meta.ai/docs/api-reference>
- <https://dev.meta.ai/docs/protocols>
- <https://dev.meta.ai/docs/protocols/responses>
- <https://dev.meta.ai/docs/sdks>
- <https://dev.meta.ai/docs/tool-calling>
- <https://dev.meta.ai/docs/tool-search>
- <https://dev.meta.ai/docs/structured-output>
- <https://dev.meta.ai/docs/reasoning>
- <https://dev.meta.ai/docs/image-generation>
- <https://dev.meta.ai/docs/image-understanding>
- <https://dev.meta.ai/docs/video-understanding>
- <https://dev.meta.ai/docs/speech-to-text>
- <https://dev.meta.ai/docs/media-segmentation>
- <https://dev.meta.ai/docs/sam/overview>
- <https://dev.meta.ai/docs/sam/segmenting>
- <https://dev.meta.ai/docs/sam/reading-segmentation>
- <https://dev.meta.ai/docs/sam/client-libraries>
- <https://dev.meta.ai/docs/file-handling>
- <https://dev.meta.ai/docs/token-counting>
- <https://dev.meta.ai/docs/search-grounding>
- <https://dev.meta.ai/docs/agent-frameworks>
- <https://dev.meta.ai/docs/muse-code>
- <https://dev.meta.ai/docs/muse-spark-open>
- <https://dev.meta.ai/docs/muse-spark-open/get-the-model>
- <https://dev.meta.ai/docs/muse-spark-open/deploy>
- <https://dev.meta.ai/docs/muse-spark-open/hardware-platforms>
- <https://dev.meta.ai/docs/muse-spark-open/settings>
- <https://dev.meta.ai/docs/muse-glimmer>
- <https://dev.meta.ai/docs/muse-glimmer/get-the-model>
- <https://dev.meta.ai/docs/muse-glimmer/deploy>
- <https://dev.meta.ai/docs/muse-glimmer/quantization>

**Meta product / event pages**
- <https://dev.meta.ai/models/muse-wonder>
- <https://dev.meta.ai/models/muse-spark>
- <https://dev.meta.ai/models/muse-image>
- <https://dev.meta.ai/models/muse-glimmer>
- <https://dev.meta.ai/models/muse-spark-open>
- <https://dev.meta.ai/events/global-hackathon>
- <https://dev.meta.ai/help>

**Meta research / Hugging Face**
- <https://ai.meta.com/blog/segment-anything-model-3/>
- <https://huggingface.co/meta-models/Muse-Spark-Open-816B-A42B> (gated)
- <https://huggingface.co/meta-models/Muse-Glimmer-30B>
- <https://huggingface.co/meta-models/Muse-Glimmer-30B-GGUF>

**Inference partners**
- <https://www.baseten.co/library/muse-spark-open>
- <https://www.baseten.co/library/muse-glimmer-30b>
- <https://fireworks.ai/models/fireworks/muse-spark-open>
- <https://fireworks.ai/models/fireworks/muse-glimmer-30b>
- <https://modal.com/library/meta/muse-spark-open>
- <https://modal.com/library/meta/muse-glimmer-30b-nvfp4>

**Event co-organizer (non-Meta, credit details only)**
- <https://cerebralvalley.ai/e/meta-ai-developer-hackathon>
