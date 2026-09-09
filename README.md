# ai-max-395-deploy

Deploy any user-specified model on AMD Ryzen AI Max+ 395 "Strix Halo"
hardware.

English | [简体中文](README_zh.md)

## Features

Written for coding agents (opencode, DeepSeek harness, standalone) and for
operators of Strix Halo machines. Every rule and number is validated on
Strix Halo hardware (Fedora 44 Server, 128GB unified memory with a
BIOS-carved 96GB GPU pool); pitfall entries state measured failure modes.

- Seven-step deployment loop for any GGUF model: survey, memory math, engine
  decision (per model architecture x ROCm version), acquisition, build,
  service registration, verification.
- Measured family recipes: Qwen3.8-Flash-Next (qwen4exp) and Qwen3.8-27B
  (qwen35, ROCmFP4 lineage).
- Speculative decoding (MTP draft heads plus ngram-mod), vision (mmproj), and
  the thinking-mode contract verified against the official model card:
  `enable_thinking`, `reasoning_effort`, `preserve_thinking`.
- Operations discipline: systemd lifecycle, download routing (ModelScope
  direct / Mac relay / ghfast), SELinux handling, uninstall and rollback
  with archives.
- Benchmark caliber rules that prevent known false alarms: prefill at
  pp4096 caliber only, decode at depth, read speculative output before
  believing the counter.
- Non-LLM family recipes: ComfyUI vision stack (ERNIE-Image / Krea 2, fp8 on
  gfx1151) and CPU OCR pipelines (MinerU, PaddleOCR); ASR (FunASR,
  SenseVoice) and TTS (IndexTTS-2.5) per the family files.

## Supported models at a glance

| Model | Family file | Validation |
|---|---|---|
| Qwen3.8-Flash-Next (qwen4exp, 125B-A6B, vision, 262k ctx) | `llm-flashnext-flashnext.md` | validated end-to-end |
| Qwen3.8-27B (qwen35, ROCmFP4 lineage) | `llm-qwen38-27b.md` | validated (engine swap-proof passed) |
| Any other GGUF LLM | `model-deploy-generic.md` | procedure-supported, engine decided per arch (S3) |
| ERNIE-Image-Turbo / ERNIE-Image / Krea 2 (ComfyUI) | `vision-ocr-families.md` | validated |
| MinerU 3.4.5 / PaddleOCR 3.7.0 (CPU document parsing) | `vision-ocr-families.md` | validated |
| FunASR paraformer / SenseVoiceSmall / CAM++ / 2pass | `asr-funasr.md` | validated |
| IndexTTS-2.5 (GPU, TheRock torch) | `tts-indextts.md` | validated |
| qwen3.5-0.8B on NPU (FLM/lemonade) | `llm-qwen38-27b.md` §2.4 | not a supported path |

## Install

The package is a directory of markdown files; no build step.

- opencode: add this directory to `skills.paths` in `opencode.json`, or copy
  the folder to `~/.config/opencode/skills/ai-max-395-deploy/`.
- DeepSeek harness: register through its plugin/skill registry so that
  `vibeweaver-dsh` (if present) can bind to it.
- Standalone agents: read the files directly; no registration required.

## Usage

For an agent, in order:

1. Read `SKILL.md`. It declares hardware facts, rules R1-R15, and the phase
   map into the reference files.
2. Resolve the agent-stack binding (SKILL.md §0.1): vibeweaver present
   means its workflow governs and this package provides domain content;
   absent means the package loop and Phase Gates govern standalone.
3. Run the seven-step loop from `reference/model-deploy-generic.md`:
   S1 survey, S2 memory math, S3 engine decision, S4 acquisition, S5 build,
   S6 service, S7 verify, S8 uninstall or replace.

## File structure

| File | Content |
|---|---|
| `SKILL.md` | Router: hardware model, rules R1-R15, phase map, pitfall index |
| `reference/model-deploy-generic.md` | The seven-step loop, engine x ROCm decision matrix, Phase Gate 2G |
| `reference/vibeweaver-binding.md` | Harness detection and routing; S-step to vibeweaver mapping |
| `reference/system-base.md` | Distro matrix, ROCm install paths, kernel params, transfer pitfalls, Phase Gate 1 |
| `reference/llm-flashnext-flashnext.md` | Qwen3.8-Flash-Next worked example: memory ledger, dead-end ledger, baselines, thinking contract |
| `reference/llm-qwen38-27b.md` | Qwen3.8-27B / ROCmFP4 lineage (qwen35 family) |
| `reference/gpu-discipline.md` | #6012 eviction firewall, GPU process budget, failure playbook |
| `reference/testing-and-baselines.md` | Test levels, benchmark caliber rule, baselines and thresholds, Release Gate |
| `reference/asr-funasr.md`, `reference/tts-indextts.md` | ASR / TTS family recipes |

## Model selection & measured performance

| Model | Choice | Engine | Prefill (pp4096) | Decode (short context) | Max context |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next | UD-IQ4_XS (93.7GB) | EngramHalo.cpp `strix-halo-qwen4exp` + ROCm 7.14 core | 477.3 t/s | 28.7-42.1 t/s with MTP | 262,144 with MTP |
| Qwen3.8-27B | ROCmFP4-FAST (14.5GB) | q38rocm ROCmFPX (stock/ovf) + ROCm 7.2.3 | ~365 t/s (1k-prompt caliber) | 32.3 t/s with MTP n4 | 262,144 |

Both run on one Strix Halo box in mutually exclusive resident slots. The
Decode column is a *benchmark condition* (short context, `n_predict=256`,
ngram-friendly content, MTP acceptance 79-93%): prose sits toward the low end,
code/rewrite toward the high end. Production long-form reads lower and that is
expected — novel prose with 2k+ token outputs at 25-62k effective context
reads **14.8-18 t/s**; ngram-hit content (echo/code/copy) 26-49 t/s. Three-
factor attribution in `reference/testing-and-baselines.md` §6.2a.

## Validation status

- Fedora 44 Server: validated end to end on Strix Halo hardware. Every
  command in this package ran there.
- Ubuntu and Arch: procedure-projected, not hardware-validated. The
  substitution matrix lives in `reference/system-base.md` §1.0. Verify each
  substituted step at deploy time and record deltas back into that file.
- Boundaries: llama.cpp only for LLMs (no vLLM/SGLang path); YaRN beyond
  native context excluded (retrieval miss, measured); NPU paths are out of
  scope; vision/OCR/ASR/TTS families follow their own runtimes as filed
  above.

## License

MIT — see [LICENSE](LICENSE). Upstream engines keep their own licenses
(Apache-2.0 for llama.cpp, Qwen Community License for model weights).
