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
  decision (per model architecture x runtime), acquisition, build, service
  registration, verification.
- Measured family recipes: Qwen3.8-Flash-Next (qwen4exp) and Qwen3.8-27B
  (qwen35, ROCmFP4 lineage; historical reference). The Flash-Next family
  documents an **engine lineage** (EngramHalo → Nathanw → Gufo) with the
  measured selection notes that explain each move.
- **Containerized engine deployment (podman)**: rootless storage on the data
  volume, SELinux `label=disable`, GPU passthrough, `loginctl enable-linger`,
  systemd unit pattern, ghcr.nju.edu.cn image mirror — the Gufo stack
  (MIT, gfx1151-native) is the worked example.
- Speculative decoding (MTP draft heads; shared/plain/`-eh` sidecar
  generations are engine-specific), vision (mmproj), and the thinking-mode
  contract verified against the official model card: `enable_thinking`,
  `reasoning_effort`, `preserve_thinking`.
- Operations discipline: systemd lifecycle, download routing (ModelScope
  direct / Mac relay / ghfast / ghcr mirror), sha manifests from the repo
  APIs, SELinux handling, uninstall and rollback with archives.
- Benchmark caliber rules that prevent known false alarms: prefill at
  pp4096 caliber (or nonce-fresh 25K), decode at depth, 200K needle as the
  production-parity acceptance test, read speculative output before
  believing the counter.
- Non-LLM family recipes: ComfyUI vision stack (ERNIE-Image / Krea 2, fp8 on
  gfx1151) and CPU OCR pipelines (MinerU, PaddleOCR); ASR (FunASR,
  SenseVoice) and TTS (IndexTTS-2.5) per the family files.

## Supported models at a glance

| Model | Family file | Validation |
|---|---|---|
| Qwen3.8-Flash-Next (qwen4exp, 125B-A6B, vision, 262k ctx) | `llm-flashnext-flashnext.md` | validated end-to-end (current engine: **Gufo**, podman; engine lineage in the file) |
| Qwen3.8-27B (qwen35 dense) | `llm-qwen38-27b.md` | validated end-to-end (current engine: **Gufo**, UD-Q4_K_XL + DFlash2; host-RAM staging constraint in file; ROCmFP4 lineage = historical appendix) |
| Qwen3-ASR 1.7B + Qwen3-TTS 1.7B (CustomVoice) | `speech-qwen3-gufo.md` | validated (Gufo BF16 safetensors; TTS→ASR round-trip word-exact) |
| Any other GGUF LLM | `model-deploy-generic.md` | procedure-supported, engine decided per arch (S3) |
| ERNIE-Image-Turbo / ERNIE-Image / Krea 2 (ComfyUI) | `vision-ocr-families.md` | validated |
| MinerU 3.4.5 / PaddleOCR 3.7.0 (CPU document parsing) | `vision-ocr-families.md` | validated |
| FunASR paraformer / SenseVoiceSmall / CAM++ / 2pass | `asr-funasr.md` | validated (HISTORICAL — disabled since 2026-09) |
| IndexTTS-2.5 (GPU, TheRock torch) | `tts-indextts.md` | validated (HISTORICAL — disabled since 2026-09) |
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
| `reference/model-deploy-generic.md` | The seven-step loop, engine x runtime decision matrix, container-engine path, Phase Gate 2G |
| `reference/vibeweaver-binding.md` | Harness detection and routing; S-step to vibeweaver mapping |
| `reference/system-base.md` | Distro matrix, ROCm install paths (7.14 core = historical), kernel params, transfer pitfalls, Phase Gate 1 |
| `reference/llm-flashnext-flashnext.md` | Qwen3.8-Flash-Next worked example: engine lineage (EngramHalo→Nathanw→Gufo), Gufo/podman recipe, pitfalls, baselines, thinking contract |
| `reference/llm-qwen38-27b.md` | Qwen3.8-27B on Gufo worked example: UD-Q4_K_XL + DFlash2, host-RAM staging constraint, pitfalls, Phase Gate 27 (ROCmFP4 lineage = HISTORICAL appendix) |
| `reference/speech-qwen3-gufo.md` | Qwen3-ASR + Qwen3-TTS (Gufo) family recipe: variants, API contract, codec pitfall, Phase Gate 3/4 |
| `reference/vision-ocr-families.md` | ComfyUI (ERNIE-Image / Krea 2 / Qwen-Image-2.1) and MinerU / PaddleOCR CPU pipelines |
| `reference/load-unload-panel.md` | model-panel :8300 — the load/unload surface: registry, API, VRAM guard, load-window discipline |
| `reference/gpu-discipline.md` | #6012 eviction firewall, GPU process budget, failure playbook |
| `reference/testing-and-baselines.md` | Test levels, benchmark caliber rule, baselines and thresholds, Release Gate |
| `reference/asr-funasr.md`, `reference/tts-indextts.md` | ASR / TTS family recipes |

## Model selection & measured performance

| Model | Choice | Engine | Prefill | Decode (MTP) | Max context |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next | UD-Q4_K_XL (111.3GB) | **Gufo** (MIT, podman, gfx1151-native) | **~1436 t/s** (fresh 25K); 1317 t/s avg on a 219K-token request | 25-45 t/s (measured band) | 262,144 |
| Qwen3.8-27B | UD-Q4_K_XL (16.35GiB) + DFlash2 Q4_K_M | **Gufo** (MIT, podman, gfx1151-native) | **314.9 t/s pp_eff** (219K needle, measured) · official Q4 656 t/s pp (C1) | 70.56 t/s tg (DFlash2, official) | 262,144 |
| Qwen3.8-27B (HISTORICAL) | ROCmFP4-FAST (14.5GB) | q38rocm ROCmFPX (stock/ovf) + ROCm 7.2.3 | ~365 t/s (1k-prompt caliber) | 32.3 t/s (n4) | 262,144 |

Each family file is organized as an **engine lineage**: for one model it
lists the validated engines (current recommendation, zero-build fallback,
historical reference) with their measured profiles and when to pick each,
and every point on the line keeps an executable restore path. All numbers
are *benchmark-condition* readings (fixed prompts, filler/echo content,
high MTP acceptance): real workloads such as novel prose read lower and
that is expected — three-factor attribution in
`reference/testing-and-baselines.md` §6.2a.

## Validation status

- Fedora 44 Server: validated end to end on Strix Halo hardware. Every
  command in this package ran there.
- Ubuntu and Arch: procedure-projected, not hardware-validated. The
  substitution matrix lives in `reference/system-base.md` §1.0. Verify each
  substituted step at deploy time and record deltas back into that file.
- Boundaries: LLM engines = llama.cpp-family forks plus Gufo (no vLLM/SGLang
  path); YaRN beyond native context excluded (retrieval miss, measured); NPU
  paths are out of scope; vision/OCR/ASR/TTS families follow their own
  runtimes as filed above.

## License

MIT — see [LICENSE](LICENSE). Upstream engines keep their own licenses
(Apache-2.0 for llama.cpp, MIT for Gufo, Qwen Community License for model
weights).
