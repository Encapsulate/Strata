# Encapsulate Tesla V100 32GB / Coder IQ1_M coding-scope benchmark

Date: 2026-10-05  
GPU: Tesla V100-SXM2-32GB, SM70, driver 580.173.02  
Engine: Strata `0.1.31`, locally rebuilt for CUDA architecture 70  
Model: `qwen3.8-flash-next-coder-iq1_m`  
Weights: 58,408,584,928 bytes / 54.40 GiB, two GGUF shards  
Context: 131,072 tokens  
KV: `int8`, 32,768 resident cells  
Runtime: `--prefill auto` selecting the validated 2,048-token chunk, MTP4

## Coding-scope results

The route was tested with thinking explicitly disabled, temperature 0, and
code-only output requests. The checks were structural, deterministic, and
intended to qualify repository coding usefulness rather than claim broad
software-correctness coverage.

| Probe | Prompt tokens | Completion tokens | Decode tok/s | Result |
|---|---:|---:|---:|---|
| Python LRU cache + TypeScript retry helper | 150 | 384 | 14.7 | passed structural checks; response reached the bounded cap |
| Python interval merging with validation | 65 | 192 | 10.3 | passed structural checks; response reached the bounded cap |

The checks required the expected class/function names and implementation
markers: `OrderedDict`/`move_to_end` for the LRU cache, `AbortSignal` and
`setTimeout` for retry, and `ValueError` plus sorting for interval merging.
The first combined probe was capped at 384 output tokens; its completion was
sufficient for the LRU and retry checks but ended at the cap.

## Reproduce

Start the Coder route through the local handoff gateway or its on-demand unit:

```bash
systemctl --user start strata-coder-iq1m.service
curl http://127.0.0.1:18082/health
```

Send an OpenAI-compatible request to
`http://127.0.0.1:18082/v1/chat/completions` with model
`qwen3.8-flash-next-coder-iq1_m` and:

```json
{"temperature":0,"max_tokens":192,"stream":false,
 "chat_template_kwargs":{"enable_thinking":false}}
```

The existing Swift IQ3_XXS benchmark remains separately documented at
`bench/results/2026-10-05-v100-sm70/README.md`.
