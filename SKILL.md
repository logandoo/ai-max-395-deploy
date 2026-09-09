---
name: amd-ai-max-395-model-deployment
description: >
  Deployment skill for AMD AI Max+ 395 "Strix Halo" (gfx1151 iGPU, 128GB
  unified memory, BIOS 96GB GPU-pool carve, OS sees ~30GB) on Fedora 44.
  TWO layers: (A) a MODEL-AGNOSTIC seven-step deployment procedure (survey →
  memory math → engine decision per model-arch × ROCm version → download
  routing → build → service → verification → restore) that works for
  ANY user-specified model; (B) per-family recipes — Qwen3.8-Flash-Next
  (qwen4exp, EngramHalo + ROCm 7.14 core, resident) and Qwen3.8-27B
  (qwen35, ROCmFP4 lineage). Covers llama.cpp fork selection, MTP/speculative
  decoding, giant lookup tensors, ModelScope/Mac-relay download routing,
  ROCm multi-version coexistence, systemd+SELinux+service registration,
  GPU-context discipline, benchmark caliber discipline, offload semantics,
  and a pitfall compendium where every entry states a measured failure mode.
  TRIGGER when the user asks to deploy/replace/offload/restore ANY model or
  inference service on an AI Max 395 / Strix Halo machine, debug deployment
  failures (load hang, DMA spin, SIGSEGV, 203/EXEC, eviction, MTP collapse,
  OOM during load, wheel ABI, Vulkan crash), pick a quant/engine, or ask
  about deployment options and performance baselines on this hardware.
---

# AMD AI MAX 395 Model Deployment

All commands, rules and numbers in this package are derived from a validated
device: **** (Fedora 44 Server, 2026-09). Pitfall entries state
measured failure modes. Workspace evidence index: §7.

## 0. How to use this skill — READ FIRST

- **Target hardware:** AMD Ryzen AI Max+ 395 (16C/32T Zen5), gfx1151 iGPU
  (Radeon 8060S, PCI f4:00.0) + aie2p NPU (`/dev/accel/accel0`), 128GB
  LPDDR5X unified memory, **BIOS-carved 96GB GPU pool** (OS sees ~30GB).
  This topology is THE constraint that shapes every decision
  (model-deploy-generic.md S2).
- **OS: Fedora 44 Server = the VALIDATED baseline.** **Ubuntu / Arch: a
  portability layer is provided** (system-base.md §1.0 matrix — ROCm install,
  kernel-param tool, security module, firewall substitutions) but is
  **procedure-projected, NOT hardware-validated**; verify each substituted
  step at deploy time and record deltas back into that file. Kernel-param
  (R7) and security-module (R11) rules are distro-parameterized there.
- **Keyword convention (RFC-2119, enforced):** **MUST** / **MUST NOT** /
  **NEVER** / **STOP** mark rules whose violation breaks the deployment or
  the machine. They are not style.
- **Progressive disclosure (mandatory):** this file is the router. Before
  executing any phase, **read that phase's reference file IN FULL** (§4).
  Each reference file ends with a **Phase Gate** — executable checks with
  expected outputs. **If any gate check fails: STOP, fix, re-run.**
- **Deploying a model = the seven-step loop** (model-deploy-generic.md §0):
  S1 survey → S2 memory math → S3 engine decision → S4 acquisition →
  S5 build → S6 service → S7 verify (→ S8 offload/replace →
  S9 restore). Per-family files are worked examples, not substitutes for
  the loop.
- **Mandatory sequence:** Phase 0 → Phase 1 → Phase 2 loop (any model; all
  of S1-S7 require Phase 1 green) → Phase 5 discipline applies continuously →
  Phase 6 Release Gate before declaring done. Phase 3/4 (ASR/TTS) only when
  the user asks for those families.
- **Harness binding (MUST resolve at task start, §0.1):** if a vibeweaver
  harness is present, its workflow governs the deployment (this skill's
  loops are vibeweaver-shaped and map 1:1); if absent, the skill's own loop
  + Phase Gates govern standalone. Details: `reference/vibeweaver-binding.md`.

## 0.1 Harness binding — detect FIRST, one minute

1. **Detect the harness:**
   - **opencode:** the session's available-skills list contains `vibeweaver`
     (authoritative), or `~/.config/opencode/skills/vibeweaver/SKILL.md` /
     `.opencode/skills/vibeweaver/SKILL.md` exists on disk.
   - **DeepSeek harness:** the `vibeweaver-dsh` plugin is installed/registered.
   - Otherwise → **standalone mode** (this skill alone).
2. **Route:**
   - vibeweaver present → **load it and run the deployment under its
     binding contract** (acceptance-first with `> cap=5 stall=3×`,
     verification loop with `diagnosis:` on every FAIL, evidence gates,
     ADRs for AUTO decisions, PAUSED packets at hard stops, independent
     review for major changes, Memory Gate). This skill then supplies the
     DOMAIN content: the S1-S8 procedure, the decision matrix, device facts
     and pitfalls. JS-plugin mechanics (write-gate hooks) are harness-
     internal — irrelevant to the procedure. Mapping table:
     `reference/vibeweaver-binding.md`.
   - standalone → the skill's own loop governs: S1-S8 in order, Phase Gate
     checks after each phase, acceptance criteria + verification log +
     ADR/evidence artifacts exactly as specified in the reference files.
     **The artifacts are identical in both modes** — the reference files
     are vibeweaver-shaped on purpose, so a standalone agent produces the
     same audit trail without knowing vibeweaver exists.
3. Record the detection result (harness + evidence path) in the survey
   artifact (S1) — one line, e.g. `harness: opencode+vibeweaver (skill dir
   ~/.config/opencode/skills/vibeweaver)`.

## 1. Hardware model (verify with S1 at every task start)

| Component | Port | systemd unit |
|---|---|---|
| **LLM llama-server — Qwen3.8-Flash-Next (UD-IQ4_XS + MTP, ctx 262144)** | 8000 | `flashnext` |
| ComfyUI (ERNIE-Image / Krea 2) | 8188 | `comfyui` |
| speech ASR / TTS | 8201 / 8202 | `speech-asr` / `speech-tts` |
| FunASR 2pass runtime | 10095 | `funasr-runtime` |
| MinerU / PaddleOCR (CPU) | 8301 / 8302 | `mineru-api` / `paddleocr-serve` |
| lemond (NPU qwen3.5-0.8B) | 13305 | `lemond` |

The validated deployment keeps exactly one LLM onload at a time; every
other stack is offloaded (inactive, weights on disk, startable on demand).
q38-llama and flashnext are mutually exclusive on :8000.

Two ROCm runtimes coexist **by design**: `/opt/rocm` (7.2.3) +
`/opt/rocm/core-7.14` (qwen4exp fast path). Each engine builds against the
prefix its family requires — never remove either.

**Offload/onload terminology (binding for every deployment):** "offload" a
model = stop its service and free VRAM; weights stay on disk and the model
stays startable on demand. "onload" = the resident set. Deleting weights
is a separate, explicitly-confirmed destructive action and is not part of
any deploy/offload instruction.

## 2. Non-negotiable rules (enforced by Phase Gates)

| # | Rule | Consequence of violation |
|---|---|---|
| R1 | **MUST** download from CN-reachable sources directly on the machine; **MUST** relay HF/GitHub/repo.amd.com via the Mac or `ghfast.top` device-direct; verify sizes against the repo manifest | device has no reliable direct HF route |
| R2 | **MUST** pin engine + model by sha256/bytes and re-verify on device; sha-failed artifacts are deleted and re-downloaded fresh | silent corruption = undiagnosable |
| R3 | **MUST** take a read-only device survey before any change | no baseline = no attribution |
| R4 | Long-running services **MUST** be systemd units; **NEVER** `nohup` from ssh; long downloads are stopped by PID, never by pattern-kill | ssh-scoped cgroup reaps the service; pattern-kill cannot distinguish targets |
| R5 | GPU-process budget: decide **coexist vs replace** before loading a new model; ≤3 GPU procs total; **LLM and TTS must NEVER overlap** | ROCm #6012 eviction cascade; measured overlap collapse |
| R6 | **MUST** match engine × ROCm version to the model ARCHITECTURE (generic §S3 matrix) — a fast engine for family A hangs family B | qwen4exp on 7.2.3: DMA hang; on Vulkan: 6.9 t/s |
| R7 | Kernel-param changes **MUST** go through the distro tool (Fedora `grubby --update-kernel=ALL` / Ubuntu `update-grub` / Arch `grub-mkconfig`) and **MUST** be verified via `/proc/cmdline` | mkconfig-class tools drop `root=`/`iommu=pt` (boot failure) |
| R8 | Every claim **MUST** have an on-disk artifact | audit discipline |
| R9 | Credentials stay in `.env.local` (gitignored) | secret hygiene |
| R10 | A rollback path **MUST** exist before any mutation (archive + re-download URLs) | recovery without one is guesswork |
| R11 | Every new executable dir under `/data` **MUST** satisfy the distro security module before systemd start — Fedora/SELinux: `chcon -R -t bin_t`; Ubuntu/Arch: no SELinux step, diagnose Permission denied from journalctl instead | SELinux exec denial = 203/EXEC |
| R12 | **The deployment config file is the single source of truth** for service args — unit rendered from template + preflight; no drift between validated config and deployed unit | config drift produces unevidenced behavior |
| R13 | Benchmarks at **standard caliber** (pp≥4096; decode at d0 AND depth; read the output) | small-prompt prefill measures fixed overhead only; speculative decoding can fake speed with garbage |
| R14 | Lifecycle code **MUST** accept `inactive\|failed` as terminal-stopped + units **MUST** carry VRAM-drain ExecStartPre + `KillSignal=SIGINT` | ROCm teardown SEGV → unit "failed"; restart raced reclaim → crash |
| R15 | Destructive scripts **MUST** be gated (`UNINSTALL=YES`) + idempotent + archive-before-delete; weight deletion is never part of offload semantics | deletion is irreversible |

## 3. Directory conventions (device layout)

| Purpose | Path |
|---|---|
| Payload base | `/data` (xfs on LVM `fedora/lvol0`, ~448GB) — root LV is 15G, keep it clean |
| Resident LLM (flashnext) | `/data/flashnext/{models,build-eh-out,src,logs,rollback}` + `/opt/rocm/core-7.14 → /data/rocm714/...` |
| Second LLM (q38rocm) | `/data/q38rocm` (27B + 35B weights, engine archive, drafters, mmproj) |
| ComfyUI | `/data/comfyui/{app,venv,models}` |
| Speech | `/data/speech/{app,itsrc,models,checkpoints,venv-*}` |
| OCR | `/data/ocr/{venv-*,cache,remote}` |

## 4. Progressive-disclosure map — read BEFORE each phase

| Phase | Read (in full) | Covers |
|---|---|---|
| 0+1 | `reference/system-base.md` | prerequisites, download strategy, BIOS/kernel/storage, ROCm (7.2.3 base + 7.14 core coexistence), tuning, SSH pitfalls, Phase Gate 1 |
| 0.1 | `reference/vibeweaver-binding.md` | harness detection (opencode vibeweaver / deepseek vibeweaver-dsh / standalone), routing rule, S-step ↔ vibeweaver mapping |
| 2 (ANY model) | `reference/model-deploy-generic.md` | **the seven-step loop**: memory math, engine×ROCm decision matrix, acquisition routing, build, service, standard-caliber verify, offload/replace, restore — Phase Gate 2G |
| 2 (family) | `reference/llm-flashnext-flashnext.md` | Qwen3.8-Flash-Next worked example: memory ledger, dead-end ledger, baselines, thinking contract |
| 2 (family) | `reference/llm-qwen38-27b.md` | Qwen3.8-27B / ROCmFP4 lineage recipe (qwen35 arch, ROCmFPX engine) |
| 2b (family) | `reference/vision-ocr-families.md` | ComfyUI (ERNIE-Image / Krea 2, fp8 on gfx1151) and MinerU / PaddleOCR CPU pipelines |
| 5 | `reference/gpu-discipline.md` | #6012 eviction firewall, GPU process budget, failure playbook |
| 6 | `reference/testing-and-baselines.md` | testing standard, calibers, known-good baselines, Release Gate |
| 3 (ASR family) | `reference/asr-funasr.md` | funasr split pipeline, SenseVoice, 2pass runtime, Phase Gate 3 |
| 4 (TTS family) | `reference/tts-indextts.md` | TheRock gfx1151 wheel rebuild, IndexTTS-2.5, Phase Gate 4 |

## 5. Cross-cutting pitfalls (one-line index; details in phase files)

- quoted ssh heredoc; zsh `echo ===X===` needs quotes → system-base
- `sudo -S tee` swallows pipe → /tmp then `sudo cp` → system-base
- ssh-inline `$VAR` expansion → scp script files (never inline nohup) → system-base
- zsh does NOT word-split `$var` (`set -- $var` yields ONE arg) → split explicitly → system-base
- scp `-q` to a directory target can fail silently → explicit filename + `ls -la` → system-base
- SELinux 203/EXEC on new /data bin dirs → chcon -R bin_t (every build dir) → generic S5
- root LV 15G: multi-GB dnf installs exhaust it → rpm2cpio/deb-extract to /data + symlink → generic S5
- pip-gguf writer API ≠ repo gguf-py API → introspect before scripting → generic S3
- `GGML_CUDA_ENABLE_UNIFIED_MEMORY` = qwen4exp garbage-output poison → generic S3
- MTP sidecar tensor names drift between converter releases → rename, don't rebuild → generic S3
- teardown SEGV → unit "failed" is terminal-stopped; accept + drain VRAM before restart → generic S6
- benchmark caliber: pp≥4096 or nothing; read spec-decode output → generic S7
- hf-mirror `-C -` resume can corrupt → sha256 mandatory after every download; sha-fail = fresh re-download → generic S9
- downloader scripts can be incomplete vs what the service loads → audit script ↔ service model list → generic S9
- dangling symlink + `makedirs` = FileExistsError → rm/mkdir/re-ln → generic S9
- restore = exact filenames (config.json not bigvgan_config.json) + per-stack availability smoke → generic S9
- IndexTTS anti_alias JIT build failure is tolerable (native fallback); fatal = missing bigvgan files → generic S9
- two llama residents on :8000 are mutually exclusive → swap-proof pattern → generic S9
- `--reasoning off`-style flags belong to specific model families — carry flags per family, never between them → family files
- binary name collision (multiple `llama-server`) → identify via `/proc/<pid>/cmdline` → family files
- ModelScope delistings → verify URLs before writing download scripts → system-base

## 6. Verification & release

Deployment is **NOT done** until: Phase Gate 2G (generic) + the family file's
numbers reproduced within band + the Release Gate in
`reference/testing-and-baselines.md` passes — all with on-disk evidence.

## 7. Provenance

This package is the distilled procedure. Raw deployment evidence (iteration
logs, decision records, benchmark transcripts, screenshots) is retained in
the maintainer workspace and is not published.
