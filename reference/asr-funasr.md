# Phase 3 — ASR: FunASR stack (paraformer / SenseVoice / CAM++ / 2pass)

Read this file IN FULL before Phase 3. MUST/NEVER/STOP = enforced (SKILL.md §0).

## 3.1 Runtime shape

- venv: `<base>/speech/venv-asr` (funasr 1.4.11) — **ASR is CPU-only**
  (GPU budget rule R5: do not put ASR on the GPU).
- Models (ModelScope direct, ~60MB/s): paraformer-contextual (hotwords),
  fsmn-vad, cam++ (speaker embedding), ct-punc-realtime,
  paraformer-online (streaming), SenseVoiceSmall.
- ⛔ SeACo and plain paraformer-large are **delisted on ModelScope (404)** →
  contextual paraformer is the hotword successor. Verify URLs before
  scripting downloads.
- Gateway: `asr_server.py` :8201 — `POST /transcribe` +
  `WS /ws/transcribe/stream`; the generic HTTP contract carries **float32**
  PCM (convert at the edge if the client speaks PCM16).
- Chatbot attach (current production state): `provider_type="general"`,
  `is_dashscope=false`, `base_url=http://<host>:8201`. ⛔ Honest scope
  note: the chatbot's **LLM leg is still cloud (deepseek)** — attaching it
  to the local llama-server is NOT done; do not assume otherwise.
- Meeting 2pass: FunASR main-branch Python runtime (vendored) on :10095,
  loopback CPU, plus a WS bridge adapter (ready-on-start;
  binary-before-start guard; is_end ack error passthrough; empty-segment
  filtering; fallback to the legacy path when the runtime is unreachable).
  Voiceprint: `POST /voiceprint/enroll` + `GET /voiceprint` with CAM++
  embeddings written atomically to a shared store.

## 3.2 The split pipeline — MUST NOT use the merged pipeline

⛔ funasr 1.4.11's merged pipeline (vad+spk attached) does not produce
`sentence_info` for contextual paraformer ("No timestamp found") →
**MUST** use the five-stage split pipeline:

```
m_vad(max_end_silence_time=700) → per-segment m_asr → m_spk embeddings
  → cosine clustering (threshold 0.6) → m_punc
```

- ⛔ fsmn-vad default `max_end_silence_time=8000` merges a whole
  conversation into one segment → **MUST** set 700 explicitly.
- ⛔ funasr's torch path rejects Short dtype ("normal_kernel_cpu not
  implemented") → feed **float32**.
- Speaker semantics: voiceprint matching is primary, clustering is
  fallback (ADR D-26).

## 3.3 SenseVoice switch (pitfalls)

- Lightweight loading installs only SenseVoice+fsmn-vad, but
  `/transcribe`'s diarization needs paraformer/CAM++/punc → **MUST**
  lazy-load legacy models (`_ensure_legacy_models()`) on first use.
- `onnxruntime` **MUST** be pinned to 1.20.1 (install from the aliyun
  mirror; newer ORT breaks the VAD path).
- ⛔ Silero v5 ONNX silently outputs garbage probabilities under ORT
  (no exception — max ~0.067 on real speech) → streaming VAD **MUST** be
  fsmn-vad, not Silero.
- ⛔ Dynamic silence threshold inflates with utterance length (END never
  fires on long speech) → use the fixed 700ms budget.
- Diplex/dupe guard: chatbot voice duplex only accepts dashscope/mimo
  modes — add an explicit `generic` branch; its WS supervisor **MUST** run
  as a background task (inline `await` deadlocks `_receive_loop` and audio
  is never drained).

## 3.4 2pass meeting runtime notes

- The runtime's sv/vocos paths only use plain-torch functions
  (`_hz_to_mel`/`_mel_to_hz`/fbank/Resample) — keep the torchaudio shim
  (see tts-indextts.md) compatible with that.
- Speaker DB is a shared library file (`speaker_db.json`) written
  atomically by the gateway.

## Phase Gate 3 — MUST pass before declaring ASR done

| # | Check | Expected |
|---|---|---|
| G3.1 | `systemctl is-active speech-asr` | active |
| G3.2 | `curl -s :8201/health` | `models_loaded=true` (or ready flag per contract) |
| G3.3 | POST /transcribe with a 16kHz Chinese wav | JSON fields complete: non-empty `text`, `language`, `timestamps[{start_time,end_time,text}]`, `segments[{speaker,speaker_confidence,...}]`, `speaker_mode="cam++"`, `duration>0` |
| G3.4 | WS stream with 2.5s trailing silence | final segment fires (VAD END) |
| G3.5 | 2pass long session (5min audio) | 3.3×realtime; partials+segments+2 speakers+final ~1145 chars; no 120s truncation |
| G3.6 | per-family contract suite (14+19 cases) | all green |

**If any check fails: STOP — diagnose (§3.2/§3.3 pitfalls), fix, re-run.**
