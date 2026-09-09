# Phase 2 (family) — Qwen3.8-Flash-Next (qwen4exp) — CURRENT RESIDENT on the machine

Worked example of model-deploy-generic.md on this exact box (the resident
Flash-Next deployment on this box). Read the
generic file first.

## F1. Model & memory ledger (S2 worked example)

- unsloth/Qwen3.8-Flash-Next-GGUF **UD-IQ4_XS**: 93.66GB total = backbone
  39.33GB + **PLE/engram single tensor 54.4GB (q8_0)** + MTP sidecar 4.13GB
  + mmproj-F16 0.90GB. Total 98.4GB.
- Topology: 96GB GPU pool + 15.5GB GTT + ~30GB OS RAM.
- ⛔ Dead-end ledger (do NOT retry): all-in-VRAM (54.4GB single tensor
  allocation spins hipMalloc even with 59GB free), n-cpu-moe sweeps
  (12/20 — freeze point moves, still spins), unified-memory env (OOM + a
  garbage-output poison), dio load (OOM: stages all GPU weights through
  30GB host RAM). Working answer: **EngramHalo branch streams the PLE
  mmap-lazy from SSD** (~1GiB hot set; VRAM 73.8GB = backbone+MTP+KV+bufs).

## F2. Engine (S3 worked example)

- **Aristo94/EngramHalo.cpp @ strix-halo-qwen4exp** + bundled
  `llama-cpp-25992-rocm-host-buffer.patch` (apply; the per-buffer-mmap patch
  is obsolete upstream). Base: post-#27742 master.
- Built against **ROCm 7.14.60850 core** at `/opt/rocm/core-7.14`
  (symlink → `/data/rocm714/opt/rocm/core-7.14`, 47 amdrocm-*7.14.1 RPMs
  rpm2cpio-extracted — root LV 15G cannot hold a dnf stack install).
  Main `/opt/rocm` (7.2.3) untouched — coexistence by design.
- ⛔ 7.2.3 loads of qwen4exp hang in DMA blit (user-space spin,
  `hsaCopyStagedOrPinned`); Vulkan = 6.9 t/s MTP collapse; stock #27742 =
  hang + depth cliff. EngramHalo + 7.14 is the only validated fast path on
  this silicon (39.2 t/s their bench, reproduced).
- Build: `-DGGML_HIP=ON -DAMDGPU_TARGETS=gfx1151 -DGGML_RPC=ON
  -DROCM_PATH=/opt/rocm/core-7.14 -DHIP_PLATFORM=amd -DCMAKE_PREFIX_PATH=…`
  (+ hipcub/rocprim/SPIRV-headers-devel via dnf-download+cpio when missing).

## F3. Production launch parameters (flashnext.service)

```
-m UD-IQ4_XS-00001-of-00003.gguf -md MTP/mtp-Qwen3.8-Flash-Next-Q8_0-eh.gguf
--spec-type draft-mtp,ngram-mod --spec-draft-n-max 12 --spec-draft-p-min 0.70
--mmproj mmproj-F16.gguf -ngl 999 -fa on -ctk q8_0 -ctv q8_0
-lm mmap -c 163840 -np 1 -b 8192 -ub 2048 -t 4 --jinja
--alias qwen3.8-flash-next --host 0.0.0.0 --port 8000
+ unit env: LD_LIBRARY_PATH=/opt/rocm/core-7.14/lib:…/rocm_sysdeps/lib, ROCBLAS_USE_HIPBLASLT=1
```

- **MTP sidecar MUST be the `-eh.gguf` rewrite**: unsloth's current converter
  names the head tensors `blk.48.nextn.hc_head_{norm,down,up}` while this
  fork expects detached `output_hc_{norm,down,up}`. The rewrite (python
  `gguf`, 3 keys, 4.1GB, shapes 10240 / 10240×320 / 320×10240) lives beside
  the original. A fresh unsloth re-download needs the same treatment.
- ctx=262144 validated on this box with MTP (30k-depth needle HIT; the
EngramHalo recipe upstream validates MTP only up to a 164K slot).
- Draft depth A/B (measured, same prompt): prose tg256@d0 = 28.7 (n4/p0.75)
  → 31.5 (n6) → 31.7 (n12) plateau; **code/file-rewrite = 42.1 t/s at n12**
  (MTP 89.5%). ngram-mod misses cost nothing → n12 is the keeper.

## F4. Measured baselines (this box — testing-and-baselines.md caliber)

| Metric | Value | Reference band (same fork+silicon) |
|---|---|---|
| **pp4096** | **477.3 t/s** | EngramHalo 396-502 · kingjones777 385 |
| tg256 @ d0 (prose) | 28.7-31.7 t/s (n4→n12) | EngramHalo 35.3-39.3 on code tg400 |
| tg @ d4k | 33.5 t/s (no cliff) | stock #27742 cliffs 3.5× (#27856) |
| code/file-rewrite @ n12 | **42.1 t/s** (MTP 89.5%) | drluoto 47.1 (echo-heavy, 128G-RAM box) |
| load (mmap, cold-ish) | HEALTH_OK ~75s | dio=33s but OOMs on 30GB-OS box |
| VRAM resident | 73.8/96GB | by design (PLE on SSD-lazy) |

⛔ Caliber trap: a ~30-token prompt "prefill" reads ~40 t/s — meaningless.
pp4096 or nothing.

## F5. Operational facts

- Service `flashnext.service` :8000, lifecycle
  `script/linux/flashnext_{deploy,start,stop,status,rollback}.sh`.
- Clean stop lands the unit in **`failed`** (ROCm teardown SEGV) — expected;
  lifecycle code accepts `inactive|failed`.
- Restart needs the ExecStartPre VRAM drain (74GB reclaim is async).
- Re-download path (rollback): ModelScope `unsloth/Qwen3.8-Flash-Next-GGUF`
  (`UD-IQ4_XS/*`, `MTP/...`, `mmproj-F16.gguf`) + rename step F3 + engine
  rebuild recipe F2 (source tarball relayed via Mac).
- On the reference deployment the other families run offloaded (inactive,
  startable on demand).
  q38 rollback = `flashnext_rollback.sh` (archive + ghfast re-download 14.5GB).

## F6. Thinking-mode invocation contract (verified against the official model card)

The official card controls thinking via THREE chat-template kwargs — all three
verified working on this deployment through `/v1/chat/completions`
`chat_template_kwargs`:

| kwarg | official default | verified behavior on this box |
|---|---|---|
| `enable_thinking` | true (undefined → ON) | ON by default; thinking parsed into `reasoning_content`, `content` clean. `false` → direct answer, `reasoning_content=None` |
| `reasoning_effort` | xhigh (xhigh/medium/low) | accepted; `low` measurably shortens reasoning |
| `preserve_thinking` | true (multi-turn: re-inject `<think>+reasoning_content` into history) | behavioral differential PASS: with history `reasoning_content` round-tripped, model recalls a code word from prior-turn thinking (`KNOWS`); `false` → genuinely stripped (`DOES-NOT-KNOW`) |

Client contract (MUST document for consumers):
1. Thinking mode default = ON; clients read `reasoning_content` (separated by
   llama.cpp reasoning-format auto) — matching the official card\u0027s
   framework-separation guidance. Raw `/completion` still emits inline
   `<think>` (expected, native endpoint does not parse).
2. **Multi-turn preserved thinking requires the client to send
   `reasoning_content` back inside assistant history messages** — the
   template re-injects it as `<think>` blocks (official `preserve_thinking:
   True` semantics). A client that drops reasoning_content silently gets
   stripped history (off-semantics).
3. Official sampling recommendations: thinking mode `temp=0.6, top_p=0.95,
   top_k=20`; non-thinking `temp=0.7, top_p=0.80, top_k=20, min_p=0,
   presence_penalty=1.5`. Our unit does NOT pin sampling server-side — pass
   these client-side when benchmarking quality.
4. ⛔ Do NOT carry over the old q38 flag `--reasoning off` — this is a
   thinking model; the flag would fight the template (old unit lineage).
5. Probes: behavioral differentials on the live service (code-word recall
   with `preserve_thinking` on/off); template source:
   `tokenizer.chat_template` in GGUF shard 1 (unsloth-fixed: developer role /
   merged system / tool calling — thinking mechanics identical to official).

