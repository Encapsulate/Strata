# Encapsulate Tesla V100 32GB / Volta SM70 benchmark

Date: 2026-10-05  
GPU: Tesla V100-SXM2-32GB, driver 580.173.02  
Engine: Strata `0.1.31`, active binary SHA-256 `1c052fb3007a89e518c66f12c1ca64caf16848eb70b585483913b04d9177b8ea`  
Source: `v100-sm70`, runtime source commit `263213c`  
Model: `swift-1.5-iq3_xxs`  
Runtime: `--prefill 2048 --spec 4 --kv int8 --kv-resident 32768 --max-context 131072`  
Loaded model footprint: 31,994 MiB VRAM at the final idle sample; 59.1 GiB system RAM used; 32,768 MiB VRAM total.  

## Results

The prompt text was generated deterministically as numbered `benchmarkN` words. `prompt_tokens` is the tokenizer count
reported by Strata. Each request generated one token.

| Prompt words | Prompt tokens | Prompt/prefill ms | **Prefill tok/s** | Wall time | Decode tok/s | Result |
|---:|---:|---:|---:|---:|---:|---|
| 2,048 | 9,185 | 65,036 | **141** | 65.1 s | 32.4 | completed |
| 4,096 | 19,425 | 21,471 | **905** | 21.6 s | 31.3 | completed |
| 8,192 | 39,905 | 27,127 | **1,471** | 27.4 s | 38.6 | completed |

`Prefill tok/s` is `prompt_tokens / prompt_ms * 1000`, taken from Strata's request metrics. `Decode tok/s` is the
one-token generation rate reported for the same request and is not a meaningful long-answer throughput measurement.

The first request included cold/request setup effects and had no useful prefix reuse. Later requests benefited from
resident experts/cache state and reusable prompt prefixes. The model cold-load took about 198 seconds before the
service became ready. The important result is that a long prompt completed without the previous SM70 stall while the
engine used the safe 2,048-token prefill chunk.

## Reproduce the measurement

Start the V100 server, wait for `/health` to report `"loaded":true`, then send an OpenAI-compatible request. The
benchmark used deterministic numbered words, `max_tokens: 1`, `temperature: 0`, and `stream: false`; Strata's
`/metrics` endpoint supplied `prompt_ms`, `prompt_tokens`, and `decode_tok_s`.

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
