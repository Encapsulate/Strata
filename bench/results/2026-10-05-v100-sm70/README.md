# Encapsulate Tesla V100 32GB / Volta SM70 benchmark

Date: 2026-10-05  
GPU: Tesla V100-SXM2-32GB, driver 580.173.02  
Engine: Strata `0.1.31`, active binary SHA-256 `1c052fb3007a89e518c66f12c1ca64caf16848eb70b585483913b04d9177b8ea`  
Source: `v100-sm70`, runtime source commit `263213c`  
Model: `swift-1.5-iq3_xxs`  
Runtime: `--prefill 2048 --spec 4 --kv int8 --kv-resident 32768 --max-context 131072`  
Loaded model footprint: 31,994 MiB VRAM at the final idle sample; 59.1 GiB system RAM used; 32,768 MiB VRAM total.  

## Matched realistic coding-review benchmark

Swift received the same prompts as the Coder route: actual Strata
Python/CUDA/CMake repository excerpts plus a senior-maintainer request to find
three correctness/performance risks and propose a minimal patch plan. Each
prompt had a unique context prefix and Strata reported `0 reused` tokens. The
answer cap was 384 tokens and the task was run with temperature 0 and thinking
disabled.

| Model | Prompt tokens | Reused | Prefill tok/s | Completion | Decode tok/s | Wall time |
|---|---:|---:|---:|---:|---:|---:|
| Swift IQ3_XXS | 2,828 | 0 | 19.4 | 384 | 8.5 | 190.9 s |
| Coder IQ1_M | 2,828 | 0 | 19.6 | 384 | 8.3 | 191.0 s |
| Swift IQ3_XXS | 6,370 | 0 | 29.4 | 384 | 11.5 | 250.0 s |
| Coder IQ1_M | 6,370 | 0 | 29.8 | 384 | 10.8 | 249.7 s |

These are the useful agent-facing numbers for this V100 deployment. The
similar prompt rates show that the shared SM70 runtime, KV streaming, and
host/RAM path dominate this workload; the IQ1_M Coder model does not deliver a
magical 1,000+ tok/s real-agent experience.

The earlier numbered-word/one-token run is retained only as a controlled
engine probe in the project history. It included warm/cache effects and is not
comparable to this task-level benchmark.

## Reproduce the measurement

Start the V100 server, wait for `/health` to report `"loaded":true`, then send
the same repository-review prompt to each route. Use a unique prefix for every
run, `max_tokens: 384`, `temperature: 0`, thinking disabled, and `stream: false`.
Confirm the Strata log reports `0 reused` before recording `prompt` and
`decode` rates. The older numbered-word command below is only a small engine
smoke probe, not the realistic benchmark above.

```bash
curl http://127.0.0.1:18082/health
curl http://127.0.0.1:18082/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"swift-1.5-iq3_xxs","messages":[{"role":"user","content":"Reply with exactly: benchmark"}],"max_tokens":1,"temperature":0,"stream":false}'
curl http://127.0.0.1:18082/metrics
```

`--prefill 2048` is the internal batch size for processing a long prompt; it is not a 2,048-token context limit.
The service still advertises a 131,072-token context. The earlier failure was caused by the generic 8,192-token
*prefill chunk* exhausting V100 runtime headroom, not by an 8,192-token prompt being unsupported.
