# Stable IQ1/IQ3 Strata routes for Harness, Hermes, and Open WebUI

This is the repeatable setup that stopped the local IQ1/IQ3 routes from
falling into long thinking dives, one-token repetition, and stalled-looking
requests.

It was qualified on one Tesla V100-SXM2 32 GB (Volta SM70), but the routing
and request-normalization rules also apply to other GPUs. Adjust the runtime
memory settings for your hardware.

## What this fixes

The failures were not one single model bug. They came from several conditions
stacking together:

- A client could omit the thinking switch, while the Qwen/Strata template
  defaulted to deep thinking.
- IQ1 could receive greedy or overly narrow sampling and repeat the same token.
- A long agent history, tools catalog, or RAG payload could leave too little
  output/context headroom.
- The generic Volta prefill chunk could exhaust temporary VRAM and appear to
  stall.
- Restarting a model during an active generation could turn a slow request into
  a corrupted or repeated session.

This setup reduces those failure modes. It does **not** make a quantized model
factually infallible: use RAG, tools, citations, and an independent check for
claims that matter.

## Use one gateway for every client

Point DeepSeek Harness, Hermes, and Open WebUI at the same OpenAI-compatible
gateway, for example:

```text
http://127.0.0.1:11435/v1
```

Do not give each client a separate model loader or point some clients directly
at the Strata worker unless you are deliberately debugging. The gateway should
own model handoff so only one heavy Strata route occupies a 32 GB GPU at once.

Expose these stable model IDs:

| Model ID | Quantization | Use |
|---|---|---|
| `swift-1.5-iq3_xxs` | IQ3_XXS | general chat, reasoning, tools, long prompts |
| `qwen3.8-flash-next-coder-iq1_m` | IQ1_M | coding and editing |

The IQ1 route is the more repetition-prone route, so it needs the stronger
sampling guard below.

## Volta runtime settings

For an SM70/V100 build, use the V100 branch and the safe prefill setting:

```bash
git clone --branch v100-sm70 https://github.com/Encapsulate/Strata.git
cd Strata
cmake -S . -B build-v100 -G Ninja -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES=70 -DSTRATA_ENABLE_CUDA=ON \
  -DSTRATA_EXPERIMENTAL_SM60=ON -DSTRATA_BUILD_TESTS=OFF \
  -DSTRATA_NATIVE_EXPERTS=ON
ninja -C build-v100 strata
```

Use `--prefill auto` on SM70. The V100 source maps that to the validated
2,048-token internal chunk. It does **not** reduce the context window to 2,048
tokens; the tested context remains 131,072. A generic 8,192-token prefill
chunk can consume the V100's temporary headroom and look like a hang.

The tested long-context baseline was:

```text
--expert-cache auto --prefill 2048 --spec 4 --spec-min-p 0.5
--kv int8 --kv-resident 32768 --max-context 131072
```

Cold model loading can take minutes. Wait for the worker to become ready before
judging a simple `hi` request. Do not reload the worker while it is generating.

## The anti-loop sampler guard

Apply this normalization in the shared gateway immediately before forwarding a
Strata chat request. `setdefault` is intentional for IQ3; IQ1 uses floors so a
client cannot accidentally request greedy decoding.

```python
if model == "swift-1.5-iq3_xxs":
    payload.setdefault("temperature", 0.70)
    payload.setdefault("top_p", 0.90)
    payload.setdefault("top_k", 40)
    payload.setdefault("repetition_penalty", 1.06)
    output_limit = 32768

elif model == "qwen3.8-flash-next-coder-iq1_m":
    if not isinstance(payload.get("temperature"), (int, float)) or payload["temperature"] <= 0:
        payload["temperature"] = 0.65
    if not isinstance(payload.get("top_p"), (int, float)) or payload["top_p"] < 0.90:
        payload["top_p"] = 0.90
    if not isinstance(payload.get("top_k"), int) or payload["top_k"] < 32:
        payload["top_k"] = 40
    if not isinstance(payload.get("min_p"), (int, float)) or payload["min_p"] < 0.04:
        payload["min_p"] = 0.04
    repetition_penalty = payload.get("repetition_penalty")
    if (not isinstance(repetition_penalty, (int, float))
            or repetition_penalty < 1.10):
        payload["repetition_penalty"] = 1.12
    payload.setdefault("penalty_last_n", 512)
    output_limit = 8192

for key in ("max_tokens", "max_completion_tokens"):
    if isinstance(payload.get(key), (int, float)):
        payload[key] = min(int(payload[key]), output_limit)
```

The IQ1 output cap is deliberate. A bounded, clean answer is preferable to a
long corrupted continuation. Let the harness compact and continue when more
work is genuinely needed.

## Thinking control: none, low, or medium

Normalize all client spellings at the gateway and pass the model-native switch.
For Strata, the reliable controls are `chat_template_kwargs.enable_thinking`
and the request-level `think` boolean. Map `xhigh` to `high` if a client sends
it; Strata supports `none`, `low`, `medium`, and `high`.

For ordinary conversation, default an omitted control to thinking **off**:

```json
{
  "model": "swift-1.5-iq3_xxs",
  "messages": [{"role": "user", "content": "hi"}],
  "chat_template_kwargs": {"enable_thinking": false},
  "think": false,
  "max_tokens": 512
}
```

For low or medium thinking, send the explicit client effort and enable the
template switch. Do not insert a textual `<think>` marker and assume it will
control the model; that marker was unreliable across the Ollama and Strata
paths.

```json
{
  "reasoning_effort": "low",
  "chat_template_kwargs": {
    "enable_thinking": true,
    "reasoning_effort": "low"
  },
  "think": true
}
```

If a user says “none” or “off”, set `think: false` and
`enable_thinking: false`. Hermes can still use tools, skills, and RAG with
thinking off; those are request capabilities, not a requirement to expose the
model's private reasoning.

## Context, RAG, tools, and continuation policy

Keep the full pipeline, but make its budget explicit:

- Keep RAG retrieval focused. Send the top relevant chunks, not the entire
  knowledge base.
- Keep only essential tools in the fast chat profile. Use the full catalog for
  an agent task, not for a greeting.
- Reserve output space. A 131K context window is not a promise that a request
  with a huge history, tools schema, and RAG payload can also produce a huge
  answer.
- Compact before the context is full. The tested local continuation preset used
  a compact summary budget of 8,192 tokens and allowed up to 50 repairable
  continuations.
- Start a fresh chat after a visibly corrupted or repeated session. Preserve
  the old session for diagnosis; do not repeatedly append to it.

This is why the stable middle profile can include RAG, skills, filesystem/Bash,
Mnemosyne, and essential tools. Excluding web/search, subagents, workflows, or
the full plan catalog is a latency profile choice, not a model requirement.
Enable those catalogs for tasks that need them, while keeping the short-chat
profile small.

## Verification checklist

Run these checks after changing the gateway or model route:

```bash
python -m py_compile path/to/your/gateway.py
curl -fsS http://127.0.0.1:11435/health
curl -fsS http://127.0.0.1:11435/v1/models
```

Then test each route with three separate fresh requests:

1. `hi` with thinking off and a 256–512 token cap.
2. A short coding or RAG question with thinking low or medium.
3. One tool-capable request with the normal tool schema.

Confirm that:

- the selected model ID is the one actually resident;
- the first request may be slow only while loading/warming;
- output is more than a repeated single token;
- `finish_reason` is `stop` or a known bounded limit;
- no request remains in an endless reasoning phase when thinking is off;
- RAG/tool calls still work through the same gateway.

For a real latency comparison, use an uncached prompt and report prompt tokens,
decode tokens, and whether the model was cold. Do not compare a one-token cache
probe with an end-to-end agent turn.

## If it still appears stalled

Check, in this order:

1. Is the worker cold-loading or actually serving `/health`?
2. Is another client holding the GPU or triggering a model handoff?
3. Is the request carrying a huge persisted history, tools catalog, or RAG dump?
4. Did the gateway forward `think: false`/`true` as intended?
5. Did the gateway apply the IQ1/IQ3 sampler guard and output cap?
6. Is an active generation still running? Wait for it before restarting.

Only after those checks should you stop and relaunch the worker. A restart is a
recovery action, not a sampling fix.

## What was verified locally

The fixes documented here were exercised on the V100 Strata branch:

- Swift IQ3_XXS and Coder IQ1_M both completed uncached repository-style
  prompts through the shared route.
- IQ1 completed a bounded coding qualification cleanly.
- The IQ1 anti-loop floors and Swift repetition guard passed payload/syntax
  checks.
- Native Q5/Swift thinking control was repaired with request-level `think`.
- Long-session compaction/continuation tests passed, including recovery from a
  prior repeated one-token continuation.
- The SM70 prefill guard prevented the previous long-prompt stall.

See the benchmark records linked from the repository README for hardware and
throughput details.
