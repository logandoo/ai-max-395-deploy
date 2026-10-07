# Phase 2 (core) — Deploying ANY user-specified model (model-agnostic procedure)

Read this file IN FULL for every NEW model deployment. MUST/NEVER/STOP = enforced (SKILL.md §0).
Per-family specifics live in sibling files (`llm-flashnext-flashnext.md` — current resident;
`llm-qwen38-27b.md` — Qwen3.8-27B on Gufo + host-RAM staging constraint);
this file is the procedure that generalizes across models.

## 0. The seven-step loop (executed top to bottom; every step leaves an artifact)

> **Harness precedence:** if a vibeweaver harness is installed (opencode
> `vibeweaver` skill / deepseek `vibeweaver-dsh` plugin — detection in
> SKILL.md §0.1 and vibeweaver-binding.md), its binding contract GOVERNS the
> loop (acceptance-first, verification iterations, evidence gates, ADRs);
> the S-steps below are then the domain procedure executed INSIDE that
> contract, and the mapping table lives in vibeweaver-binding.md §3. In
> standalone mode this file's loop + Phase Gates govern as written.

```
S1 survey → S2 model/quant decision (memory math) → S3 engine decision (arch × ROCm version)
→ S4 acquisition (download routing) → S5 build/configure → S6 service → S7 verify (standard caliber)
→ S8 uninstall/replace (when the user says "换装/卸载")
```
Any step failing its check: STOP, diagnose, re-run. NEVER proceed past a red check.

## S1 — Survey (R3, MUST before any change)

Read-only snapshot, saved as evidence files: OS/kernel, `amd-smi list` (pool MiB),
`free -h`, `df -h`, **current GPU process inventory** (`ps -eo pid,args`,
VRAM used via `/sys/class/drm/card0/device/mem_info_vram_used`),
**existing model services** (`systemctl list-units | grep -iE "llama|vllm|comfy|speech"`).

- ⛔ Deploying a second model while another holds VRAM → decide FIRST: coexist
  (GPU budget, gpu-discipline.md) or **replace** (S8). NEVER "start it and see".
- Sudo on this device: the remote-ops channel has no saved-password
  elicitation → use the workspace `lib.sh` `rsh`/`rh` pattern (RP env +
  `sudo -S`), or key-auth ssh.

## S2 — Model/quant decision (memory math MUST be written down before downloading)

1. Identify the model's true memory split. For MoE models the checkpoint is
   **NOT one blob**: backbone (attention/shared) + experts + optional giant
   lookup tensors (qwen4exp `per_layer_token_embd` = 51.2B params ≈ 54.4GB at
   q8_0 — the single largest tensor). Read the safetensors/GGUF index and
   **write the three numbers down**.
2. Budget against the box topology (SKILL.md §0): **96GB GPU pool (BIOS carve)
   + 15.5GB GTT + ~30GB OS-visible RAM**. Pick the largest quant where
   `resident weights + KV(ctx, cache-type) + compute buffers ≤ 96GB`, ≥2GB slack.
3. If a **single tensor** exceeds the free pool at its load point, that quant
   is **undeployable as-resident** — pick a smaller quant, or an engine that
   streams that tensor from disk (mmap-lazy; **Gufo does this natively** for
   the qwen4exp PLE — the old EngramHalo fork was the historical
   implementation of the same idea).
4. ⛔ A quant that "just barely doesn't fit" is NOT fixable by flag iteration
   (measured dead-ends: n-cpu-moe sweeps, unified-memory env) — change the
   quant or the engine.
5. Record the decision as an ADR (options + memory ledger).

## S3 — Engine decision (architecture × ROCm version — the matrix that bit us)

The runtime is **per-model-family sensitive**, not global (container-native
engines remove the host-runtime coupling entirely):

| Model arch / family | Engine that worked | Runtime | Evidence |
|---|---|---|---|
| qwen4exp (Qwen3.8-Flash-Next) | **Gufo** (`gufo-org/gufo`, MIT, podman) — **CURRENT** | container ships its own HIP | 25K fresh ~1436 t/s; 200K 166.6s (pp≈1317); needle 3/3 |
| qwen4exp | Nathanw v0.7.6.1 Vulkan portable (fallback) | bundled RADV | 200K 1071s (pp≈205); flags in family file F4-3 |
| qwen4exp | EngramHalo.cpp `strix-halo-qwen4exp` (+ROCm 7.14 core) | 7.14 core | ⛔ legacy — request-triggered heap-corruption defect; do not deploy (family file F4-2) |
| qwen4exp | drluoto Vulkan fork / stock #27742 | any / 7.2.x | 6.9 t/s MTP collapse / load hang + depth cliff (#27856) |
| qwen35 (Qwen3.8-27B) | **Gufo** (`gufo serve llm`, podman) — **CURRENT** | container ships its own HIP | Q4_K_XL load 5.7s / GPU 39.7G @262K; ⛔ Q8_K_XL OOM on 30G-OS boxes (host staging, family file §2.2) |
| qwen35 | julianmb/q38rocm ROCmFPX (stock 215 / ovf) | 7.2.3 | validated; HISTORICAL (appendix in family file) |
| qwen3-ASR/TTS 1.7B | **Gufo** (`gufo serve asr\|tts`, BF16 safetensors) — **CURRENT** | container | ASR 15.27× RT / TTS 2.54× RT (official); round-trip word-exact (measured) |
| generic GGUF (llama/gpt-oss/etc.) | upstream llama.cpp ROCm | 7.2.3 | stock path |

Decision rules:
1. **Search first** (fork landscape moves weekly): llama.cpp discussions + HF
   model discussions for "<model> Strix Halo" — a tuned fork likely exists;
   landing the wrong fork costs a full rebuild cycle. **Check the engine's
   last-commit date + issue tracker before committing**: a fork that went
   docs-only while its architecture still has crash-class bugs is a dead end
   (EngramHalo lesson — migrate, don't debug).
2. **Prefer container-native engines when they exist** (Gufo): the image
   ships its own runtime, so host ROCm version is irrelevant. Two constraints
   before downloading anything: (a) the engine's **supported-quant list**
   (Gufo rejects unsloth dynamic mixes at load: `unsupported format IQ3_S`);
   (b) **sidecar generations** (MTP shared/original/`-eh` are per-engine).
   Host ROCm 7.2.3 remains for legacy stacks; container engines need no host
   prefix (an older EngramHalo deployment used a second 7.14-class runtime —
   method retained in system-base §1.3b).
3. ROCm-version landmine (measured): **7.2.3 DMA-blit hang loading qwen4exp**
   (eu-stack: `hsaCopyStagedOrPinned → BusyWaitSignal` user-space spin; ROCm
   issue #6027 family) — new architectures are suspect on <7.11.
4. ⛔ `GGML_CUDA_ENABLE_UNIFIED_MEMORY` MUST stay ABSENT for qwen4exp
   (controlled A/B: garbage output; ggml checks presence, not value).
5. ⛔ Speculative/MTP on **Vulkan llama.cpp forks** collapsed historically
   (drluoto 6.9 t/s); the Nathanw stack later made Vulkan MTP usable
   (fallback tier) and Gufo does MTP natively. Pick the backend by the
   engine's own validated table, not by backend folklore.
6. Speculative decoding needs a **matching draft sidecar AND loader naming**:
   GGUF tensor names move between converter releases (`blk.N.nextn.hc_head_*`
   vs `output_hc_*`). Verify the sidecar loads with THIS engine before
   benchmarking; a metadata-key rename via python `gguf` is a 4GB, 1-minute fix
   (mind the pip-gguf writer API: `write_header_to_file → write_kv_data_to_file
   → write_ti_data_to_file → write_tensor_data × N → flush/close`).

## S4 — Acquisition (download routing R1 + sha R2)

1. CN-reachable (ModelScope/Huawei/Aliyun/hf-mirror) → device direct
   (ModelScope ~40-130MB/s). HF/GitHub → **Mac relay** then scp. GitHub
   release/archive assets also work device-direct via
   `ghfast.top/<original-url>` (validated). `repo.amd.com` from device
   ≈ 1MB/s → relay its RPMs via Mac.
2. Verify byte sizes against the repo API manifest (ModelScope
   `/api/v1/models/<id>/repo/files?Recursive=true`, HF tree) — N/N exact or STOP.
3. Prefer official mirrors of quants (unsloth ships to ModelScope within days).
4. ⛔ `scp -q file host:/tmp/` can fail silently — scp to an **explicit
   filename** and `ls -la` on the machine before use.
5. Space check before big downloads: root LV is 15G (dnf of multi-GB stacks
   exhausts it) — /data is the payload home (S5).
6. **sha manifests come from the APIs, not guesswork:** ModelScope
   `…/repo/files?Root=<dir>` returns a `Sha256` per file (verify N/N after
   download — a 4-shard 111.3GB fetch was re-verified 4/4 against the
   manifest); the HF side
   gives the LFS oid via the tree API (`hf-mirror.com/api/models/<id>/tree/
   <rev>/<dir>`) — cross-check sidecars between mirrors. Measured bulk speed
   on this box: ModelScope device-direct 38-45MB/s.
7. **Container images**: `ghcr.io/<org>/<img>` is not reliably reachable from
   the device — use the `ghcr.nju.edu.cn/<org>/<img>` mirror (validated for
   the Gufo runtime, 5.34GB). Pull with `TMPDIR` on /data (S5.9) or the pull
   dies on the full root LV.

## S5 — Build/configure

1. Shallow-clone the pinned branch **on the Mac** (R1), tar excluding `.git`
   (`COPYFILE_DISABLE=1`), scp, extract on device.
2. Build against the S3-selected ROCm prefix:
   `ROCM_PATH=<prefix> CMAKE_PREFIX_PATH=<prefix> PATH=<prefix>/bin:$PATH`.
3. ROCm runtime acquisition without dnf (root LV 15G — a multi-GB `dnf
   install` WILL exhaust `/`): `rpm2cpio *.rpm | cpio -idm` into `/data/<dir>`,
   then `ln -s` into `/opt/rocm/<name>`. Dependency closure: iterate
   `rpm -qpR` × repo `primary.xml` server-side until fixpoint
   (worked example shipped in this package's family file).
4. Missing build headers (hipcub/rocprim/SPIRV-headers): `dnf download` +
   rpm2cpio into `/opt/rocm/include` — never a full stack `dnf install`.
5. ⛔ **SELinux**: EVERY new executable directory under `/data` MUST get
   `chcon -R -t bin_t <dir>` before systemd can exec it — the rule applies to
   every new build directory. Symptom: unit fails `status=203/EXEC Permission denied`.
6. Engine flags: start from the fork's own validated recipe table, record
   them in the deployment config, render the systemd unit from a template —
   the deployment config is the **single source of truth** (no drift between
   validated config and deployed unit).
7. Preflight in the deploy script: binary exists + the exact flags the unit
   passes exist in `llama-server --help` (fork flags drift — `--tensor-read-lazy`
   existed upstream but not in one fork's build).
8. Python-side GGUF tooling: pip `gguf` on device (Tsinghua mirror); pip-gguf
   writer API differs from the repo gguf-py — introspect before scripting.
9. **Container engines (no build step) — the current default for qwen4exp.**
   `podman pull` the mirrored image, then run it; there is NO source build and
   host ROCm is irrelevant. Non-negotiables learned the hard way (all measured):
   - **Rootless podman**: graphroot MUST live on /data
     (`~/.config/containers/storage.conf`); root LV cannot hold images.
   - **TMPDIR MUST be on /data** for pull/run (`TMPDIR=/data/<…>`): podman
     stages blobs in `/var/tmp` → root-full = "no space left on device"
     mid-pull.
   - **SELinux**: `--security-opt label=disable` is required for containers
     reading `/data` bind mounts (symptom without it: `Failed to open file` at
     model load — looks like a missing file, is a label denial).
   - **GPU passthrough**: `--device /dev/kfd --device /dev/dri --group-add
     keep-groups --ulimit memlock=-1 --security-opt seccomp=unconfined`
     (keep-groups needs crun).
   - **Bench the engine's supported quant list BEFORE a 100GB download**:
     Gufo rejects unsloth dynamic mixes (`unsupported format IQ3_S`) — only
     the documented target (UD-Q4_K_XL) is safe.
   - **Test endpoints**: run `--help` + probe `/health` + `/v1/models`; not
     all engines keep llama.cpp's `/completion` or a WebUI.

## S6 — Service registration

1. systemd unit from template: `Type=simple`, `Restart=on-failure`,
   `KillSignal=SIGINT` (ROCm teardown segfaults on SIGTERM), `TimeoutStopSec=240`,
   and an **ExecStartPre VRAM-drain wait** (restart races the async ~74GB BO
   reclaim → allocator crash):
   ```ini
   ExecStartPre=/bin/sh -c 'i=0; while [ $i -lt 90 ]; do v=$(cat /sys/class/drm/card0/device/mem_info_vram_used 2>/dev/null || echo 999); [ "$v" -lt 8589934592 ] && break; sleep 1; i=$((i+1)); done; exit 0'
   ```
2. ⛔ After ANY clean stop the unit may rest in **`failed`** (teardown SEGV) —
   lifecycle code MUST accept `inactive|failed` as terminal-stopped. A red
   unit state after a deliberate stop is expected cosmetics, not an incident.
3. Lifecycle scripts `<id>_{deploy,start,stop,status,rollback}.sh` —
   cgroup-safe (no pkill), health-poll timeouts sized to the model load
   (93GB mmap ≈ 75-120s). ⛔ `--load-mode dio` stages ALL GPU weights through
   host RAM: on this 30GB-OS-visible box that OOMs (23G RSS + 35G swap peak,
   measured) — mmap is the default.
4. If the machine runs a model load/unload surface, register the new unit in
   it — a deployed-but-unregistered model is not operable by the user.
5. **Container unit pattern (Gufo-class engines).** Same lifecycle rules, but
   the ExecStart is a foreground `podman run --rm --name <c>` and the unit
   needs two extra pieces:
   ```ini
   [Service]
   User=<user>
   Environment=XDG_RUNTIME_DIR=/run/user/<uid>
   Environment=TMPDIR=/data/<dir>/tmp
   ExecStart=/usr/bin/podman run --rm --name <c> \
     --userns=keep-id:uid=<uid>,gid=<gid> --security-opt label=disable \
     --device /dev/kfd --device /dev/dri --group-add keep-groups \
     --ulimit memlock=-1 --security-opt seccomp=unconfined \
     -p <port>:<port> -v /data/<dir>/models:/models:ro <IMAGE> \
     <engine> <serve-args…>
   KillSignal=SIGINT
   ```
   - `loginctl enable-linger <user>` MUST be on, or a rootless unit cannot
     start at boot; podman forwards SIGINT into the container.
   - Non-ROCm engines may stop **clean** (unit ends `inactive`, not `failed`) —
     lifecycle code accepts both (R14).
   - `--served-model-name` (or engine equivalent) MUST equal the id clients
     send — Gufo rejects unknown model names where llama-server ignored the
     field. Add it to the preflight checklist.
   - `active_model` attribution on panels still works: the podman argv
     carries `-m <path>` (but the path is the **container** path — helper
     scripts that sed `-m` must target `/models/...`, not the host path).

## S7 — Verify (standard caliber)

- **Prefill MUST be measured with ≥4096-token prompts** (`pp4096`). A
  ~30-token prompt "prefill" reads ~40 t/s and is meaningless (fixed overhead
  dominates); the same server measures **477 t/s** at pp4096. Compare only
  same-caliber numbers, ideally against the fork's published table for this
  silicon.
- Decode at d0 **and** at depth (≥4k) — depth cliffs are arch+backend+fork
  specific (QSA indexer paths; stock #27742 cliffed 3.5×, tuned forks don't).
- Speculative config A/B: draft depth ladder (n4/n6/n9/n12) on the SAME
  prompt; prose plateaus early, code/echo workloads keep climbing (measured
  28.7 → 31.5 prose, the machine → 42.1 code-rewrite between n4 and n12).
- **Read the output** — speculative decoding can report great t/s while
  emitting garbage (measured twice on this model family).
- **Large-context acceptance = needle test via `/v1/chat/completions`**, with
  a nonce-prefixed prompt (forces fresh prefill — slot caches otherwise fake
  instant prefill). 3 needles at ~10/50/90% depth of a ~200K-token haystack;
  gate = 3/3 recall + wall time within the engine band (measured:
  Gufo 219,406 tok in 166.6s / pp≈1317; Nathanw 1071.6s / pp≈205).
- ⛔ Do NOT use raw `/completion` for acceptance: some engines don't
  implement it (Gufo has none) and this model returned a 1-token empty answer
  through it (measured) — endpoint mismatch looks like model failure.
- Gate: acceptance criteria in an acceptance file BEFORE S2 (vibeweaver
  loop); every claim → artifact on disk (R8).

## S8 — Offload / replace

1. Survey + inventory (du per dir, unit list, load/unload surface state) — before evidence.
2. Stop + disable old units; **archive small irreplaceable artifacts**
   (engine binaries, presets, unit files) to `rollback/`.
3. Delete model weights only (venvs/code are cheap and reversible — but SAY
   so in the report: a 18G venv dir is not "models left behind").
4. Update the load/unload surface registry (remove dead entries) + firewall
   if ports change.
5. Keep a written re-download path for everything deleted (URLs in config).
6. Scripts MUST be gated (`UNINSTALL=YES`) and idempotent on re-run.

## Phase Gate 2G — MUST pass before declaring ANY model deployment done

| # | Check | Expected |
|---|---|---|
| G2G.1 | `systemctl is-active <unit>` | active |
| G2G.2 | `curl :<port>/health` | ok |
| G2G.3 | `curl :<port>/v1/models` | lists the model (+ vision capability if mmproj mounted) |
| G2G.4 | pp4096 (≥4096-token prompt) | within the fork's published band for this silicon |
| G2G.5 | tg256 @ d0 and @ ≥4k depth | no cliff; within published band; output READ and clean |
| G2G.6 | `chcon -R -t bin_t` on the binary dir | applied — **N/A for container engines** (no /data binary exec; instead verify the image is present + pinned) |
| G2G.7 | the model is registered in the machine's load/unload surface | registered + operable |
| G2G.8 | lifecycle scripts exist workspace+device, stop accepts `inactive\|failed` | pass |
| G2G.9 | rollback path: archive + re-download URLs written | pass |
| G2G.10 | served model id matches what clients send (`--served-model-name` / alias) | `curl` with the client's model name returns 200 (not 404 model_not_found) |
| G2G.11 | 200K needle acceptance (fresh, `/v1/chat/completions`) | 3/3 recall; wall time within engine band |

**If any check fails: STOP — diagnose, fix, re-run.**

## S9 — Restore (weights missing / corrupted / host migrated)

This procedure applies when weights are gone or suspect and the service
must be made available again.

1. **Inventory against manifests FIRST.** Every stack's file list lives in a
   deploy script or family file (ComfyUI: `comfyui_deploy.sh` ERNIE_FILES /
   KREA_FILES; 27B: pinned sha256 in the deployment config; speech:
   `models_download.sh`; OCR: `models_all.sh`). Write the full list with
   sizes + sha256 before downloading.
2. **Download routing** (S4) — measured mirror coverage:
   ModelScope device-direct for ComfyUI/ASR/TTS/OCR/Qwen3.6; **NO ModelScope
   mirror for** julianmb ROCmFP4 GGUFs and `nvidia/bigvgan*` → Mac relay
   (hf-mirror from Mac: 6-43MB/s; from the DEVICE hf-mirror stalls at 0 —
   never use it there).
3. ⛔ **sha256 is MANDATORY after every download, and `-C -` resume of a
   dropped connection can corrupt** (observed: resumed file +15.8MB oversize,
   sha mismatch). Rule: interrupt → resume is fine, but the final artifact
   MUST pass sha; if sha fails, DELETE and re-download fresh (never resume a
   sha-failed file twice).
4. **Exact-filename trap:** runtime code opens literal names
   (`config.json`, not `bigvgan_config.json`). Renaming downloaded files to
   the expected names is part of placement.
5. **Incomplete downloader scripts are a failure mode:** the speech
   `models_download.sh` lacked SenseVoiceSmall (the ASR server's default
   model since the sensevoice switch) — ASR failed with "model not
   registered". Audit each downloader script against what its service
   actually loads, fix the script (workspace + device), then download.
6. **Dangling symlinks raise FileExistsError:** paddle service startup
   `makedirs(~/.paddlex)` over a dangling symlink → crash.
   Fix: `rm` symlink, `mkdir -p` real dir (chown), re-`ln -sfn`.
7. **Per-stack availability smoke (start → health → stop = offload)** is the
   availability proof — run it per family, not one batch claim. Special
   cases: q38-llama (historical family — restore recipe in
   llm-qwen38-27b.md) and flashnext share :8000 (mutually exclusive residents
   — swap-proof: stop resident, start other, verify, restore, measured
   15s/75s); speech-tts loads ~10GB (health window ≥6 min); ComfyUI starts
   lazy (server up ≠ models loaded — run one generation).
8. Tolerable-failure knowledge: IndexTTS `anti_alias_activation_cuda` JIT
   build failure is NON-fatal (native fallback) — the fatal error is missing
   bigvgan files. Don't chase the ext build; place the files.

