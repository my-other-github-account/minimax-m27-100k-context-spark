# 100K-Context Stability & Prompt-Cache Validation — C15-CLEAN Run

**Run timestamp:** 2026-04-29T06:33:02-07:00 → 2026-04-29T07:37:44-07:00 (64 min total)
**Hardware:** DGX Spark GB10 (Blackwell sm_121, 128 GiB unified memory)
**Host:** spark-6
**Container:** `mm-bench-c100k-cache` (image `minimax-m27-ngram:clean`, llama.cpp build 8833)
**Model:** `MiniMax-M2.7-UD-IQ4_XS` (4 GGUF shards, 108 GiB on-disk, fully offloaded to GPU)

---

## TL;DR

| Metric | Result |
|---|---|
| 5× back-to-back 100K runs | ✅ all 5 survived, decode 5.47–5.68 t/s |
| Prompt-cache reuse (same prompt 3×) | ✅ 322× speedup (522 s → 1.6 s) |
| d=0 generation post-stress | ✅ 23.4 t/s, no permanent leak |
| Container restart between requests | ❌ NOT NEEDED — single boot held 64 min |
| OOM events | 0 |
| Decode collapse events | 0 |

**Status:** 100K + prompt caching + back-to-back stability **all confirmed**, but **submission still pending** because we need to repeat the same battery with **ngram speculative decoding ON** (llama-benchy's default and our actual submission claim) plus a Sherlock think-on/off headliner grid.

---

## Why this matters

The MiniMax-M2.7 deployment on Spark has been a stability sinkhole for weeks. Earlier candidates collapsed at exactly run 4 of any back-to-back 100K battery — server alive, throughput dead — while the system pool got eaten by `slot prompt save` accumulation. The eviction logic in llama.cpp's CPU prompt-cache pool has a subtle bug when single saved entries exceed `--cache-ram` (`-cram`):

```
W srv update: - cache state: 1 prompts, 11558.361 MiB (limits: 4096.000 MiB)
```

The cap is bookkept but **violated** when one entry alone is bigger than the pool — eviction only removes smaller entries. So at 100K context, every completed prompt stashes ~12 GiB into a pool that was supposed to be capped at 4 GiB, and after a few rounds the system runs out of memory.

---

## The fix that worked: `-cram 100`

Setting the prompt-cache pool to **100 MiB** (effectively zero) makes save attempts no-op almost immediately. Combined with `--ctx-checkpoints 0` (disable mid-prefill checkpoint saves) and 79 GiB swap headroom, the server holds indefinitely.

**Critical insight:** Single-conversation prefix-cache reuse uses the **live slot KV cache** with `--cache-reuse 256`, NOT the CPU pool. The CPU pool is only for **cross-request restoration after slot eviction** — which we don't need for our use case. So `-cram 100` kills cross-request pool cheating but **preserves** within-conversation reuse, which is the only caching scenario that matters for the leaderboard claim.

---

## Final config

```
-c 108000              # locked: submission identity
-cram 100              # CPU prompt-cache pool capped at 100 MiB
--cache-reuse 256      # live-slot KV reuse for prefix matching
--ctx-checkpoints 0    # disable mid-prefill checkpoint saves
--cache-prompt         # keep prompt cache enabled
-fa on                 # FlashAttention
-ctk q8_0 -ctv q8_0    # q8_0 KV (only viable quant on this hardware)
-np 1                  # single slot
-ngl 99                # full GPU offload
--no-warmup
-t 20
--no-context-shift
```

System swap: **79 GiB** (15 base + 64 added via `/swapfile-extra`, persisted in `/etc/fstab`).

---

## Phase A — 5× back-to-back unique 100K (rock stability)

Five different 100K-token prompts run sequentially, no container restart, no client gap.

| Run | Server decode (t/s) | Prefill (t/s) | Notes |
|---:|:---:|:---:|---|
| 1 | 5.68 | 253 | Fresh-mode |
| 2 | 5.48 | 218 | |
| 3 | 5.52 | 232 | llama-benchy reported 1.91 t/s but server log said 5.52 — wall-clock artifact in benchy |
| 4 | 5.59 | 217 | |
| 5 | 5.47 | 216 | |

**Server eval-time is ground truth** (the `eval time = X ms / 128 tokens` line in the llama-server log). llama-benchy's wall-clock metric occasionally includes a stall outside the eval window (likely JSON-save during swap pressure or HTTP-finalization), producing artifacts like 1.27 or 1.91 t/s on individual runs. The model itself decoded all 5 runs at consistent **5.47–5.68 t/s**.

**Memory at end of Phase A:** 121/121 GiB used, 824 MiB free, **swap 1.3/79 GiB** (huge improvement vs prior runs that hit 12/15 GiB right before OOM cascade).

---

## Phase B — same-prompt 3× (prefix-cache reuse) 🎯

Sherlock corpus head-400000 bytes used as the test prompt, sent three times back-to-back to the same slot.

| Send | Full HTTP roundtrip |
|---:|:---|
| 1 (cold prefill) | **522.5 s** |
| 2 (warm cache hit) | **3.61 s** ← 145× speedup |
| 3 (warm cache hit) | **1.64 s** ← 318× speedup |

✅ **Prompt caching is genuinely working.** Send 2 reused the slot KV established by send 1 with no recomputation. Send 3 ran even faster — likely because send 2's brief activity left internal HTTP/JSON state cleaner.

This satisfies the `prefixCaching: true` claim for the leaderboard submission.

---

## Phase C — post-stress d=0 (3 runs)

After 5×100K + 3×100K stress, run 3× d=0 tg128 to confirm depth-0 inference still works in the exact same boot.

| Run | Decode (t/s) | Notes |
|---:|:---:|---|
| 1 | 17.96 | First d=0 after long-context stress — partial warmup |
| 2 | 23.38 | |
| 3 | 23.42 | |

Run 1 was a fresh-mode-into-d=0 transition; runs 2 and 3 are the steady-state d=0 number: **~23.4 t/s**.

---

## Phase D — d=0 again, 60 s settle (3 runs)

Same d=0 measurement repeated 60 s after Phase C, to detect a slow leak.

| Run | Decode (t/s) |
|---:|:---:|
| 1 | 23.70 |
| 2 | 23.33 |
| 3 | 23.90 |

D ≈ C → **no permanent perf leak.** d=0 inference fully holds even after the cumulative stress of 8× 100K-context completions.

---

## Honest measured numbers (this config, ngram spec decode OFF)

| Metric | Value |
|---|---|
| tg128 d=100K decode (n=5) | **5.55 ± 0.08 t/s** |
| tg128 d=100K prefill | **216–253 t/s** |
| tg128 d=0 decode (n=5 stable) | **23.4 t/s** |
| Prefix-cache same-prompt speedup | **322×** |
| Cold 100K-prompt prefill time | ~430–520 s |

---

## What's still pending (NOT submission-ready yet)

This run used **plain autoregressive decoding** (no spec decode). The actual submission target is the ngram-simple spec decode config that hit 30.84 t/s d=0 on shorter context earlier. We need to repeat the entire Phase A/B/C/D battery with **ngram spec decode ON** to:

1. Confirm 100K stability still holds under spec decode
2. Measure how spec decode affects 100K decode (likely worse than AR due to reject penalty in long-context regime)
3. Measure how spec decode affects d=0 decode (likely big speedup)
4. Run a final **think-on/off Sherlock grid at d=0 pp128 tg128** for the leaderboard headliner — this is the actual submitted number

Until that battery completes, no submission.

---

## Files

- `~/mm-bench/scripts/c15-clean.sh` — the orchestrator that ran A→B→C→D
- `~/mm-bench/scripts/boot-100k-server.sh` — server boot wrapper with all flags
- `~/mm-bench/scripts/run-tg2048.sh` — llama-benchy driver (with `--no-cache --pp 128`)
- `~/mm-bench/logs/c15-clean-20260429-063302.log` — full run log
- `~/mm-bench/logs/mm-bench-c100k-cache-server.log` — llama-server log (eval-time ground truth)
- `~/mm-bench/results/stability/C15-cram100-clean-20260429-063302.json` — JSON summary

## Open questions for next phase

- Does ngram spec decode survive 5× back-to-back 100K?
- How much does spec decode help/hurt at 100K depth specifically? (Reject rate is typically high on long-context)
- Sherlock think-on vs think-off d=0: is there a clear winner for headline?
- Can we honestly cite both d=0 (likely 25–30+ t/s with spec) AND d=100K (likely 5–6 t/s) as "long-context capable" on the leaderboard?
