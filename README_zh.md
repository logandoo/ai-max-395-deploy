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
  × ROCm 版本）、获取、构建、服务注册、验收。
- 实测家族配方：Qwen3.8-Flash-Next（qwen4exp）与 Qwen3.8-27B（qwen35，
  ROCmFP4 谱系）。
- 投机解码（MTP draft head 加 ngram-mod）、视觉（mmproj），以及对照官方
  模型卡验证的思考模式契约：`enable_thinking`、`reasoning_effort`、
  `preserve_thinking`。
- 运维纪律：systemd 生命周期、下载路由（ModelScope 直下 / Mac 中转 /
  ghfast）、SELinux 处理、offload 与回滚（含归档）。
- 基准口径规则，杜绝已知假读数：prefill 只认 pp4096 口径、decode 测深度、
  投机解码先读输出再信计数器。
- 非 LLM 家族配方：ComfyUI 视觉栈（ERNIE-Image / Krea 2，fp8 on gfx1151）
  与 CPU OCR 管线（MinerU、PaddleOCR）；ASR（FunASR、SenseVoice）与 TTS
  （IndexTTS-2.5）见家族分册。

## 支持模型一览

| 模型 | 家族分册 | 验证 |
|---|---|---|
| Qwen3.8-Flash-Next（qwen4exp，125B-A6B，视觉，262k ctx） | `llm-flashnext-flashnext.md` | 端到端验证 |
| Qwen3.8-27B（qwen35，ROCmFP4 谱系） | `llm-qwen38-27b.md` | 验证（引擎换装实证通过） |
| 任意其他 GGUF LLM | `model-deploy-generic.md` | 程序级支持，引擎按架构定（S3） |
| ERNIE-Image-Turbo / ERNIE-Image / Krea 2（ComfyUI） | `vision-ocr-families.md` | 验证 |
| MinerU 3.4.5 / PaddleOCR 3.7.0（CPU 文档解析） | `vision-ocr-families.md` | 验证 |
| FunASR paraformer / SenseVoiceSmall / CAM++ / 2pass | `asr-funasr.md` | 验证 |
| IndexTTS-2.5（GPU，TheRock torch） | `tts-indextts.md` | 验证 |
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
| `reference/model-deploy-generic.md` | 七步环、引擎×ROCm 决策矩阵、Phase Gate 2G |
| `reference/vibeweaver-binding.md` | harness 检测与路由；S-step 到 vibeweaver 的映射 |
| `reference/system-base.md` | 发行版矩阵、ROCm 安装路径、内核参数、传输坑、Phase Gate 1 |
| `reference/llm-flashnext-flashnext.md` | Qwen3.8-Flash-Next 实例：显存账本、死路清单、基线、思考契约 |
| `reference/llm-qwen38-27b.md` | Qwen3.8-27B / ROCmFP4 谱系配方（qwen35 家族） |
| `reference/gpu-discipline.md` | #6012 eviction 防线、GPU 进程预算、故障处置 |
| `reference/testing-and-baselines.md` | 测试分级、基准口径规则、基线与阈值、Release Gate |
| `reference/asr-funasr.md`、`reference/tts-indextts.md` | ASR / TTS 家族配方 |

## 模型选择与实测性能

| 模型 | 选型 | 引擎 | Prefill（pp4096） | Decode（短上下文） | 最大上下文 |
|---|---|---|---|---|---|
| Qwen3.8-Flash-Next | UD-IQ4_XS（93.7GB） | EngramHalo.cpp `strix-halo-qwen4exp` + ROCm 7.14 core | 477.3 t/s | 28.7-42.1 t/s（MTP） | 262,144（MTP） |
| Qwen3.8-27B | ROCmFP4-FAST（14.5GB） | q38rocm ROCmFPX（stock/ovf）+ ROCm 7.2.3 | ~365 t/s（1k-prompt 口径） | 32.3 t/s（MTP n4） | 262,144 |

两家族运行于同一台 Strix Halo 机器的互斥驻留槽位。表中 Decode 为**基准条件**
（短上下文、`n_predict=256`、ngram 友好内容、MTP 接受率 79-93%）：散文贴近低端、
代码/改写贴近高端。生产长文生成明显更低属正常——新颖散文 + 2k+ token 输出在
25-62k 有效上下文下约 **14.8-18 t/s**；ngram 命中内容（复读/代码/改写）26-49 t/s。
三因子归因见 `reference/testing-and-baselines.md` §6.2a。

## 验证状态

- Fedora 44 Server：端到端验证，包内全部命令均在 Strix Halo 硬件上执行。
- Ubuntu 与 Arch：程序投射，未做硬件验证。替换矩阵见
  `reference/system-base.md` §1.0。部署时逐步验证 substituted 步骤，并将
  差异记录回该文件。
- 边界：LLM 仅 llama.cpp（不含 vLLM/SGLang 路径）；原生上下文之外的 YaRN
  排除（实测检索 miss）；NPU 路径不在覆盖范围；ASR/TTS/视觉/OCR 家族沿用
  各自运行时（见上表分册）。

## 许可

MIT——见 [LICENSE](LICENSE)。上游引擎沿用各自许可（llama.cpp 为
Apache-2.0，模型权重为 Qwen Community License）。
