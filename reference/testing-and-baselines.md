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
| L3 Benchmark | timings vs baselines (§6.2) | `/completion` `timings` with pinned prompts (1061 / 32755 tok, n_predict=256, temp=0) | after engine/config change |
| L4 Fidelity | MTP does not corrupt output | OrderedDict-style structured output byte-compare, MTP on | after engine change |
| L5 Regression | capability intact | 262K needle (3 depths, verbatim recall); vision 6-probe set if mmproj mounted | major engine/ctx change |
| L6 Soak | stability | 15min×2 harness: ASR/WS/TTS/llamaloop counters, **eviction=0, GPU ctx=budget** | before sign-off |

**Fresh-run rule:** verify on the exact tree/config being delivered — a
green run proves only the tree it ran on. No commit/config change after
the last test.

## 6.2 Known-good baselines + thresholds

### 6.2a RESIDENT — Qwen3.8-Flash-Next (flashnext, EngramHalo + ROCm 7.14)

| Check | Baseline (measured) | PASS threshold |
|---|---|---|
| LLM health | `{"status":"ok"}`; /v1/models lists `qwen3.8-flash-next` (multimodal) | exact |
| **pp4096** (7601 tok prompt) | **477.3 t/s** | ≥ 350 t/s (reference band: fork table 396-502; kingjones777 385) |
| tg256 @ d0 (prose) | 28.7-31.7 t/s (n-max 4→12) | ≥ 26 t/s |
| tg @ d4k | 33.5 t/s (NO depth cliff — tuned QSA gather; d0→d20k controlled re-test 24.3→22.0 t/s = −9.5%) | ≥ 25 t/s |
| code/file-rewrite decode | **42.1 t/s** (MTP accept 89.5%, n-max 12) | ≥ 38 t/s |
| production long-form decode (novel prose, 2k+ tok output, effective ctx 12-62k during decode) | **14.8-18 t/s** (2026-09-09 journal audit) | informational — expected, NOT a regression (see caveat) |
| MTP acceptance | 73-93% by workload | report, not a gate (workload-dependent) |
| load time (mmap cold-ish) | HEALTH_OK ~75s (93.7GB) | ≤ 3 min |
| max ctx | **262,144 + MTP** (slot 分配 + 30k needle HIT, pp385/tg20 @30k) | needle HIT |
| VRAM resident | 73.8/96GB (PLE disk-lazy by design) | informational — NOT "all weights resident" |
| vision (mmproj-F16) | red-square probe correct ("红色") | 1/1 + no garbage |
| lifecycle | unload→load real-HTTP suite | all checks pass |
| benchmark tooling | device-side scripts (pp4096/tg@d0/tg@depth + draft ladder) | cite in evidence |
| per-stack availability smoke | device-side script — start → health/generate → stop per family; q38 swap-proof includes resident restore (75s) | all stacks PASS; final = only the resident active |
| restore | weights absent → re-download + sha verify + per-stack smoke | sha matches recorded values; every family health-verified |

**Decode expectation caveat (read before comparing production numbers).**
The tg rows above are *benchmark conditions*: depth 0-4k, `n_predict=256`,
ngram-friendly content (acceptance 79-93%). Production long-form workloads
read much lower and that is expected — three-factor attribution:
1. MTP acceptance is workload-dominant: novel prose accepted/step 0.6-1.1 vs
   ~2+ on filler/code → decode regresses toward pure target rate.
2. Output length grows the KV during decode: a 20k-52k prompt plus a 2.6k-16.8k
   output decodes at effective ctx 12k-62k, not at the prompt depth.
3. Pure input depth is minor: controlled 1k→20k = −9.5% (24.3→22.0 t/s at
   256-tok output, same content, cold prefix).
Rough production bands: short-ctx short-output 20-24 t/s; long-form at
25-62k effective ctx 14.8-18 t/s; ngram-hit content (echo/code/copy) 26-49 t/s.


### 6.2b SECOND FAMILY — Qwen3.8-27B (q38-llama, OFFLOAD: weights on disk, sha verified, startable on demand; :8000 exclusive with the resident)

Reference for the ROCmFP4/q38rocm lineage; numbers measured on the machine
at 262K/32K configurations:

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

1. Build prompts on the relay host; calibrate via `/tokenize` to 1061 and
   32755 tokens (±5%).
2. POST to `/completion` **from the machine itself** (loopback — excludes
   network jitter): `{"prompt":…, "n_predict":256, "temperature":0.0,
   "cache_prompt":true, "stream":false}`.
3. Read server-side `timings`: `prompt_per_second` (prefill),
   `predicted_per_second` (decode).
4. Cache A/B: re-send the identical prompt; `cache_n` should ≈ prompt
   length and `prompt_n` ≈ 4.
5. GPU proof in parallel: `rocm-smi --showuse --showmemuse` every 2s
   during the 32K prefill.
6. Save all JSON + monitor logs as evidence files; cite them in the log.

## 6.4 Rollback paths (MUST exist before mutation, R10)

- **flashnext (current):** `script/linux/flashnext_rollback.sh` (gated
  `ROLLBACK=ROLLBACK`): stops flashnext → restores archived q38 engine/units
  (`/data/flashnext/rollback/q38-engine-archive.tar.gz`) → re-downloads the 27B
  GGUF via ghfast if missing (14.5GB) → starts q38-llama → health.
  Flash-Next itself is fully reconstructable from ModelScope
  (`unsloth/Qwen3.8-Flash-Next-GGUF`)+ the sidecar rename step +
  the family-file build recipes.
- Second LLM (q38): `script/linux/flashnext_rollback.sh` restores the archived
  engine/units and re-downloads weights; device-side `~/q38-*.sh` live in
  the archive (`/data/flashnext/rollback/`).
- Speech: `script/linux/speech_rollback.sh` (full uninstall; llama/lemonade
  untouched) → `speech_deploy.sh all` re-deploys self-contained (drill
  closed once).
- Boot: fstab/lemond/q38-hw-tweak/firewalld/wired-static are all
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
