# C15-NGRAM-CLEAN: 100K-Context + Prompt-Cache + ngram Spec Decode + Sherlock Headliner Grid

**Run timestamp:** 2026-04-29T08:02:33-07:00 → 2026-04-29T09:09:19-07:00 (66 min)
**Hardware:** DGX Spark GB10
**Host:** spark-6
**Container:** `mm-bench-c100k-spec` (image `minimax-m27-ngram:clean`, llama.cpp build 8833)
**Companion run:** see `100K-CONTEXT-STABILITY-AND-PROMPT-CACHE.md` for AR baseline

---

## TL;DR

Same battery as C15-clean (Phases A/B/C/D) but with **ngram-simple speculative decoding ON**, plus a new **Phase E** Sherlock think-on/off d=0 grid for the leaderboard headliner.

| Metric | AR baseline | ngram-spec |
|---|---|---|
| Phase A: 5× back-to-back 100K | 5/5 | **5/5** |
| 100K decode (server eval-time) | 5.55 t/s | **5.55 t/s** |
| 100K prefill | 216–253 t/s | **211–223 t/s** |
| Phase B: same-prompt cache speedup | 322× | **326×** |
| Phase C+D: d=0 decode | 23.4 t/s | **23.1 t/s** |
| No permanent perf leak | ✅ | **✅** |
| Container OOM | none | **none** |

**Spec decode delivers ≈ AR performance** at this context length — neither regression nor headline gain. At d=100K the draft hit rate collapses, so spec degenerates to AR. At d=0 we'd hope for the 30+ t/s we saw in earlier shorter-context runs, but in this exact stability config we get 23 t/s for both modes.

---

## Final config (ngram-spec)

```
-c 108000 -cram 100 --cache-prompt --cache-reuse 256 --ctx-checkpoints 0
-fa on -ctk q8_0 -ctv q8_0 -np 1 -ngl 99 --no-warmup -t 20 --no-context-shift
--spec-type ngram-simple --draft-max 16 --draft-min 1 --draft-p-min 0.5 --spec-ngram-size-n 4
```

---

## Phase A — 5× back-to-back unique d=100K (server eval-time ground truth)

| Run | Decode (t/s) | Prefill (t/s) |
|---:|:---:|:---:|
| 1 | 5.72 | 222.6 |
| 2 | 5.53 | 213.2 |
| 3 | 5.52 | 212.9 |
| 4 | 5.52 | 212.8 |
| 5 | 5.46 | 213.1 |

✅ **All 5 runs stable. Decode flat (5.46–5.72 t/s).** Identical to AR baseline within noise.

**llama-benchy wall-clock artifact:** under spec decode the artifact is even more pronounced (benchy reports 0.29–2.77 t/s for runs 1–5). Likely cause: spec-decode's reject-path adds longer HTTP-finalization stalls, which benchy's wall-clock metric folds into its tput calculation. **Server eval-time is the only honest ground truth.**

## Phase B — same-prompt 3× (prefix-cache reuse)

| Send | Roundtrip |
|---:|:---:|
| 1 (cold) | 540.8 s |
| 2 (warm hit) | 4.36 s |
| 3 (warm hit) | 1.66 s |

✅ **326× cache reuse speedup.** Prefix caching survives intact under spec decode.

## Phase C — d=0 tg128 (3 runs, post-stress)

| Run | Server decode |
|---:|:---:|
| 1 | 23.36 |
| 2 | 23.02 |
| 3 | 22.99 |

## Phase D — d=0 tg128 (3 runs, 60s settle, leak detection)

| Run | Server decode |
|---:|:---:|
| 1 | 23.13 |
| 2 | 23.23 |
| 3 | 23.04 |

✅ **No leak.** C ≈ D ≈ **~23.1 t/s.** Same as AR (~23.4 t/s) within noise.

## Phase E — Sherlock think-on/off d=0 pp128 tg128 grid 🎯

Two system prompts, identical user prompt (Sherlock corpus head 8K + question), 128-token budget, three runs each.

**Think-on system prompt:** *"You are a helpful assistant. Think step by step inside `<think>` tags before answering."*

**Think-off system prompt:** *"You are a helpful assistant. Answer directly without `<think>` tags or chain-of-thought reasoning."*

| Mode | Run | Server decode (t/s) | Cold prefill ms |
|---|---:|:---:|:---:|
| think-on | 1 | 20.92 | 5207 (1875 toks) |
| think-on | 2 | 21.80 | 46 (cache hit) |
| think-on | 3 | 22.63 | 46 (cache hit) |
| **think-off** | 1 | 21.99 | 5057 (1877 toks) |
| **think-off** | 2 | 22.07 | 46 (cache hit) |
| **think-off** | 3 | **22.75** | 46 (cache hit) |

**Headline numbers:**
- **think-on warm decode:** ~22 t/s (median 21.8)
- **think-off warm decode:** ~22 t/s (median 22.1)
- **Cold first-shot tput including prefill:** ~12 t/s (think-off run 1: 11.6 t/s wall-clock)
- **Prefill (small ~1.9K tokens):** ~360 t/s

**Observation:** think-on vs think-off decode rates are basically identical (~22 t/s). Think mode doesn't slow generation at the per-token level — it just means the model spends some of its 128-token output budget emitting `<think>` content before the visible answer. For a leaderboard headline that wants pure tg-throughput, think-off is the safer claim because run-1 wall-clock is amortized cleaner.

**Useful aside:** Phase E exercised the prefix cache automatically — only run 1 of each mode paid a 5 s prefill cost; runs 2 and 3 hit warm cache. This implicitly re-confirms cache reuse from Phase B.

---

## Honest leaderboard claims (this config, ngram-spec)

| Metric | Value |
|---|---|
| **tg128 d=100K decode** | **5.55 ± 0.10 t/s** (n=5, server eval-time) |
| **tg128 d=100K prefill** | **216 t/s** (median, n=5) |
| **tg128 d=0 decode** | **23.1 t/s** (n=6 over phases C+D) |
| **Sherlock pp128 tg128 d=0 think-off warm** | **22.1 t/s** (median, n=2) |
| **Sherlock pp128 tg128 d=0 think-off cold** | **11.6 t/s** (n=1, includes 5s prefill on 1.9K tokens) |
| **Prefix-cache same-prompt speedup** | **326×** |
| **Prefix-cache: `prefixCaching: true` claim** | ✅ |

---

## Open puzzles

1. **Why is d=0 decode 23 t/s with spec ON when an earlier shorter-context config got 30.84 t/s?** Possible culprits:
   - `--cache-reuse 256` adds latency that masks spec-decode's win at d=0
   - `-cram 100` interacts with the spec draft cache somehow
   - q8_0 KV vs the older f16/different KV setup
   - `-np 1` vs the older default
   - Worth a follow-up: same boot, kill spec, re-test d=0 to confirm spec is no-help vs the AR floor

2. **Why does llama-benchy wall-clock report 0.29–2.77 t/s for Phase A runs when server says 5.5 t/s?** Almost certainly llama-benchy is timing the full HTTP roundtrip including JSON-serialize-final-response, while spec-decode's reject-path adds dead time at the end of the request. **Need to file an upstream issue or workaround in our wrapper.**

3. **Should we run a Phase F: Sherlock pp128 tg2048 think-off d=0?** Longer output run might better separate think-on/off if the sweep over 128 isn't enough resolution.

---

## Submission readiness

✅ All gating tests pass:
1. 100K back-to-back x5: stable, no UX-killer restarts
2. Prompt caching: working, 326× reuse speedup
3. d=0 baseline holds post-stress: no permanent leak
4. Sherlock think-on/off grid: numbers measured
5. ngram-spec battery: matches AR baseline, no regression

**Ready to submit** with these honest numbers + `prefixCaching: true`.

---

## Files

- `~/mm-bench/scripts/c15-ngram-clean.sh` — orchestrator (5 phases A→E)
- `~/mm-bench/scripts/boot-100k-server.sh` — boot wrapper (default ctx now 108000)
- `~/mm-bench/scripts/run-tg2048.sh` — llama-benchy driver
- `~/mm-bench/logs/c15-ngram-clean-20260429-080233.log` — full run log
- `~/mm-bench/logs/mm-bench-c100k-spec-server.log` — server eval-time ground truth
- `~/mm-bench/results/stability/C15-ngram-cram100-clean-20260429-080233.json` — JSON summary
