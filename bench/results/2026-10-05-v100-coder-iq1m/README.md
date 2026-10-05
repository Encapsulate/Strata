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

## Direct coding-task qualification

The route was then given a substantive code-generation request: implement a
typed, thread-safe Python 3.11 TTL/LRU cache module and focused pytest tests,
including expiry, LRU eviction, statistics, and concurrent `get_or_set`.
Thinking was disabled, temperature was 0, and the output ceiling was 2,048
tokens.

| Prompt tokens | Completion tokens | Wall time | Decode rate | Finish | Result |
|---:|---:|---:|---:|---|---|
| 157 | 1,855 | 55.5 s | 33.5 tok/s | stop | completed implementation and tests; required structural markers present |

The same task at 512 and 1,024 output tokens reached the requested cap and
was intentionally recorded as truncated. This demonstrates that output
budget, not only model speed, matters for complete coding responses on a
single 32 GB V100.

## Matched realistic review comparison

Using the same actual Strata Python/CUDA/CMake excerpts and review prompt as
the Swift benchmark, with a unique prefix and `0 reused` tokens:

| Prompt tokens | Reused | Prefill tok/s | Completion | Decode tok/s | Wall time |
|---:|---:|---:|---:|---:|---:|
| 2,828 | 0 | 19.6 | 384 | 8.3 | 191.0 s |
| 6,370 | 0 | 29.8 | 384 | 10.8 | 249.7 s |

Swift measured 19.4/29.4 prompt tok/s and 8.5/11.5 decode tok/s on the same
two requests. This is the realistic agent-facing comparison; the earlier
numbered-word probe is retained only as a controlled engine test.

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
