# Phase 0+1 — System Base (Fedora 44 Server / Workstation)

Read this file IN FULL before running any command of Phase 0 or Phase 1.
Keyword convention: MUST/MUST NOT/NEVER/STOP are enforced rules (SKILL.md §0).

## 1.0 Distro portability (Fedora validated · Ubuntu/Arch procedure-projected)

**Validation status: ONLY Fedora 44 is end-to-end validated** (every command
below was executed on the machine). Ubuntu/Arch substitutions below are
**projected from upstream packaging, NOT yet validated on hardware** — treat
as a starting map, verify each step at deploy time (and record deltas back
into this file).

| Component | Fedora 44 (✅ validated) | Ubuntu (⚠️ projected) | Arch (⚠️ projected) |
|---|---|---|---|
| Kernel freshness | 7.1.10 default (✓ gfx1151 mature) | 24.04 GA 6.8 **too old** → use HWE/OEM (≥6.11); 25.04+ fine | rolling ✓ |
| ROCm base install | el9 RPM closure via `amdgpu-install` (§1.3) — first-class | **easiest of the three**: `amdgpu-install` .deb from repo.radeon.com (Ubuntu LTS officially supported) or repo.amd.com apt pool | `[extra]` rocm packages (`rocm-core`, `hip-runtime-amd`, …) or TheRock tarball |
| ROCm 7.14-class core | rpm2cpio closure → `/opt/rocm/core-7.14` (§1.3b) | repo.amd.com publishes `.deb` pools — `dpkg-deb -x` to `/data` + symlink, same shape; or TheRock dist tarball | same manual-extract shape; Arch rocm tracks upstream fast — check `pacman -Si rocm-core` version first |
| Multi-version coexistence | `/opt/rocm` + `/opt/rocm/core-7.14` symlink | same symlink pattern | same |
| Kernel params (R7) | `grubby --update-kernel=ALL` + verify `/proc/cmdline` | `/etc/default/grub` + `update-grub` + verify | `grub-mkconfig -o` (or systemd-boot loader entries) + verify |
| Security module | **SELinux enforcing** → ⛔ `chcon -R -t bin_t` on every /data bin dir (203/EXEC) | AppArmor — exec of /data binaries not blocked; **no chcon step** | none — **no chcon step** |
| Firewall | firewalld (`--permanent --add-port`) | `ufw allow <port>` (if ufw enabled; often none on servers) | none by default — open nothing unless a firewall is installed |
| `/etc/profile.d/rocm.sh` PATH guard | required for manual closures | required for manual extractions (deb/apk installs set ldconfig themselves) | same |
| Python | 3.14 system (pip gguf quirks noted) | 3.12 on 24.04 — pip-gguf API drift note applies per version | rolling |
| TheRock dist tarballs | ✅ works | ✅ works | ✅ works |

Everything else in the seven-step loop (S1-S8: memory math, engine decision,
  (systemctl list-units | grep -iE "llama|vllm|comfy|speech")),
The one caveat that does NOT transfer: SELinux 203/EXEC is Fedora-only — on
Ubuntu/Arch, a failing unit with "Permission denied" is something else
(paths/ownership), diagnose from `journalctl` first.

## Phase 0 — Prerequisites

1. **Download strategy (R1, MUST):**
   - CN-reachable sources (ModelScope, Huawei Cloud, Aliyun, fedora mirrors)
     → download **directly on the machine** (ModelScope measured ~60MB/s).
   - HF / GitHub / blocked PyPI artifacts → download on the **relay host
     (Mac)**, then scp/rsync to the machine.
   - Large artifacts: verify **sha256 on both ends** (relay + device).
2. **Credentials:** keep SSH/sudo passwords in `.env.local` (gitignored);
   reference them from config — never hardcode (R9).
3. **Baseline survey (R3, MUST) before any change** — read-only snapshot:
   OS release, kernel, `lscpu`, `free -h`, `lsblk`/LVM, NICs, `lsmod`,
  `dmesg | tail`. Store the survey as an evidence file on the relay host.
4. Verify ModelScope/HF URLs exist **before** writing download scripts —
   models get delisted (SeACo and plain paraformer-large are already 404).

## Phase 1 — System base

### 1.1 BIOS / kernel

- BIOS **MUST** carve the 96GB GPU pool (otherwise ROCm sees only a tiny
  VRAM slice and nothing else matters). Verify: `amd-smi list` shows GPU0
  with **98304 MiB**. *(BIOS setting is vendor firmware — distro-independent.)*
- Kernel **MUST** be gfx1151-mature (≥6.11 class; Fedora 44 ships 7.1.10 ✓).
  Ubuntu 24.04 GA 6.8 is too old — boot HWE/OEM first. Cmdline **MUST**
  contain `iommu=pt` (required for NPU SVA and part of the validated baseline).
- ⛔ Kernel-parameter changes are **distro-tool-specific and MUST be
  verified via `/proc/cmdline` afterwards** (the principle behind R7):
  Fedora `grubby --update-kernel=ALL` (⛔ `grub2-mkconfig` alone drops
  `root=`/`iommu=pt` from BLS entries — real boot failure) · Ubuntu
  `/etc/default/grub` + `update-grub` · Arch `grub-mkconfig` or the
  systemd-boot loader entry.
- NIC vendor driver r8127: build/install **against the running kernel**
  (a kernel upgrade removes the module; in-kernel r8169 is a usable
  fallback); enable via `modules-load.d`. Wired static IP strongly
  recommended (Wi-Fi off on the validated device).

### 1.2 Storage — edition-specific

**Server (validated):** LVM the free disk → xfs → mount `/data`
(fstab by UUID). All models/engines/venvs live under `/data`.
```
lvcreate fedora -n lvol0 -l 100%FREE && mkfs.xfs /dev/fedora/lvol0
# fstab: UUID=<uuid>  /data  xfs  defaults  0 0  → systemctl daemon-reload && mount /data
```

**Workstation (variant):** **no extra mount** — create `$HOME/aimax` and
use it as the base dir. Complete the **path-substitution checklist**
  a device-side deployment config, systemd units, and
device-side scripts **before** Phase 2. SELinux: if a systemd unit fails
with `Permission denied` on a home-dir binary → `chcon -t bin_t <binary>`.

### 1.3 ROCm 7.2.3 base (validated path on Fedora 44 — Ubuntu/Arch see §1.0 matrix)

```bash
# F44 has NO fedora/44 directory; the upstream README's 9.5 path is 404 → use el9 pkgs
dnf install amdgpu-install-7.2.3.70203-1.el9.noarch.rpm   # prefer a local file repo (40 RPMs, sha256-verified, offline-reinstallable)
amdgpu-install --usecase=rocm --no-32bit                  # runtime: rocm-dev hip hipblas rocblas hipblaslt hsa-rocr rocsolver comgr
```

- ⛔ Manual RPM-closure install ships **no profile script** → **MUST**
  create `/etc/profile.d/rocm.sh` (adds `/opt/rocm/bin` to PATH, with a
  de-dup guard). Without it, non-login shells (ssh commands, systemd)
  cannot find `amd-smi`/`rocm-smi`.
- Mixed ABI is proven to work: `/opt/rocm` 7.2.3 core + fedora 7.1
  hipblas/rocblas/rocsolver.
- **ROCm 10: do NOT upgrade yet** — no gfx1151 decode-kernel gain, engines
  are bound to the 7.2.x ABI, and the el9 workaround chain carries
  regression risk. Re-evaluate on: upstream ROCm10 prebuilt, or a
  third-party A/B decode gain >10%.
- Verify: `rocminfo` lists Agent gfx1151 **and** Agent aie2p;
  `rocm-smi` shows **95.99 GiB** VRAM.

### 1.3b ROCm 7.14 core — second runtime, coexists with 7.2.3

Newer model families (qwen4exp) need ROCm ≥7.11-class userspace: 7.2.3 hangs
their loads in DMA blit (`hsaCopyStagedOrPinned` user-space spin — ROCm issue
#6027 family). **Two runtimes coexist by design:**

- `/opt/rocm` → 7.2.3 (q35/ROCmFPX family baseline; do not touch)
- `/opt/rocm/core-7.14` → symlink to `/data/rocm714/opt/rocm/core-7.14`

Acquisition (root LV is 15G — a multi-GB `dnf install` exhausts `/`):
1. RPMs: `amdrocm-{core-devel7.14-gfx1151, runtime7.14, blas7.14-gfx1151}`
   from `https://repo.amd.com/rocm/packages-multi-arch/rhel10/x86_64/`
   (Fedora 44 accepts the rhel10 packages; glibc 2.42 ≥ el10 requirement).
2. ⛔ Device-direct speed ≈ 1MB/s → download on the **Mac relay**; resolve the
   dependency closure by iterating `rpm -qpR` on downloaded RPMs against the
   repo `primary.xml` (provides→package map) until fixpoint (~47 RPMs, 1.5GB).
3. Extract WITHOUT rpmdb: `rpm2cpio *.rpm | cpio -idm` under `/data/rocm714`
   (as the service user), then `sudo ln -sfn <dir> /opt/rocm/core-7.14`.
4. Build engines against it: `ROCM_PATH=/opt/rocm/core-7.14
   CMAKE_PREFIX_PATH=… PATH=/opt/rocm/core-7.14/bin:$PATH`.
5. Missing build headers (hipcub/rocprim/glslang/SPIRV-headers): `dnf
   download` the `-devel` RPMs + rpm2cpio into `/opt/rocm/include` (worked
   for hipcub 4.2.0.70203 on 7.2.3 too).

### 1.3c Transfer & shell pitfalls

- ⛔ zsh does NOT word-split unquoted `$var` (`set -- $var` yields ONE
  argument) — split explicitly (`$=var` in zsh or an array) or run the loop
  under `bash -c`.
- ⛔ `scp -q file host:/dir/` can fail SILENTLY (observed 2×) — scp to an
  explicit remote filename and `ls -la` it on the machine before use.
- GitHub assets from the machine work via `ghfast.top/<original-github-url>`
  (validated for release tarballs + model files); repo.amd.com is ~1MB/s
  device-direct → Mac relay.

### 1.4 Hardware tuning

- `q38-hw-tweak.service`: `governor=high` + THP madvise (unit retries until
  the card enumerates). Workstation: also set the power profile to
  **performance** (power-profiles-daemon).

### 1.5 Firewall (Server AND Workstation)

- Serving ports **MUST** be reachable — mechanism is distro-specific:
  Fedora firewalld (validated): `sudo firewall-cmd --permanent --add-port=8000/tcp && sudo firewall-cmd --reload`
  Ubuntu: `sudo ufw allow 8000/tcp` (only if ufw is enabled — often inactive on servers)
  Arch: nothing by default — check whether any firewall is installed first.
- Workstation's default zone is restrictive — this step is easy to miss
  there (see SKILL.md §3 checklist).

### 1.6 SSH / transfer pitfalls (hit in production)

1. Remote multiline scripts: `ssh user@host 'bash -s' <<'REMOTE'`
   (quoted heredoc — prevents local expansion).
2. ⛔ `echo "$PW" | sudo -S tee FILE` **swallows the pipe** (sudo's password
   `read()` eats the whole buffer; tee creates an empty file) → write to
   `/tmp` first, then `sudo cp` into place.
3. ⛔ NEVER scp-overwrite a bash script **while it is running** (bash reads
   by offset — undefined behavior) → upload under a new name, then `mv`.
4. ⛔ ssh-inline `nohup bash -c "...$VAR..."` leaks `$VAR` to local
   expansion → always scp a script file instead.
5. Piping into `tail` swallows the script's exit code → `set -o pipefail`.
6. macOS tar/rsync: set `COPYFILE_DISABLE=1` (AppleDouble pollution).
7. Env vars across ssh: assemble **one** variable locally (or base64) and
   pass it through; beware `${VAR:-default}` treats an empty string as set.
8. SSH faillock: high-frequency password auth locks the account 2–5 min
   (auto-unlock) — throttle batch jobs at night.
9. Identify "what is running" via `/proc/<pid>/cmdline` — never by process
   name or startup banners (banners can mask bind failures).

## Phase Gate 1 — MUST pass before Phase 2/3/4

| # | Check | Expected |
|---|---|---|
| G1.1 | `cat /proc/cmdline` | contains `iommu=pt` |
| G1.2 | `amd-smi list` | GPU0 BDF f4:00.0, **98304 MiB** |
| G1.3 | `rocminfo \| grep -E "gfx1151\|aie2p"` | both agents listed |
| G1.4 | `rocm-smi --showmeminfo vram \| grep -i total` (or `rocm-smi`) | ≈95.99 GiB VRAM |
| G1.5 | Server: `df -h /data` → xfs mounted, ≥400GB · Workstation: `ls $HOME/aimax` → base dir exists | pass |
| G1.6 | `bash -lc 'command -v amd-smi rocm-smi rocminfo'` | all three resolve |
| G1.7 | baseline survey saved as a dated evidence file | pass |
| G1.8 | `firewall-cmd --list-ports` | contains `8000/tcp` (+ `8201/tcp 8202/tcp` once speech deploys) |

**If any check fails: STOP. Fix, re-run the gate, then proceed.**
