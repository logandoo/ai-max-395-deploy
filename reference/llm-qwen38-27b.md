# Phase 2 (family, HISTORICAL) — Qwen3.8-27B ROCmFP4 (llama.cpp / ROCmFPX)

> **Family recipe for the qwen35 arch / ROCmFP4 tensor-type lineage**
> (charlie12345/ROCmFPX + julianmb/q38rocm). On the reference deployment this
> stack runs offloaded: unit `q38-llama` inactive, weights on disk (sha256
> fb89c78d… verified), and the Flash-Next service owns :8000 — the two are mutually
> exclusive, so start q38-llama only after stopping flashnext. The generic
> procedure (model-deploy-generic.md) governs; this file supplies the family
> specifics and its pitfall set (B1-B9).

Read this file IN FULL before Phase 2. MUST/NEVER/STOP = enforced (SKILL.md §0).

## 2.1 Engine and model

- Repo: github.com/julianmb/q38rocm (single-model deployment package).
- **Engine dual-track (MUST understand before starting):**
  - `stock` = prebuilt v1.5.2 (`version: 215 (12f8b7e)`, sha256
    `70d11cec4fd6c148a050f80a0422d563a928c39f849e600d6b59b1d620820aa7`) —
    fine for 128K daily use and experiments.
  - `ovf` = engine-overflow source rebuild (ROCmFPX post-IU4 lineage) —
    **mandatory for 262K production** (pitfalls B2/B3).
- Acceptance probe: `llama-server --list-devices` **MUST** list both
  ROCm0 and Vulkan0.
- Model: ROCmFP4-FAST GGUF — 14,562,236,384 bytes, sha256
  `fb89c78d2be91cdb68eaaaa45b1270710bf34aa721dc1f0b9e3aa7b98d2e1da9`.
  HF unreachable from CN → **relay via Mac (R1)**, verify sha256 on device.

## 2.2 Production launch parameters (config-driven — MUST NOT hardcode elsewhere)

```
-dev ROCm0 -ngl 999 -c 262144 -np 2 -t 16
--no-mmap --cont-batching --kv-unified -ctk q8_0 -ctv turbo4
--spec-type draft-mtp --spec-draft-n-max 4 --reasoning off --temperature 0.0
```

- `--kv-unified`: **each slot gets the full 262144 ctx** (not split);
  two-slot parallel aggregation = +33% throughput.
- MTP (draft-mtp, n_max=4) is the 2.4–2.94× decode lever
  (no-MTP 13.4 t/s → MTP 32–37 t/s on this device).
- Optional vision: `QEXTRA="--mmproj <base>/mmproj/mmproj-Qwen3.8-27B-Q8_0.gguf"`
  (Q8_0 0.63GB measured equal to BF16, 6/6 probes; **on the ovf engine it
  coexists with MTP** — on stock 215 it MUST pair with `--no-mtp`, pitfall B4).
- Asymmetric TurboQuant KV (`-ctk q8_0 -ctv turbo4`) + Qwen3.8 hybrid
  attention (48 linear + 16 full layers) → sub-linear KV growth: 262K ctx
  ≈ 27GB VRAM of the 96GB pool.
- **NEVER enable YaRN beyond 262K** — measured 51–85 t/s and retrieval
  MISS; no practical value.

## 2.3 Service-ize (systemd — R4)

- ⛔ `nohup` from ssh belongs to the session cgroup and gets reaped when ssh
  drops (real outage) → run as `q38-llama.service`
  (`Restart=on-failure`, `TimeoutStartSec=900`).
- ⛔ SELinux: custom mounts (and possibly `$HOME` binaries on Workstation)
  are `unlabeled_t`/`user_home_t` → `chcon -t bin_t <llama-server-binary>`
  if the unit fails with `Permission denied`.
- Lifecycle: `script/linux/server_start.sh|stop|restart|status`
  (config-driven dual-engine routing + pidfile + **engine identity gate**).
- ⛔ **Binary name collision:** stock and ovf binaries are both named
  `llama-server`. Identity **MUST** be read from `/proc/<pid>/cmdline`
  MARK (`engine-overflow` vs `engine/bin` path segment) — never from the
  process name or a startup banner (a banner can mask a bind failure).

## 2.4 NPU positioning (read before "helpfully" adding NPU to production)

- Upstream's own measured verdict: **production = iGPU + embedded MTP**.
  Sustained NPU decoding loses to embedded MTP (standalone drafter 42.9
  t/s, degrades to ~14 under shared-DRAM-bus contention). NPU value is
  only: 1.8× long-prompt cold TTFT (870ms, but the burst's 24 tokens come
  from a 0.8B model and **cannot be corrected by the target model** =
  fidelity risk) and ~2W always-on intent routing.
- Full-stack install (acceptance was): `iommu=pt` → amdxdna autoload →
  XRT build → Lemonade 11.8.1 + FLM (qwen3.5-0.8b) → measured NPU
  41.2 tok/s decode, TTFT 0.50s.
- ⛔ Self-built XRT 2.26's `xrt-smi examine` enumerates 0 devices
  (incompatible with mainline amdxdna) — prove NPU liveness via
  `rocminfo` (aie2p) + an FLM inference, NOT xrt-smi.

### 2.4.1 Intent routing — current in-production config (the NPU offload target)

- The chatbot's intent classifier currently runs against the local LLM
  with: `intent_classify_max_chars=12` (classify only short fragment
  turns; text input always skips; long sentences skip),
  `fail-open=respond` (classifier failure never blocks a reply), 6s
  timeout harmless by design.
- This is the workload an `intent → NPU` task would offload onto
  qwen3.5-0.8b (~2W, zero iGPU contention). Do not redesign the policy
  without re-reading the intent-routing rationale in project memory.

## 2.5 Engine pitfalls (B-series)

1. F44 ROCm: only the el9 7.2.x path; manual closure needs
   `/etc/profile.d/rocm.sh` (system-base §1.3).
2. ⛔ Vulkan RADV crashes at ~50% of 262K ctx (vk::DeviceLostError,
   ggml#27154) → 262K is **ROCm0-only**; `--device auto` defaults to
   Vulkan and is a landmine at long ctx.
3. ⛔ stock 215's ROCm lane spins on CPU at 262K (GPU 0%, 14min without a
   dispatch) → 262K requires the ovf source rebuild.
4. ⛔ MTP+mmproj on stock 215: crash `missing MTP boundary ... pos=79` →
   vision then **MUST** use `--no-mtp`; ovf lineage fixed it.
5. `--reasoning off`: thinking lands in the `reasoning_content` field and
   `content` may be empty → **clients MUST read both fields**.
6. Strict MTP: `--spec-mtp-strict-qwen` gives token-exact parity with
   no-spec decoding (costs speed); default is non-strict (journal logs a
   warning) — choose deliberately.
7. `ngl=999` "failed to fit ... abort" is a harmless startup warning
   (explicit user ngl takes precedence).
8. Engine/template string substitutions MUST carry an assertion — else they
   fail silently (real A4.9 lesson).
9. "queue evicted" recovery = restart the process (HSA AsyncEventsLoop may
   busy-lock at 100% CPU, #6522) — clearing VRAM does nothing (see
   gpu-discipline.md).

## 2.6 Alternative speculative decoding — DFlash2 NO-GO (closed ruling)

The DFlash2 pilot (`docs/PILOT_20260830_dflash2_ab.md`) establishes:

- DFlash2 dual-drafter gives code +52% (56.3 vs 37.0 t/s) and looks
  greedy-safe, **BUT temp-0 output is not faithful** (≠ target-model
  argmax; OrderedDict-style structured outputs lose variants) and prose
  corrupts (prompt-echo / repetition). Root causes include
  cache×dflash cross-slot KV pollution; disabling the cache leaves a
  residual 1-of-2 mismatch.
- Loading requires the LZ fork prebuilt: stock 215 cannot load the
  DFlash2 GGUF (tensor layout generation mismatch "expected 81 got 58").
- **MUST NOT adopt for production.** Long-context work stays on embedded
  MTP. Re-evaluate only after re-reading the pilot AND an upstream
  fidelity fix (this ruling supersedes curiosity).

## 2.7 Optimization roadmap (zero-stack-change)

| Lever | Expected | Status |
|---|---|---|
| MTP n_max scan 4 → 9/12 | longer accepted drafts → decode ↑ | planned |
| ngram-mod combined with draft-mtp | structured/code boost | planned |
| Dual drafter (DFlash2-class, deterministic) | 57.4 t/s deterministic decode | **blocked by §2.6 fidelity NO-GO** |
| LZ-engine FA prefill fix | TTFT −25% (pp 413 vs 320–345 t/s) | waiting for ROCmFPX merge — "free win" when landed |

## Phase Gate 2 — MUST pass before declaring the LLM service done

| # | Check | Expected |
|---|---|---|
| G2.1 | `systemctl is-active q38-llama` (or pidfile alive) | active |
| G2.2 | `tr '\0' ' ' < /proc/<pid>/cmdline \| grep -o "engine-overflow\|engine/bin"` | matches the intended engine |
| G2.3 | `curl -s :8000/health` | `{"status":"ok"}` |
| G2.4 | `curl -s :8000/props` | `n_ctx=262144`, `total_slots=2` |
| G2.5 | journal since start: `device_info` shows `ROCm0 : AMD Radeon 8060S Graphics (98304 MiB ...)` and `adding speculative implementation 'draft-mtp'` | both lines present |
| G2.6 | short-ctx bench (see testing-and-baselines.md) | decode within threshold of 32.3 t/s |
| G2.7 | OrderedDict-style structured output, MTP on | byte-exact signature (fidelity) |

**If any check fails: STOP — diagnose (llm pitfalls §2.5), fix, re-run.**
