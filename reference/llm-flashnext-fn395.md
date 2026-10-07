# Phase 2 (family) — Qwen3.8-Flash-Next (qwen4exp) — CURRENT RESIDENT on the machine

Worked example of model-deploy-generic.md on this exact box. Read the generic
file first. This file is organized as an **engine lineage**: the same model
lists the validated engines side by side (current recommendation → zero-build
fallback → legacy datapoint), with the measured numbers and the pitfalls that
tell you when each applies.

## F1. Current stack — model & memory ledger (S2 worked example)

- Model: unsloth/Qwen3.8-Flash-Next-GGUF **UD-Q4_K_XL** — 4 shards, 111.3GB
  total; entry shard `…-00001-of-00004.gguf`; sha256 4/4 cross-checked against
  the ModelScope file API. ⛔ The current engine supports ONLY this quant on
  this model — see F4-1.
- Alternate route (Nathanw engine): **UD-IQ4_XS** (93.66GB; backbone 39.3GB +
  PLE/engram single tensor 28.8GB IQ4_NL) + the original MTP
  `mtp-…-Q8_0.gguf`. Keep fallback weights + sidecars on disk as the rollback
  layer.
- Sidecars (Gufo): **shared** MTP `MTP/mtp-Qwen3.8-Flash-Next-shared-Q8_0.gguf`
  2.79GB (sha256 `5ff54097…` = HF LFS oid); vision `mmproj-BF16.gguf` 0.91GB
  (Gufo wants BF16 — `mmproj-F16.gguf` belongs to the Nathanw route).
- Topology: 96GB GPU pool + ~30GB host RAM. Resident GPU 90.8/98.3GB
  (`amd-smi`); small host RSS — the engine memory-maps the 51B-param PLE table
  on the host and gathers rows per token (the same idea EngramHalo pioneered
  historically, now inside the engine proper).
- ⛔ Two full LLM engines cannot coexist on this carve: stop the resident
  before starting any A/B engine (measured; applies across all three engines).

## F2. Engine lineage (all three run on this box — pick by need)

| Engine | Role | fresh-25K prefill | 200K-token request (219,406 tok) | Notes |
|---|---|---|---|---|
| **Gufo** (`gufo-org/gufo`, MIT, container) | **Recommended** | **~1436 t/s** | **166.6s total / pp≈1317 t/s / needle 3/3** | container-native; recipe F3 |
| Nathanw v0.7.6.1 (Vulkan portable, 35MB) | Fallback (zero-build) | ~570 t/s | 1071.6s / pp≈205 t/s / needle 3/3 | flags F4-3 |
| EngramHalo.cpp `strix-halo-qwen4exp` | ⛔ Legacy — do not deploy | ~450-570 t/s | n/a (legacy) | request-triggered heap-corruption defect (F4-2); unmaintained fork |

Selection rules:
1. Default = **Gufo** — depth prefill is the differentiator (200K: 2.8min vs
   17.9min vs legacy); decode sits in the same hardware band as the others.
2. Fallback = **Nathanw** when the container path is unavailable (uses
   UD-IQ4_XS or Q4_K_XL; mandatory flags F4-3).
3. EngramHalo = **legacy datapoint only**. Its custom lazy-PLE/host-buffer
   path exhibited a request-triggered heap-corruption defect (F4-2) and the
   fork is unmaintained — use the maintained engines above; do not try to
   "patch it back to life".

## F3. Gufo deployment recipe (S4-S6 worked example)

### Acquire (S4)
- **Engine image**: `ghcr.nju.edu.cn/gufo-org/toolboxes/gufo-runtime:latest`
  (NJU mirror of ghcr.io; 5.34GB; pull validated on this box).
- **Model**: ModelScope device-direct — `unsloth/Qwen3.8-Flash-Next-GGUF`,
  `UD-Q4_K_XL/*` (measured 38-45MB/s; ~45min for 111.3GB). Per-shard sha256
  from the ModelScope file API (`repo/files?Root=UD-Q4_K_XL` → `Sha256`);
  verify 4/4 before first load.
- **Shared MTP**: ModelScope `MTP/mtp-Qwen3.8-Flash-Next-shared-Q8_0.gguf`;
  cross-check its sha256 against the HF LFS oid (hf-mirror API `tree/main/MTP`).
- **Vision**: `mmproj-BF16.gguf` (same repo root).
- Container pulls MUST run with `TMPDIR=/data/flashnext/tmp` (F4-4).

### Service (S6) — unit rendered from `script/linux/flashnext.service.tmpl`
Core ExecStart (keep the deployed unit plus a saved previous-version unit as
the rollback pair):

```
ExecStart=/usr/bin/podman run --rm --name gufo-flashnext \
  --userns=keep-id:uid=1000,gid=1000 --security-opt label=disable \
  --device /dev/kfd --device /dev/dri --group-add keep-groups \
  --ulimit memlock=-1 --security-opt seccomp=unconfined \
  -p 8000:8000 -v /data/flashnext/models:/models:ro <IMAGE> \
  gufo serve llm -m /models/UD-Q4_K_XL/…-00001-of-00004.gguf \
  --mmproj /models/mmproj-BF16.gguf \
  --speculative mtp --mtp-model /models/MTP/mtp-…-shared-Q8_0.gguf \
  --served-model-name qwen3.8-flash-next \
  -j 1 -c 262144 -i 0.0.0.0 -p 8000 --log-progress
```

- Unit requirements: `User=<USER>`, `Environment=XDG_RUNTIME_DIR=/run/user/1000`
  and `TMPDIR=/data/flashnext/tmp`, `KillSignal=SIGINT` (forwards into the
  container), and the standard **ExecStartPre VRAM-drain wait** (restart races
  the async ~90GB BO reclaim). `loginctl enable-linger <USER>` MUST be on, or
  rootless podman cannot start at boot.
- Endpoint surface: `/health` ✓, `/v1/chat/completions`, `/v1/models`,
  `/v1/responses` (POST) — **no `/completion`**, no built-in WebUI. Clients
  must be OpenAI-shaped; the panel access card reflects this.
- `--served-model-name` MUST equal the id clients send
  (`qwen3.8-flash-next`); Gufo rejects unknown model names (llama.cpp did not).
- `-j 1` = one execution session (≈ old `-np 1`); context is native 262144.

### Panel registration (S6 step 4)
- Entry `flashnext`: `vram_gb=91`, `health_path=/health`,
  `model_dir=/data/flashnext/models/UD-Q4_K_XL`, `model_glob=*-00001-of-*.gguf`
  (entry-shard semantics unchanged, load-unload-panel.md P3).
- Helper `set-llama-model` rewrites the **container path**
  (`-m /models/UD-Q4_K_XL/…`) — the flashnext row must be adapted for podman units.

## F4. Pitfalls (each entry = a measured failure mode)

1. ⛔ **Some engines reject unsloth "dynamic" quant mixes.** Gufo dies at load
   on UD-IQ4_XS: `tensor blk.0.ffn_gate_exps.weight has unsupported format
   IQ3_S`. The officially supported target is **UD-Q4_K_XL** — check the
   engine's supported-quant list BEFORE downloading (F3).
2. ⛔ **Legacy-fork hazard: request-triggered heap corruption.** A forked
   engine's custom lazy-PLE/host-buffer path can corrupt the glibc heap during
   request processing. The symptom chain: sized requests eventually SEGV inside
   `malloc`/`free` (core shows `_int_malloc` unsorted-bin victim=NULL and
   `__libc_free` with the thread's TLS tcache pointer wild-written to
   `0x7f…00088x`) → the unit auto-restarts → looks healthy → the next request
   dies again. It is **config-independent** (MTP off / ngram-only / no-spec /
   reduced ctx all reproduce) — flag iteration will not fix it. **If you see
   this pattern, migrate to a maintained engine; do not debug the fork.**
3. **Nathanw fallback flags (v0.7.6.1)** — `--load-mode mmap --no-host
   --no-repack --fit off --tensor-read-lazy on` are mandatory on this model;
   `-md` combined with a spec-type that is not `draft-mtp` crashes at load;
   the `-eh.gguf` sidecar belongs to EngramHalo (use the original
   `mtp-…-Q8_0.gguf` for Nathanw); v0.7.6.1 MTP pays a recurrent-checkpoint
   restore per rejected round (net-negative on some short prompts).
4. ⛔ **podman on this box**: root LV is 15GB and podman stages pull blobs in
   `/var/tmp` → `no space left on device` mid-pull. Run every pull/run with
   `TMPDIR=/data/flashnext/tmp` (graphroot already `/data/containers/storage`).
   SELinux blocks container reads of the `/data` bind mount → `--security-opt
   label=disable`; GPU passthrough needs `--group-add keep-groups` (crun).
5. ⛔ **Endpoint/caliber trap**: raw `/completion` on this model returns a
   1-token empty answer (measured) — needle/acceptance tests MUST use
   `/v1/chat/completions`; Gufo has no `/completion` at all. For fresh-prefill
   measurements, prefix a nonce (slot caches otherwise make repeats instant).
6. ⛔ **MTP sidecar generations are not interchangeable**: Gufo wants
   `mtp-…-shared-Q8_0.gguf` (2.79GB); the llama.cpp original (4.14GB) and the
   EngramHalo `-eh` rewrite (4.14GB) each belong to another engine. All three
   files may sit in `MTP/` — check which one the unit passes.
7. **A full root filesystem silently drops coredumps** (`Cannot store
   coredump …: No space left on device`) — crash forensics become impossible.
   Fix: bind-mount `/data/coredump` over `/var/lib/systemd/coredump`, `chcon
   -t systemd_coredump_var_lib_t`, raise `ProcessSizeMax/ExternalSizeMax` in
   `/etc/systemd/coredump.conf.d/`.
8. **Orphan files masquerade as space**: a 32GB `/data/swapfile` existed but
   was never in `/proc/swaps` (live swap = zram 8GB). `swapon --show` before
   trusting `df` accounting; such orphans are space-recoverable.

## F5. Measured baselines (this box; testing-and-baselines.md caliber)

| Metric | Gufo (recommended) | Nathanw (fallback) | EngramHalo (legacy) |
|---|---|---|---|
| fresh 25K prefill | **~1436 t/s** | ~570 t/s | ~450-570 t/s |
| 200K needle (219,406 tok, 3 depths) | **166.6s total; pp≈1317 t/s; 3/3** | 1071.6s; pp≈205 t/s; 3/3 | n/a (legacy) |
| decode (MTP, filler/echo band) | ~30-45 t/s (draft-limit 7) | 19.6-35.7 t/s | 28.7-42.1 t/s |
| load | ~40-60s (container + 111GB mmap) | ~90s | ~75s |
| GPU resident | 90.8/98.3GB | ~83GB | 73.8/96GB |

Caliber traps: nonce-prefix fresh-prefill prompts; read speculative output,
not just the counter; the 200K needle is the production-parity acceptance
test for this family (L5).

## F6. Thinking-mode invocation contract

Verified on the current Gufo service: request body
`chat_template_kwargs: {"enable_thinking": false}` → direct answer,
`reasoning_content` empty, `content` clean. Default = thinking ON.

- The official card's three kwargs — `enable_thinking`, `reasoning_effort`,
  `preserve_thinking` — were all verified end-to-end on the previous
  (EngramHalo) service; on Gufo only `enable_thinking:false` has been
  re-verified. Re-verify the other two before relying on them on Gufo
  (Gufo also exposes server-side `--think / --reasoning-effort /
  --preserve-thinking` defaults — prefer per-request kwargs for client parity).
- Multi-turn preserved thinking requires the client to round-trip
  `reasoning_content` in assistant history; sampling recommendations from the
  official card stay client-side (no server-side pinning).
- ⛔ Never carry the retired q38 `--reasoning off` flag across families.

## F7. Ops / rollback / disk state

- **Rollback layer:** keep the previous engine's unit file and weights on disk
  (UD-IQ4_XS + original MTP + `mmproj-F16` serve the Nathanw route); prune
  them only once the new stack has proven stable in service.
- Logs: `journalctl -u flashnext` (unit lifecycle) + `podman logs
  gufo-flashnext` (engine; `--log-progress` prints prefill/decode progress).
- Rebuild from zero: F3 acquire (all CN-reachable mirrors; sha manifests in
  the ModelScope/HF APIs) → render unit from the template → panel
  re-register.
- Keep the acceptance record on disk (see testing-and-baselines.md).
