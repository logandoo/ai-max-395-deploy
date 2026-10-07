# Family recipes — Vision (ComfyUI) and OCR (MinerU / PaddleOCR)

Both stacks run offloaded on the reference deployment: services inactive,
weights on disk, startable on demand. These are the re-deploy recipes and the
pitfalls that cost real time. Derived from the deployment workspace's stack
notes.

## V — Vision family (ComfyUI): ERNIE-Image-Turbo / ERNIE-Image / Krea 2 / Qwen-Image-2.1

Deployed: ComfyUI **0.37.4** (upgraded from 0.34.0 — CU_TAG pinned;
Qwen-Image-2.1 templates require ≥0.37.0) :8188, systemd `comfyui.service`
(on-demand), weights via **ModelScope device-direct** (Comfy-Org mirrors),
venv python3.11 at `/data/comfyui/venv`.

Validation: all three original models generated (Turbo 8-step 8.3s @1024²
hot; Krea 2 Turbo fp8_scaled 68.3s cold incl. load); fp8_scaled works on
gfx1151 (fp8 storage + bf16 compute fallback, no NaN); sha256 8/8 at the
0.34.0 delivery + 15/15 for the Qwen-Image-2.1 int8 suite
(t2i 1024² 82.9s cold, native 2048² 1911.9s — 25-step euler/simple/cfg1.0).

Non-negotiables (measured):
- Wheel pairing: torch / torchvision / triton / torchaudio **from the same
  TheRock date index** (torch 2.9.1+rocm7.13.0a20260501 family). Mixing
  dates → `operator torchvision::nms does not exist` (ABI mismatch).
- `--gpu-only` REQUIRED (weights straight into the GTT pool; default offload
  device copies to RAM twice).
- ⛔ `--disable-mmap` FORBIDDEN on this box (30GB OS RAM): safetensors loads
  whole-file → oom-kill.
- Host-RAM hard edge: loading 16GB UNETs materializes tensors despite mmap.
  ⛔ The old 32GB `/data/swapfile` was found **orphaned**
  (never in `/proc/swaps` — live swap is zram only) and was removed; do NOT
  assume disk swap protects you. If ComfyUI loads OOM on host RAM, re-create a
  swapfile **and verify activation** (`swapon --show` must list it).
- ⛔ **VRAM discipline (host-level OOM, measured):** with
  flashnext resident (**~91GB of the 96GB pool with the current Gufo
  stack** — ~5GB headroom) the Qwen-Image-2.1 suite (17.3GB start; 2K
  output or PE enhancement +9.5GB) cannot start at all without crossing the
  edge → host-RAM oom-killer can wedge the whole box.
  **Offload flashnext first (panel, ~30s), generate, then reload.** The
  panel's VRAM guard (409 VRAM_INSUFFICIENT, `force:true` escape hatch)
  enforces the declared budgets — see `load-unload-panel.md`.
- SELinux: `chcon -t bin_t` on venv python AND the launcher script.
- Logs via journal; ⛔ `StandardOutput=append:` fails on /data (209/STDOUT).
- GPU budget: ComfyUI running = 3 GPU processes (the #6012 ceiling with
  llama + TTS); never a 4th.
- Model search path: `extra_model_paths.yaml` → `/data/comfyui/models`.

Rollback: `script/linux/comfyui_rollback.sh` (keeps model archive);
re-deploy ≈ 4 min reusing local wheels/weights.

## O — OCR family (MinerU 3.4.5 + PaddleOCR 3.7.0)

CPU pipelines (no GPU budget consumed): `mineru-api.service` :8301,
`paddleocr-serve.service` :8302; venvs at `/data/ocr/venv-{mineru,paddle}`;
weights via ModelScope (`PDF-Extract-Kit-1.0`, PP-OCRv6) into
`/data/ocr/cache` with symlinks from the default home-cache paths.

Non-negotiables (measured):
- Paddlex serving extra is `paddlex[serving]` — NOT `[serve]`; PaddleOCR
  base does not include it.
- SELinux: the ENTIRE venv `bin/` directory needs `chcon -R -t bin_t`
  (entry console-scripts AND the interpreter; labeling python alone fails).
- ⛔ 15G root LV: pip/model caches under `/home` filled it (Errno 28) —
  redirect all caches to `/data/*/cache` + `ln -sfn` back.
- Headless Fedora: opencv needs `dnf install mesa-libGL`.
- MinerU needs explicit `six`; model source `MINERU_MODEL_SOURCE=modelscope`.
- PaddleOCR serving protocol is JSON InferRequest
  (`{"file":"<base64>","fileType":1}`), not multipart.

Rollback: `script/linux/ocr_rollback.sh` removes the OCR stack; ASR/TTS
restore separately via `speech_deploy.sh models+units+start`.

## Gate (family re-deploy)

| # | Check | Expected |
|---|---|---|
| GV.1 | service active + health endpoint | per the deployment config |
| GV.2 | one real generation / parse with output READ | correct, no NaN/garbage |
| GV.3 | GPU proc count within budget (#6012) | ≤ 3 |
| GV.4 | SELinux labels applied (Fedora) | no 203/EXEC in journal |
| GV.5 | caches on /data, root LV < 90% | pass |
