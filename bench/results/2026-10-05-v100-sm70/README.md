# Encapsulate Tesla V100 32GB / Volta SM70 benchmark

Date: 2026-10-05  
GPU: Tesla V100-SXM2-32GB, driver 580.173.02  
Engine: Strata `0.1.31`, active binary SHA-256 `1c052fb3007a89e518c66f12c1ca64caf16848eb70b585483913b04d9177b8ea`  
Source: `v100-sm70`, commit `fff7a4f`  
Model: `swift-1.5-iq3_xxs`  
Runtime: `--prefill 2048 --spec 4 --kv int8 --kv-resident 32768 --max-context 131072`  

## Results

The prompt text was generated deterministically as numbered `benchmarkN` words. `prompt_tokens` is the tokenizer count
reported by Strata. Each request generated one token.

| Prompt words | Prompt tokens | Wall time | Prompt rate | Result |
|---:|---:|---:|---:|---|
| 2,048 | 9,185 | 65.1 s | 141 tok/s | completed |
| 4,096 | 19,425 | 21.6 s | 899 tok/s | completed |
| 8,192 | 39,905 | 27.4 s | 1,456 tok/s | completed |

The first request included cold/request setup effects. Later requests benefited from resident expert/cache state and
prefix reuse. The important result is that a long prompt completed without the previous SM70 stall while the engine
used the safe 2,048-token prefill chunk.

`--prefill 2048` is the internal batch size for processing a long prompt; it is not a 2,048-token context limit.
The service still advertises a 131,072-token context. The earlier failure was caused by the generic 8,192-token
*prefill chunk* exhausting V100 runtime headroom, not by an 8,192-token prompt being unsupported.
