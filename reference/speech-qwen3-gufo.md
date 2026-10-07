# Phase 3/4 (family) — Qwen3-ASR + Qwen3-TTS via Gufo（现役语音栈）

替换 FunASR/IndexTTS 语音族（`asr-funasr.md` / `tts-indextts.md`
为 HISTORICAL 参考）。Gufo 的 ASR/TTS 是 **BF16 safetensors 原生路径**（audio.cpp 系），
不是 GGUF —— `gufo serve asr|tts -m <模型目录>`。0.6B、25Hz、GGUF 检查点均不支持。

## Q1. 组件与端点

| 服务 | unit | 端口 | 模型目录 | served-model-name |
|---|---|---|---|---|
| ASR | `qwen3-asr.service` | 8211 | `/data/speech-gufo/models/Qwen3-ASR-1.7B` | `qwen3-asr-1.7b` |
| TTS | `qwen3-tts.service` | 8212 | `/data/speech-gufo/models/Qwen3-TTS-12Hz-1.7B-CustomVoice` | `qwen3-tts-12hz-1.7b-customvoice` |

- 引擎：`ghcr.nju.edu.cn/gufo-org/toolboxes/gufo-runtime:latest`（podman rootless，与
  flashnext 同镜像同 GPU 直通面：`--userns=keep-id --device /dev/kfd --device /dev/dri
  --group-add keep-groups --ulimit memlock=-1 --security-opt label=disable`）。
- 权重获取（CN）：**ModelScope 设备直下**（⛔ 设备侧 hf-mirror 实测 0 速，S4 纪律）：
  - `Qwen/Qwen3-ASR-1.7B` —— 13 文件 4.70G（含 `model-0000{1,2}-of-00002.safetensors`）
  - `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` —— 15 文件 4.52G，**含 `speech_tokenizer/`
    子目录（682M codec，缺它 TTS 起不来 —— 下载器必须 Recursive 全量）**
- 实测显存：ASR **5.13G** / TTS **6.21G**（双载 11.34G）；vram_gb 声明 6 / 7（面板守卫按此计）。

## Q2. TTS 变体（Checkpoint 决定 API 形态）

| 变体 | HTTP model ID | 条件 |
|---|---|---|
| `CustomVoice`（现役） | `qwen3-tts-12hz-1.7b-customvoice` | `voice` = 内置音色名（9 个：aiden/dylan/eric/ono_anna/ryan/serena/sohee/uncle_fu/vivian） |
| `Base` | `qwen3-tts-12hz-1.7b-base` | 参考音频+文本（ICL 克隆），`--voice NAME=WAV` 注册常用克隆 |
| `VoiceDesign` | `qwen3-tts-12hz-1.7b-voice-design` | `instructions` 文字描述设计音色 |

变更变体 = 换模型目录 + served-model-name，独立 unit/端口可并行第二实例。

## Q3. API 契约（验收口径）

```sh
# ASR：multipart 转写（json|text|verbose_json；可选 language/prompt/stream）
curl -sS -X POST http://<host>:8211/v1/audio/transcriptions \
  -F file=@speech.wav -F model=qwen3-asr-1.7b -F response_format=json
# → {"text":"The quick brown fox jumps over the lazy dog."}   （实测逐字命中 TTS 输入）

# ASR 流式（上传文件 SSE）：-F stream=true -F response_format=json
# ASR 麦克风 WS：/v1/realtime?intent=transcription（PCM16 @24kHz；session.update /
#   input_audio_buffer.append/commit 协议；每 buffer ≤120s，commit ≥100ms）

# TTS：JSON → WAV（RIFF 24kHz；response_format=pcm 流式 s16le mono）
curl -sS -X POST http://<host>:8212/v1/audio/speech \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-tts-12hz-1.7b-customvoice","voice":"ryan","input":"Hello from Gufo."}' \
  --output speech.wav

curl -sS http://<host>:8212/v1/audio/voices   # 内置音色列表
# TTS WS：/v1/audio/speech/stream（vLLM-Omni 增量文本协议：session.config/input.text/
#   audio.start → binary PCM → audio.done；session.done/session.close）
```

- ASR 输入：PCM16/24/32 或 float32 WAV，≤8 声道 192kHz（前端混单声道+重采样 16k），
  录音 ≤30 分钟；长录音按低能量边界分块转写再合并，`max_tokens` 是**每块**上限。
  unit 已置 `--max-request-bytes 268435456`（256M，对齐 CLI 上限）。
- TTS 采样默认：top-k 50 / top-p 1 / temperature 0.9（talker+predictor）；`greedy:true`
  双关；`seed` 请求级回放；贪心可能不到 EOS，诊断请求配 `max_new_tokens`。
- 基准（gufo 官方，gfx1151）：ASR **15.27× 实时**；TTS **2.54× 实时**、TTFA 201ms(CustomVoice)。

## Q4. 显存纪律（R5 引申）

- **LLM 与 TTS 禁止并发推理**（实测踩踏：LLM 5 tok/s + TTS TTFA 3s→26s+）。
  weave-talk 用 `tts_after_llm=true` 强制串行 —— 应用层契约，服务层不做调度。
- flashnext 常驻 91G 时池余 ~5G：ASR(6)/TTS(7) 声明值下**双语音与 LLM 三者不可同载**
  （面板守卫 409 强制）；二选一共存或先 offload LLM。
- GPU 进程预算 ≤3（#6012）：flashnext+ASR+TTS = 3 席正好满额。
- unit ExecStartPre 为「排空或稳定」变体（等池 <8G **或** 显存 3s 不变）：小模型与常驻
  LLM 共存时不再空等 120s，卸载→快速重载的 BO 回收竞态防护保留。

## Q5. Phase Gate 3/4（语音）— 全绿才算部署完成

| # | 检查 | 期望 |
|---|---|---|
| G3.1 | `systemctl is-active qwen3-asr qwen3-tts` | active（面板 load 后） |
| G3.2 | `GET /v1/audio/voices` | 9 音色（CustomVoice） |
| G3.3 | `POST /v1/audio/speech` 输出 | RIFF/WAVE 头 + 字节数>100k（3s+ 语音） |
| G3.4 | `POST /v1/audio/transcriptions` round-trip | 与 TTS 输入文本逐字一致 |
| G3.5 | 显存账 | 实测 ≤ 声明 vram_gb +0.5G；双载 ≤ gpu 池 |
| G3.6 | 面板注册 | `/api/state` 两 id 在册、load/unload 往返绿 |
| G3.7 | sha256 | 逐文件 vs ModelScope 清单 N/N（Recursive 全量含 codec） |

## P. 坑册（measured）

- **TTS codec 子目录**：`speech_tokenizer/model.safetensors` 682M 必须在位（`Recursive=true`
  清单核对），缺文件=起服即死（S9「下载脚本不完整」族）。
- **⛔ 设备侧 hf-mirror 0 速**（实测）：语音/27B 全走 ModelScope 设备直下（38-50MB/s）。
- gufo 拒绝未知 model 名（`model_not_found`）：客户端发送的 model 字符串必须等于
  `--served-model-name`（G2G.10 同源）。
- 旧语音栈（FunASR/IndexTTS，8201/8202）注册位仍留档：端口段刻意分开（8211/8212），
  两栈并存不冲突；合并须先移除旧 unit+注册位。
- ASR 每请求两次 CPU 前端可并发；设备侧 FIFO 准入串行处理音频块，取消会清队列。
- TTS 贪心生成不到 EOS 的"跑飞"输出：请求级 `max_new_tokens` 兜底（QUALITY 已知限制）。
