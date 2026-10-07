# Phase 6 — Testing Standard, Baselines, Release Gate

Read this file IN FULL before declaring ANY deployment done.
MUST/NEVER/STOP = enforced (SKILL.md §0).

## 6.1 Testing standard (applies to every deployment change)

**Acceptance-first:** before executing a change, write numbered acceptance
criteria (one criterion = one yes/no sentence) into the workspace
an acceptance-criteria file, first line verbatim `> cap=5  stall=3×`. Iteration
bound: cap = 5 per sub-problem, stall = same criterion failing 3× in a row
→ STOP, change direction, never "retry slightly different".

**Evidence rule (R8):** every claim gets an on-disk artifact — log file,
JSON response, or a timing capture on disk. "Build passed"/"looks
right" is NOT evidence. Every FAIL log line carries a falsifiable
`diagnosis:` clause.

**Benchmark caliber rule (R13, MUST):**
- Prefill **MUST** be measured with ≥4096-token prompts. A ~30-token
  "prefill" reads ~40 t/s on this box while pp4096 reads **477 t/s** on the
  same server — small-prompt numbers are fixed-overhead noise, never
  comparable to published tables.
- Decode at d0 **AND** at depth (≥4k) — depth cliffs are arch+backend
  specific (stock #27742 cliffed 3.5× at 1k ctx on HIP).
- Speculative decoding: **read the output** before believing the counter
  (broken MTP ports emitted multilingual noise at "42 t/s"; a degenerate
  benchmark accepted its own echo at 35 t/s).

**Test levels (run in this order):**

| Level | What | Tool/command pattern | When |
|---|---|---|---|
| L1 Smoke | services up, identity right | `systemctl is-active` + `/health` + `/proc/<pid>/cmdline` MARK | after every (re)start |
| L2 Contract | API schemas byte-exact | per-family contract suites (ASR 14+19 cases; LLM /props,/v1 fields) | after deploy/config change |
| L3 Benchmark | timings vs baselines (§6.2) | `/v1/chat/completions` + pinned prompts (nonce-fresh ~25K; needle 219K, n_predict=256, temp=0); llama-server additionally `/completion` `timings`, Gufo = wall-time + `usage` | after engine/config change |
| L4 Fidelity | MTP does not corrupt output | OrderedDict-style structured output byte-compare, MTP on | after engine change |
| L5 Regression | capability intact | 262K needle (3 depths, verbatim recall); vision 6-probe set if mmproj mounted | major engine/ctx change |
| L6 Soak | stability | 15min×2 harness: ASR/WS/TTS/llamaloop counters, **eviction=0, GPU ctx=budget** | before sign-off |

**Fresh-run rule:** verify on the exact tree/config being delivered — a
green run proves only the tree it ran on. No commit/config change after
the last test.

## 6.2 Known-good baselines + thresholds

### 6.2a RESIDENT — Qwen3.8-Flash-Next (flashnext, **Gufo + UD-Q4_K_XL**, current stack)

| Check | Baseline (measured) | PASS threshold |
|---|---|---|
| LLM health | `/health` → `{"status":"ok"}`; `/v1/models` lists `qwen3.8-flash-next` (text+image, ctx 262144) | exact |
| fresh 25K prefill (nonce prompt) | **~1436 t/s** (60,748 tok / 42.3s end-to-end) | ≥ 1000 t/s |
| 200K needle (219,406 tok; needles at 10/50/90%) | **166.6s total; pp≈1317 t/s; 3/3** | ≤ 6 min AND 3/3 |
| decode (MTP draft-limit 7, filler/echo band) | 25-45 t/s measured (19.6 … 43.5 @219K) | informational — read the output |
| load | engine load 40.6s + unit start; health window | ≤ 3 min |
| GPU resident | 90.8/98.3GB (`amd-smi`) | informational |
| served-model-name | client name `qwen3.8-flash-next` accepted | no `model_not_found` |
| lifecycle | unload→load real-HTTP suite; container stop clean (`inactive`); `inactive\|failed` accepted | all checks pass |
| vision (mmproj-BF16) | red-square probe correct | 1/1 + no garbage |
| per-stack availability smoke | start → health/generate → stop per family | all stacks PASS; only the resident active at the end |
| restore | weights absent → re-download (ModelScope, sha 4/4 vs API) + smoke | sha matches; health-verified |

**Legacy-engine rows (do NOT gate against these):** an earlier stack on this
class of hardware measured pp4096 = 477.3 t/s, tg256 28.7-42.1 t/s (MTP n12),
VRAM 73.8/96GB — on UD-IQ4_XS with a different engine, and it carried a
heap-corruption defect (family file F4-2). Treat those numbers as platform
context only.

**Decode expectation caveat (read before comparing production numbers).**
The decode row is a *benchmark condition* (filler/echo content, high MTP
acceptance). Production long-form reads lower and that is expected —
attribution shifts on **MTP acceptance** (novel prose collapses acceptance)
and **effective ctx during decode** (output length grows the KV). Legacy-engine
service bands were: short-ctx short-output 20-24 t/s; long-form at 25-62k
effective ctx 14.8-18; ngram-hit content 26-49. For the current Gufo stack,
re-baseline from the unit journal after the first week in service, same
three-factor method.


### 6.2b SECOND FAMILY — Qwen3.8-27B (q38-llama) — historical reference

> **Status: historical reference** (restore recipe included). The baselines
> below are the ROCmFP4/q38rocm lineage reference, measured at 262K/32K;
> `:8000` remains exclusive with the resident.

| Check | Baseline (measured) | PASS threshold |
|---|---|---|
| LLM health | `{"status":"ok"}`; /props n_ctx=262144, slots=2 | exact |
| Short-ctx prefill (1061 tok) | 365.1 t/s | ≥ 300 t/s |
| Short-ctx decode (256 tok) | 32.3 t/s (MTP draft accept 180/198) | ≥ 28 t/s |
| 32K prefill (32755 tok) | 283.1 t/s (115.7s) | ≥ 230 t/s |
| 32K decode | 25.2 t/s (accept 189/191) | ≥ 22 t/s |
| prefix cache replay | prompt_n→4; 32K prefill 115.7s → **0.139s** | replay ≥ 100× faster (identical prompt only — see caveat) |
| GPU proof | during 32K prefill: GPU busy 90–100% (`rocm-smi --showuse`) | ≥ 85% |
| 262K needle | 3/3 verbatim recall; 219K first ask ~34min (107 t/s mean); same-prefix follow-up 61× KV reuse | 3/3 |
| 262K usage pattern | "stuff-once, ask repeatedly" works: fill 219K once, follow-ups reuse the KV (33.6s measured) | usage guidance, not a gate |
| stock engine fallback | stock ROCm0 @128K ctx, MTP on: 28.9 t/s decode | informational (daily-use fallback tier) |
| vision (if mounted) | 6/6 probes (Q8_0 = BF16); first image encode 400–700ms | 6/6 |
| ASR contract | 14+19 cases; 5min audio 3.3×realtime, 2 speakers, final 1145 chars | all green |
| Voice first-response SLO | text ~6.4s · voice turn ~11s (EoT 1.7 + LLM ~4 + TTS ~3) | >17s ⇒ co-card contention — investigate overlap |
| Soak | ASR 208/0 · WS 270/0 · TTS 90/0 · llama 45/45 · **eviction=0 · GPU ctx=2** | 0 errors, 0 evictions |
| NPU (if stack installed) | FLM decode 41.2 tok/s, TTFT 0.50s; liveness via `rocminfo` aie2p (NOT xrt-smi — enumerates 0 devices, known defect) | inference succeeds |

**Prefix-cache caveat (MTP spec boundary, measured):** the
replay numbers hold for **identical / same-prefix prompts**. For unrelated
prompts the cache load is frequently skipped with `spec-boundary-mismatch`
(cold fallback, `target-draft-restore-rejected` in the journal) — the
draft state must match the spec boundary. Multi-turn reuse rate is
therefore **lower than a no-MTP deployment**; do not read the 0.139s
replay as a general multi-turn guarantee.

Deviation beyond threshold → STOP, diagnose against the phase pitfall
sections, fix, re-run — never hand-wave a regression as "noise".

## 6.3 Benchmark method (reproducible)

1. Build prompts on the relay host; calibrate against the target engine:
   llama-server has `/tokenize` (1061 / 32755 tok ±5%); for Gufo-class
   engines target nonce-fresh ~24-25K and needle 219K and read back
   `usage.prompt_tokens` from the response.
2. POST **from the machine itself** (loopback — excludes network jitter):
   `/v1/chat/completions` for all engines; llama-server also exposes
   `/completion` **with** a `timings` object.
3. Read timings: llama-server = `timings.prompt_per_second` /
   `predicted_per_second`; Gufo = wall time + `usage`
   (`pp_eff = prompt_tokens / wall_seconds`).
4. Fresh-prefill control: nonce-prefix the prompt (or check `cache_n` on
   llama-server) — repeated prompts read from the slot cache and fake speed.
5. GPU proof in parallel: `amd-smi monitor` / `rocm-smi --showuse` during the
   big prefill.
6. Save all JSON + monitor logs as evidence files; cite them in the log.

## 6.4 Rollback paths (MUST exist before mutation, R10)

- **flashnext (current, Gufo):** keep the previous engine's unit file + the
  fallback model files on disk (UD-IQ4_XS + original MTP + mmproj-F16 for the
  Nathanw route) as the rollback layer. Full rebuild from zero: ModelScope
  `unsloth/Qwen3.8-Flash-Next-GGUF`
  (`UD-Q4_K_XL/*` + `MTP/mtp-…-shared-Q8_0.gguf` + `mmproj-BF16.gguf`,
  sha 4/4 vs the files API) + `ghcr.nju.edu.cn/gufo-org/toolboxes/gufo-runtime`
  image + the unit template (family file F3).
- q38 restore: `script/linux/flashnext_rollback.sh` (gated
  `ROLLBACK=ROLLBACK`): archived engine/units
  (`/data/flashnext/rollback/q38-engine-archive.tar.gz`) → re-download the 27B
  GGUF via ghfast if missing (14.5GB) → starts q38-llama → health.
- Second LLM (q38): `script/linux/flashnext_rollback.sh` restores the archived
  engine/units and re-downloads weights; device-side `~/q38-*.sh` live in the
  archive (`/data/flashnext/rollback/q38-extras`).
- Speech: `script/linux/speech_rollback.sh` (full uninstall; llama/lemonade
  untouched) → `speech_deploy.sh all` re-deploys self-contained (drill
  closed once).
- Boot: fstab/q38-hw-tweak/firewalld/wired-static are all
  boot-persistent; after a full reboot bring up services per unit enablement.

## 6.5 Release Gate — ALL must be green to declare done

| # | Gate | Evidence |
|---|---|---|
| RG1 | Phase Gates 1–4 all passed (each file's gate table) | gate outputs logged |
| RG2 | L1 smoke: all services active + identity MARKs correct | health/cmdline captures |
| RG3 | L2 contracts green | suite logs |
| RG4 | L3 benchmarks within §6.2 thresholds | timings JSON |
| RG5 | L4 fidelity byte-exact | signature diff empty |
| RG6 | L6 soak: 0 errors, eviction=0, GPU ctx=budget | soak harness log |
| RG7 | Rollback path executed or drilled at least once for the changed component | drill log |
| RG8 | All evidence on disk; acceptance criteria all pass | acceptance + verification log |

**Any red item: STOP — the deployment is not done.**
