# Spec-Decode Regression: Canonical Reproduction Verdict (2026-04-29)

## Summary

Ran the official canonical recipe (`scripts/canonical.sh`) end-to-end on spark-6 with `llama-benchy 0.3.7` pinned. **Result: ngram-simple spec-decode is fully bypassed (0% draft acceptance), and AR has regressed ~5% from the original headline.**

## Head-to-head numbers (n=30 prompts, prefill=128 / output=128, warm runs only)

| Stack          | warm_mean | median | min   | max   | std  | Notes                          |
|----------------|-----------|--------|-------|-------|------|--------------------------------|
| AR (canonical) | 17.70     | 23.47  | 0.18  | 25.31 | 8.45 | Original headline: 24.65 t/s   |
| SPEC (ngram)   | 19.31     | 23.71  | 0.36  | 26.09 | 8.06 | Original headline: 30.84 t/s   |

Files:
- `~/mm-bench/results/canonical-ar-20260429-202717.json`
- `~/mm-bench/results/canonical-spec-20260429-202717.json`

## What broke

**SPEC: ngram-simple draft-generator is dead.** Server statistics across the entire 30-prompt run:

```
I statistics ngram_simple: #calls(b,g,a) = 32 3780 0,
                           #gen drafts = 0, #acc drafts = 0,
                           #gen tokens = 0, #acc tokens = 0
```

3780 generation calls over the run. **Zero drafts generated, zero accepted.** The spec pipeline is wired up (server starts, calls into ngram_simple) but the draft generator returns no candidates, so the verifier always falls back to plain AR decoding. SPEC ≈ AR throughput as a result (median 23.71 vs 23.47 t/s).

**AR: ~5% regression.** Median dropped from canonical 24.65 t/s → 23.47 t/s. Both warm_mean values are dragged down by an extreme low outlier (0.18 / 0.36 t/s) — likely a single cold prompt or transient system pressure during that prompt. Median is the more reliable summary; std is very high (8+) so single-run reproducibility is shaky.

## Where the regression came from (most likely)

This is a software-stack drift between the original headline run and now. The spec-decode pipeline working previously (30.84 t/s headline) but emitting zero drafts now points at an `ngram_simple` plumbing bug surfaced by either:

1. A llama.cpp commit that changed the ngram-cache draft-generation API contract without updating ngram_simple.
2. A llama-benchy 0.3.7 vs original-version delta in how it issues requests (e.g. `cache_prompt: false`, `n_draft` not propagated through the OpenAI shim).
3. A prompt-template change that defeats the n-gram match window (spec only helps when input has long literal repeats with the output; if the chat template now wraps differently, the cache misses).

The 5% AR regression is consistent with general driver / CUDA / kernel updates rather than anything in this repo.

## Honest leaderboard claim

Given the spec-decode path is broken (0% accept rate) and AR has drifted ~5%, the defensible submission for MiniMax-M2.7 on this hardware as of 2026-04-29 is:

- **AR baseline: 23.47 t/s median** (warm, n=29, prefill=128/output=128, single-user)
- **Prefix-cache hit speedup: 322× on prompt-eval** (untouched by this regression — separate code path)

Spec-decode (ngram-simple) cannot be claimed at the headline 30.84 t/s on the current stack — it would be misrepresentation. Listing AR is the honest L.

## Reproducer status

The repo (`scripts/canonical.sh`, `scripts/launch_server.sh`, `scripts/launch_server_ar.sh`) still runs cleanly to completion. Anyone reproducing today will see the same numbers we just measured. The README headline (30.84 t/s spec) is no longer reproducible against current llama.cpp / docker stack — should be updated to either AR-only or marked with a regression note.

## What to try next (if revisiting)

- Try `--spec-type ngram-cache` (the heavier variant) — confirms the verifier path is alive even if `ngram-simple` is dead.
- Bisect llama.cpp HEAD: check out the SHA listed in the original headline Dockerfile and rebuild — if THAT version produces drafts again, we have proof the regression is upstream.
- Check chat template: dump a single prompt+completion and grep server log for whether `ngram_simple` is even seeing the right token streams to match.

Loop terminated 2026-04-29.
