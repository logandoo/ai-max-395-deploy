# Phase 4 — TTS: IndexTTS-2.5 on TheRock ROCm torch (gfx1151)

Read this file IN FULL before Phase 4. MUST/NEVER/STOP = enforced (SKILL.md §0).

## 4.1 The torch wheel — the deepest pit on gfx1151

- ⛔ **pypi/aliyun `rocm` meta-packages have no 7.13 wheels** (the nightly
  index only carries sdist) → **MUST** rebuild real wheels from the
  official TheRock template
  `build_tools/packaging/python/templates/rocm`
  (patch version, `nonce=""`, `family=gfx1151`, **and add the
  rocm_smi64 LibraryEntry** — torch 7.13 preload requires it).
- ⛔ **pypi torch (rocm7.1 line) has a gfx1151 VGPR bug → SIGSEGV** — never
  use it. Pinned set: `torch 2.9.1+rocm7.13.0a20260501 (cp311)` +
  torchaudio 2.9.0 same-date + triton 3.5.1 same-date
  (venv: `<base>/speech/venv-tts`).
- Do **NOT** set `HSA_OVERRIDE_GFX_VERSION`; set
  `TORCH_ROCM_AOTRITON_ENABLE_EXPERIMENTAL=1`.
- TheRock wheel URLs are **flat**:
  `https://rocm.nightlies.amd.com/v2/gfx1151/<wheel filename>`
  (project-subdirectory paths 404). Filename `+` encoding differs per
  source: AMD index `%2B`, Aliyun literal `+`.
- ⛔ Aliyun pytorch-wheels rejects aria2's default UA with 403 → use
  **wget first** (fall back to aria2 with a browser UA).
- ⛔ **torchaudio's C extension is ABI-drifted vs TheRock torch** (both
  0501 and 0414 dates break) → mask with a pure-torch shim that verbatim-
  extracts the wheel sources (`_hz_to_mel`, `_mel_to_hz`, `fbank`,
  `Resample`, …).
- ⛔ TheRock 0501 cp311 has an internal ABI break (libc10_hip references an
  old MessageLogger signature) → multi-thread crashes → same shim
  workaround.

## 4.2 IndexTTS-2.5 install

- `requires-python <3.12` → `dnf install python3.11`.
- ⛔ **Never pip-install index-tts** (build-deps resolution is unreliable)
  → place the code at `<base>/speech/itsrc` and run the service with
  `Environment=PYTHONPATH=<base>/speech/itsrc` (`itsrc/indextts` is the
  package).
- Auxiliary models: w2v-bert-2.0 → AI-ModelScope mirror OK; ⛔ bigvgan
  (nvidia) is 404 on ModelScope and hf-mirror is unreachable → **relay via
  Mac (R1)**: `config.json` + `generator.pt` into
  `checkpoints/hf_cache/bigvgan/`.
- Service: `tts_server.py` :8202 — `POST /chat/completions` (SSE, MiMo
  wire format) + `/v1/audio/speech`; GPU ROCm0; **inference semaphore = 1**.
- Chatbot attach (current production state): `tts_base_url=http://<host>:8202/v1`,
  `tts_api_key=local`, `tts_model=indextts-2.5`.
- `infer_generator(stream_return=True)` yields per-segment; torch
  tensor/tuple mixtures → normalize via duck-typed `to_mono_f32`.

## 4.3 Warmup and GPU discipline

- ⛔ **Cold start is 30–60s**; after tens of minutes idle the first request
  TTFA balloons to 16–26s (GPU wake/eviction family) → **MUST** warm up
  after every service (re)start; elevated first-utterance RTF is warmup,
  not a defect.
- ⛔ **LLM and TTS must NEVER overlap on the GPU** (R5): measured overlap
  = LLM 5 tok/s + TTS TTFA 3s→26s+ (RTF ~9). The application layer
  **MUST** enforce `tts_after_llm=true` serialization. Slot parallelism
  fixes queueing, not compute contention.

## Phase Gate 4 — MUST pass before declaring TTS done

| # | Check | Expected |
|---|---|---|
| G4.1 | `systemctl is-active speech-tts` | active |
| G4.2 | warmup request after start | first sentence audio within warmup budget (30–60s), then steady-state |
| G4.3 | `POST :8202/v1/audio/speech` (short text) | SSE stream yields ≥1 audio chunk; wav decodes |
| G4.4 | GPU context count (`speech_status.sh`) | GPU procs ≤2 (Server) / ≤3 incl. desktop (Workstation); eviction grep = 0 |
| G4.5 | two sequential TTS requests + one LLM request interleaved | no overlap errors; LLM decode unaffected during TTS (serialized) |

**If any check fails: STOP — diagnose (§4.1/§4.3), fix, re-run.**
