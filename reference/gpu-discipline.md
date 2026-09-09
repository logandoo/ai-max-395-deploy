# Phase 5 — GPU Context Discipline (#6012 eviction firewall)

Continuous discipline — applies from the moment the first GPU service
starts until the machine is decommissioned. MUST/NEVER/STOP = enforced.

## 5.1 The law (measured on this hardware)

- ROCm issue **#6012**: on gfx1151, **4+ concurrent GPU processes →
  "queue evicted" cascade**; 3 concurrent is stable.
- **Context budget (R5):** decide **coexist vs replace** BEFORE loading a new
  model (model-deploy-generic.md S1/S8). The resident set is **1 GPU
  process** (flashnext llama-server) — headroom for 2 more GPU procs
  by the budget, but each new resident model must first answer the
  coexist-or-replace question with the VRAM math (generic §S2).
- ⛔ **LLM decode and TTS synthesis must NEVER overlap on the GPU.**
  Measured overlap: LLM drops to 5 tok/s while TTS TTFA balloons
  3s→26s+ (RTF ~9). The application **MUST** serialize
  (`tts_after_llm=true`). Slot parallelism (-np 2) fixes queueing, not
  compute contention.
- Root-cause fix: kernel param `amdgpu.cwsr_enable=0`. It is written into
  the bootloader config but takes effect only on the NEXT reboot — until
  then the running kernel still has CWSR enabled (verify with
  `/proc/cmdline` before relying on it; apply via
  `grubby --update-kernel=ALL`, see system-base §1.1).
- MES firmware 0x91 is the good version (0x83 is broken); ROCm 7.2.3 is the
  stable line **for the q35/ROCmFP4 family** — qwen4exp loads require the
  7.14 core (`/opt/rocm/core-7.14`), see llm-flashnext-flashnext.md F2.

## 5.2 Failure playbook

- Symptom: `queue evicted` in logs → **restart the offending process**.
  ⛔ Do NOT just clear VRAM: after eviction the HSA AsyncEventsLoop can
  busy-lock at 100% CPU (#6522) — a process restart is the only clean
  recovery.
- ⛔ Teardown SEGV (ROCm 7.x, measured on every clean stop): the unit lands
  in **`failed`**, not `inactive`. This is terminal-stopped, not an incident
  — lifecycle code MUST accept `inactive|failed` as terminal-stopped.
  Load-order rule after any stop: **wait for VRAM to drain
  (<8GB) before the next start** (unit ExecStartPre does this; ad-hoc
  restarts must too — a 74GB BO reclaim is asynchronous and racing it
  crashes the allocator).
- After any maintenance: run the status script —
  **eviction must be 0; GPU proc count must equal the budget.**

## 5.3 Pre-flight for ANY new GPU process (MUST answer before launch)

1. Is the current GPU context count < budget (5.1)? If not: STOP.
2. Will it overlap with LLM or TTS? If yes: STOP (serialize first).
3. Is there a rollback (how do I kill exactly this process, by unit/pidfile)?
4. Did the VRAM drain after the previous stop?
5. Is the change worth risking a production eviction? (Experiments belong
   in a maintenance window.)
