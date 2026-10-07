# Phase 2 (ops) — model-panel :8300 — the machine's load/unload surface

Every model deployed by this skill MUST be registered here (S6 step 4 /
Phase Gate 2G.7): a deployed-but-unregistered model is not operable by the
user. Read this file before touching the panel, before adding a registry
entry, and before debugging "the model loaded but calls fail".

## P1. What it is

- **model-panel.service** :8300 — FastAPI panel (`/data/model-panel`, venv
  python3.11 + fastapi/uvicorn/httpx), user `<USER>`, enabled at boot. It is
  the ONLY sanctioned load/unload mechanism: it runs the root-owned whitelist
  helper `/usr/local/sbin/model-panel-act` (sudoers: NOPASSWD on that path
  only) and enforces the GPU budget + VRAM guard before every start.
- **Registry = the deployment config** (`/data/model-panel/config.toml`,
  rendered from the workspace `config.toml` by `script/linux/panel_deploy.sh`;
  the config file is the single source of truth, R12): `[[panel.models]]`
  (id/unit/accel/port/kind/vram_gb/health_path/model_dir/model_glob) and
  `[[panel.access]]` (declarative UI/API access card per model).
- **API** (authoritative reference: workspace `docs/PANEL_API.md`):
  `GET /api/state` (registry + live unit/health + GPU/RAM gauges + action
  log ring 50) · `POST /api/models/{id}/load` · `POST /api/models/{id}/unload`
  (`mode=service|vram`) · `GET /api/health`.
- Lifecycle: `script/linux/panel_{deploy,start,stop,status,test,rollback}.sh`.

## P2. Load-window 503 discipline (read before any client integration)

**A 111GB GGUF (UD-Q4_K_XL) takes ~40-60s to load** (Gufo container start +
mmap table; measured 40.6s engine `load_completed` on the reference box).
During that window the unit is starting and clients get connection errors or
503 — the panel's load call BLOCKS until its health probe goes green (or
returns `200 + HEALTH_PENDING` at timeout). **Trust `health: healthy` in
`GET /api/state`, then call the model.**

- Same window applies after every unload→load cycle (VRAM `/free`, swap,
  crash restart). Budget ≥3 min for health; the baseline threshold is
  ≤3 min (testing-and-baselines.md §6.2a).
- Gufo answers `/health` (`{"status":"ok"}`) — the panel probe is compatible
  with both llama-server and Gufo backends.

## P3. Split-GGUF entry semantics (flashnext)

Qwen3.8-Flash-Next ships as **N shard files = ONE model** (current: UD-Q4_K_XL
= 4 shards `…-00001-of-00004.gguf`; the former UD-IQ4_XS fallback was removed). Qwen3.8-27B (q38-gufo) is a SINGLE file
(`Qwen3.8-27B-UD-Q4_K_XL.gguf`) beside two sidecars (mmproj / DFlash2) — its
`model_glob = "Qwen3.8-27B-UD-*.gguf"` selects the target and excludes the
sidecars. Only the entry shard is loadable (`-m` points at it; the loader pulls the
other shards implicitly). Therefore:

- the registry entry filters `model_glob` to the entry shard
  (`*-00001-of-*.gguf`) — `detail.disk_models` lists **selectable loadable
  entries**, not raw files; a split set appears as ONE row;
- never "swap" between shards — there is nothing to swap. The swap verb
  (`load` with `model_file`) exists for directories that hold ≥2 real models.

## P4. VRAM guard (load-time reservation mechanism)

Loading a `accel=gpu` model with a declared `vram_gb` is rejected with
`409 VRAM_INSUFFICIENT` when `need > total − max(used_now, Σ other active
residents' vram_gb)`. Rationale: lazy services (ComfyUI=22GB) reserve their
budget at load time — "idle at load, wall at generate" is the host-OOM
failure mode this guard exists to prevent. `{"force": true}` bypasses with an
audit note; the GPU-process budget (#6012, ≤3) always applies. The active
target is self-excluded (idempotent reload / swap = net-zero).

## P5. Helper model-swap dispatch

`sudo /usr/local/sbin/model-panel-act set-llama-model <file.gguf>` rewrites
the `-m …gguf` line of the owning unit and `daemon-reload`s. Dispatch is by
the file's directory:

| directory | unit rewritten |
|---|---|
| `/data/flashnext/models/UD-Q4_K_XL` (Gufo unit — `-m` in the unit is the **container path** `/models/UD-Q4_K_XL/…`; the helper's sed targets that) | `flashnext.service` |
| `/data/q38gufo/models` (Gufo unit — container path `/models/…`; Qwen3.8-27B UD 量化族) | `q38-gufo.service` |

Filename whitelist: `^[A-Za-z0-9._-]+\.gguf$`; missing file → exit 2; each
rewrite keeps a `.bak-panel` backup of the unit. Any new llama family that
needs swapping adds ONE directory+unit row here (and in the helper) — do not
generalize the helper into a sed shell.

## P6. Offload/onload vocabulary + operational pitfalls

- "unload" = stop the service (VRAM freed; weights stay on disk). ComfyUI
  also has `mode=vram` (`POST /free`, process kept, lazy reload next
  generate). "load" = start (+ optional `model_file` swap). Deleting weights
  is NEVER part of unload (SKILL.md §1 terminology).
- After a deliberate stop the unit may rest in `failed` (ROCm teardown SEGV)
  — terminal-stopped, not an incident (gpu-discipline.md §5.2).
- Restart races the async ~74GB BO reclaim → the unit's ExecStartPre waits
  for VRAM < 8GB before start; ad-hoc restarts must do the same.
- The action log ring (50) is the panel-side audit trail; long-term audit
  goes through `journalctl -u model-panel`.
- **Containerized units (Gufo)**: stop is clean — the unit ends `inactive`
  (no ROCm teardown SEGV); the helper/panel accept `inactive|failed` either
  way. `active_model` attribution still works: the podman main process argv
  carries `-m /models/…` and `ops.llama_active_model` reads it from
  `/proc/<MainPID>/cmdline`. VRAM-drain ExecStartPre still applies (a restart
  races the async ~90GB BO reclaim exactly as with native engines).
- CSRF: mutating calls with a cross-site `Origin` header get 403
  (`CSRF_ORIGIN_MISMATCH`); curl without Origin is unaffected.

## Phase Gate 2G-P — MUST pass after registering a model

| # | Check | Expected |
|---|---|---|
| GP.1 | `GET /api/state` registry | new id present with correct unit/port/kind |
| GP.2 | `POST /api/models/<id>/load` | 200 + `health: healthy` (allow the load window) |
| GP.3 | `detail.disk_models` (llama) | lists loadable entries only (split sets = 1 row) |
| GP.4 | VRAM guard interaction | declared `vram_gb` correct; guard math matches measured resident |
| GP.5 | unload → load round-trip | both green; `failed` terminal state accepted |

**If any check fails: STOP — fix the registry/config, not the caller.**
