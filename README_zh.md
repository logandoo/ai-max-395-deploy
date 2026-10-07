# ai-max-395-deploy

[English](README.md) | 简体中文

在 AMD Ryzen AI Max+ 395（Strix Halo）硬件上部署任意用户指定模型：七步
agent 流程、实测引擎配方与基线。

## 功能

面向编码 agent（opencode、DeepSeek harness、standalone）与 Strix Halo 机器
的运维者。包内每条规则与数字均在 Strix Halo 硬件（Fedora 44 Server，
128GB 统一内存，BIOS 划分 96GB GPU 池）上验证；坑册条目均对应实测失败
模式。

- 面向任意 GGUF 模型的七步部署环：survey、显存数学、引擎决策（按模型架构
  × 运行时）、获取、构建、服务注册、验收。
- 实测家族配方：Qwen3.8-Flash-Next（qwen4exp）与 Qwen3.8-27B（qwen35，
  ROCmFP4 谱系；历史参考）。Flash-Next 分册含完整的**引擎谱系**
  （EngramHalo → Nathanw → Gufo）与各引擎的实测选型依据。
- **容器化引擎部署（podman）**：rootless 存储落数据盘、SELinux
  `label=disable`、GPU 透传、`loginctl enable-linger`、systemd unit 范式、
  ghcr.nju.edu.cn 镜像中转——Gufo 栈（MIT、gfx1151 原生）为实例。
- 投机解码（MTP draft head；shared/普通/`-eh` 侧车为引擎专属、不可混用）、
  视觉（mmproj），以及对照官方模型卡验证的思考模式契约：`enable_thinking`、
  `reasoning_effort`、`preserve_thinking`。
- 运维纪律：systemd 生命周期、下载路由（ModelScope 直下 / Mac 中转 /
  ghfast / ghcr 镜像）、仓库 API 取 sha 清单、SELinux 处理、offload 与回滚
  （含归档）。
- 基准口径规则，杜绝已知假读数：prefill 只认 pp4096 口径（或 nonce-fresh
  25K）、decode 测深度、200K needle 为生产等价验收、投机解码先读输出再信
  计数器。
- 非 LLM 家族配方：ComfyUI 视觉栈（ERNIE-Image / Krea 2，fp8 on gfx1151）
  与 CPU OCR 管线（MinerU、PaddleOCR）；ASR（FunASR、SenseVoice）与 TTS
  （IndexTTS-2.5）见家族分册。

## 支持模型一览

| 模型 | 家族分册 | 验证 |
|---|---|---|
| Qwen3.8-Flash-Next（qwen4exp，125B-A6B，视觉，262k ctx） | `llm-flashnext-flashnext.md` | 端到端验证（当前引擎：**Gufo**，podman；引擎谱系见分册） |
| Qwen3.8-27B（qwen35 密集） | `llm-qwen38-27b.md` | 端到端验证（现役引擎：**Gufo**，UD-Q4_K_XL + DFlash2；宿主 RAM 暂存约束见分册；ROCmFP4 谱系=HISTORICAL 附录） |
| Qwen3-ASR 1.7B + Qwen3-TTS 1.7B（CustomVoice） | `speech-qwen3-gufo.md` | 验证（Gufo BF16 safetensors；TTS→ASR round-trip 逐字命中） |
| 任意其他 GGUF LLM | `model-deploy-generic.md` | 程序级支持，引擎按架构定（S3） |
| ERNIE-Image-Turbo / ERNIE-Image / Krea 2（ComfyUI） | `vision-ocr-families.md` | 验证 |
| MinerU 3.4.5 / PaddleOCR 3.7.0（CPU 文档解析） | `vision-ocr-families.md` | 验证 |
| FunASR paraformer / SenseVoiceSmall / CAM++ / 2pass | `asr-funasr.md` | 验证（HISTORICAL——2026-09 起 disabled） |
| IndexTTS-2.5（GPU，TheRock torch） | `tts-indextts.md` | 验证（HISTORICAL——2026-09 起 disabled） |
| qwen3.5-0.8B on NPU（FLM/lemonade） | `llm-qwen38-27b.md` §2.4 | 非支持路径 |

## 安装

本包为一个 markdown 文件目录；无构建步骤。

- opencode：将该目录加入 `opencode.json` 的 `skills.paths`，或复制到
  `~/.config/opencode/skills/ai-max-395-deploy/`。
- DeepSeek harness：经其插件/skill 注册表注册，`vibeweaver-dsh`（如安装）
  即可绑定。
- Standalone agent：直接读取文件，无需注册。

## 使用方法

Agent 按序执行：

1. 读 `SKILL.md`：硬件事实、规则 R1-R15、指向各分册的 phase map。
2. 判定 agent 栈绑定（SKILL.md §0.1）：装有 vibeweaver 则由其工作流治理，
   本包提供领域内容；未装则按本包的 loop 与 Phase Gate 独立执行。
3. 按 `reference/model-deploy-generic.md` 执行七步环：S1 survey、S2 显存
   数学、S3 引擎决策、S4 获取、S5 构建、S6 服务、S7 验收、S8 offload 或
   换装。

## 文件结构

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 路由：硬件模型、规则 R1-R15、phase map、坑册索引 |
| `reference/model-deploy-generic.md` | 七步环、引擎×运行时决策矩阵、容器引擎路径、Phase Gate 2G |
| `reference/vibeweaver-binding.md` | harness 检测与路由；S-step 到 vibeweaver 的映射 |
| `reference/system-base.md` | 发行版矩阵、ROCm 安装路径（7.14 core=历史）、内核参数、传输坑、Phase Gate 1 |
| `reference/llm-flashnext-flashnext.md` | Qwen3.8-Flash-Next 实例：引擎谱系（EngramHalo→Nathanw→Gufo）、Gufo/podman 配方、坑册、基线、思考契约 |
| `reference/llm-qwen38-27b.md` | Qwen3.8-27B on Gufo 实例：UD-Q4_K_XL + DFlash2、宿主 RAM 暂存约束、坑册、Phase Gate 27（ROCmFP4 谱系=HISTORICAL 附录） |
| `reference/speech-qwen3-gufo.md` | Qwen3-ASR + Qwen3-TTS（Gufo）家族配方：变体、API 契约、codec 坑、Phase Gate 3/4 |
| `reference/vision-ocr-families.md` | ComfyUI（ERNIE-Image / Krea 2 / Qwen-Image-2.1）与 MinerU / PaddleOCR CPU 管线 |
| `reference/load-unload-panel.md` | model-panel :8300 加载/卸载面：注册表、API、显存守卫、加载窗纪律 |
| `reference/gpu-discipline.md` | #6012 eviction 防线、GPU 进程预算、故障处置 |
| `reference/testing-and-baselines.md` | 测试分级、基准口径规则、基线与阈值、Release Gate |
| `reference/asr-funasr.md`、`reference/tts-indextts.md` | ASR / TTS 家族配方 |

## 模型选择与实测性能

| 模型 | 选型 | 引擎 | Prefill | Decode（MTP） | 最大上下文 |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next | UD-Q4_K_XL（111.3GB） | **Gufo**（MIT，podman，gfx1151 原生） | **~1436 t/s**（新鲜 25K）；219K 请求均速 1317 t/s | 25-45 t/s（实测带） | 262,144 |
| Qwen3.8-27B | UD-Q4_K_XL（16.35GiB）+ DFlash2 Q4_K_M | **Gufo**（MIT，podman，gfx1151 原生） | **314.9 t/s pp_eff**（219K needle 实测）· 官方 Q4 656 t/s pp（C1） | 70.56 t/s tg（DFlash2，官方） | 262,144 |
| Qwen3.8-27B（HISTORICAL） | ROCmFP4-FAST（14.5GB） | q38rocm ROCmFPX（stock/ovf）+ ROCm 7.2.3 | ~365 t/s（1k-prompt 口径） | 32.3 t/s（n4） | 262,144 |

家族分册按**引擎谱系**组织：同一模型给出多个已验证引擎（当前推荐、零构建
回退、历史对照），各自附实测画像与适用场景，任意一点都留有可执行的恢复
路径。表中数字均为**基准条件**口径（固定提示词、filler/复读内容、高 MTP
接受率）：新颖散文等真实负载读数更低属正常——三因子归因见
`reference/testing-and-baselines.md` §6.2a。

## 验证状态

- Fedora 44 Server：端到端验证，包内全部命令均在 Strix Halo 硬件上执行。
- Ubuntu 与 Arch：程序投射，未做硬件验证。替换矩阵见
  `reference/system-base.md` §1.0。部署时逐步验证 substituted 步骤，并将
  差异记录回该文件。
- 边界：LLM 引擎 = llama.cpp 系 fork + Gufo（不含 vLLM/SGLang 路径）；
  原生上下文之外的 YaRN 排除（实测检索 miss）；NPU 路径不在覆盖范围；
  ASR/TTS/视觉/OCR 家族沿用各自运行时（见上表分册）。

## 许可

MIT——见 [LICENSE](LICENSE)。上游引擎沿用各自许可（llama.cpp 为
Apache-2.0，Gufo 为 MIT，模型权重为 Qwen Community License）。
