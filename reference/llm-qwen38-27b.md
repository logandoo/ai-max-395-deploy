# Phase 2 (family) — Qwen3.8-27B on Gufo（现役配方）

> **qwen35 arch / dense 27B 的家族配方。** 现役引擎 = **Gufo 容器引擎**
> （与 flashnext 同镜像不同 loader）。ROCmFP4/ovf 谱系（julianmb/q38rocm）已移出生产 ——
> 历史配方见本文件 §HISTORICAL。通用七步循环（model-deploy-generic.md）仍然管辖；
> 本文件只补家族特有面。

Read this file IN FULL before Phase 2. MUST/NEVER/STOP = enforced (SKILL.md §0).

## 2.1 Engine and model（现役）

- 引擎：`ghcr.nju.edu.cn/gufo-org/toolboxes/gufo-runtime:latest`（podman rootless，
  `gufo serve llm`）。容器自带 HIP runtime，宿主 ROCm 版本无关。
- 权重（ModelScope 设备直下，sha256 逐文件对官方清单核验）：
  | 文件 | 大小 | sha256（头 16） | 用途 |
  |---|---|---|---|
  | `Qwen3.8-27B-UD-Q4_K_XL.gguf` | 16.35 GiB | `3f227079003add25…` | 目标权重 |
  | `mmproj-BF16.gguf` | 0.87 GiB | `83ee4f4f205fa514…` | 视觉侧车（lazy upload） |
  | `Qwen3.8-27B-DFlash2-Q4_K_M.gguf` | 1.06 GiB | `1a25c56858e1ebe9…` | DFlash2 草稿（Q4/Q8 目标通用） |
  仓库：`unsloth/Qwen3.8-27B-GGUF` + `z-lab/Qwen3.8-27B-DFlash2-GGUF`。
  gufo 官方生产目标 = **UD-Q4_K_XL / UD-Q8_K_XL** 两个（其余 UD 混合量化不保证支持
  —— IQ 族有 `unsupported format` 拒载先例）。

## 2.2 ⛔ 宿主 RAM 硬约束（本家族第一坑，measured）

**gufo 的 qwen 密集家族 loader 会把整个权重文件拷入宿主匿名内存**（`mmap(MAP_ANONYMOUS)`
+ `hipHostRegisterMapped` 零拷贝映射给 GPU；源码 `src/models/qwen/hip/model_loader.cpp`
`MapRegisteredRegion`）→ **宿主 RAM 峰值 ≈ 权重文件大小**。

| 量化 | 权重 | 需要宿主暂存 | BIOS 96G 划分机（OS 见 ~30G） |
|---|---|---|---|
| UD-Q4_K_XL | 16.35 GiB | ~18.2G（实测 rss） | ✅ 可载（余量 ~10G） |
| UD-Q8_K_XL | 29.30 GiB | ~30G+ | ⛔ **OOM 击杀**（实测 3 次，死点 anon-rss 21.8G） |

- 对照：flashnext（qwen4exp）走**专属流式 loader**（分块 O_DIRECT 上传，111G 权重装载
  rss 仅 393MB）—— 同箱可行。瓶颈是 loader 设计 × OS 可见 RAM，不是机器总内存。
- **分片重切无效**：region 全量驻留（`CreateWeightRegions` 逐 region 驻留拷贝），
  拆成 N 片仍是总和驻留。
- 唯一缓解 = 让 OS 可见 ≥35G（改 BIOS GPU 划分 96→80 会让 flashnext 91G 常驻无法运行，
  等价于换机）。**部署前先算这本账**（S2 显存数学的宿主侧）。
- 症状识别：`journalctl -k` 只见 `Out of memory: Killed process (gufo) … anon-rss:…`，
  服务端日志只有 `load_phase phase=target_weights` 然后死 —— 无显式报错。

## 2.3 Reference launch parameters（config 驱动，勿他处硬编码）

```sh
gufo serve llm -m /models/Qwen3.8-27B-UD-Q4_K_XL.gguf \
  --mmproj /models/mmproj-BF16.gguf \
  --speculative dflash2 --dflash-model /models/Qwen3.8-27B-DFlash2-Q4_K_M.gguf \
  --served-model-name qwen3.8-27b \
  --think off \
  -j 2 -c 262144 -i 0.0.0.0 -p 8000 --log-progress
```

- `--served-model-name` 必须等于客户端发送的 model 字符串（gufo 拒绝未知名 → 404
  `model_not_found`；G2G.10）。
- `--think off`：服务端默认关思考（旧 `--reasoning off` 语音管线先例）；客户端可用
  `reasoning_effort` per-request 覆盖。
- `-j 2`：双会话（旧 `-np 2` 语音链路并发结论：聚合吞吐 +33%）。Q4@262K 实测
  **GPU 39.7G / 宿主 rss 18.2G / 载入 5.7s**（`load_completed` 日志）。
- DFlash2（`--speculative dflash2`，adaptive 默认）：官方 Q4 基准 656 t/s pp ·
  70.56 t/s tg 单用户 · 123 t/s 8 并发聚合（gfx1151）。
- **实测验收基线**：200K needle `prompt_tokens=219406 / wall=696.8s /
  pp_eff=314.9 t/s / SCORE 3/3`（10/50/90% 三深度逐字召回，nonce 前缀 fresh prefill）；
  load_completed 5.7s（Q4_K_XL）；算术抽查 17×23=391 正确。
- 视觉：`--mmproj` 挂载后 `/v1/models` 上报 `input_modalities:["text","image"]`，
  `image_url` base64/https 均可（20MiB/16 图/15s 预算）。

## 2.4 Service（q38-gufo.service，容器模式）

- S6.5 容器 unit 模板：podman run + `--userns=keep-id` + `label=disable` + kfd/dri 直通
  + `memlock=-1`；`loginctl enable-linger <USER>` 必须在。
- `KillSignal=SIGINT`（podman 转发进容器）+ `TimeoutStopSec=180` + ExecStartPre VRAM
  排空等待（与 flashnext 互斥换载，卸载后 BO 回收异步）。
- 与 flashnext **互斥于 :8000**（96G 池容不下两个完整 LLM）：面板 offload→load 换载，
  VRAM 守卫强制（409 VRAM_INSUFFICIENT）。
- 面板注册：`id=q38-gufo`，`kind=llama`，`model_glob="Qwen3.8-27B-UD-*.gguf"`
  （排除 DFlash2/mmproj 侧车）；helper 分派行 `set-llama-model`：
  目录 `/data/q38gufo/models` → unit `q38-gufo.service`，容器路径 `/models`。

## 2.5 Pitfalls（本家族，measured）

- ⛔ **宿主暂存 OOM**（§2.2）——选量化前先算 OS 可见 RAM 账。
- gufo 无 `/completion` 端点（历史家族坑同源）：验收走 `/v1/chat/completions`。
- 载入窗极短（Q4 5.7s）但**首请求**仍要等 `/v1/models` 就绪（TCP 通 ≠ 模型好；
  llmkube 教训同源）。
- `--think off` 服务端默认下输出短而直接 —— 不是"模型变笨"；要推理用 per-request
  `reasoning_effort`。
- 旧 ROCmFP4 权重/引擎文件名族（`Qwen3.8-27B-ROCmFP4-FAST.gguf`、`engine-overflow`）
  与 gufo 无关；恢复 = 另行下载 gufo 目标文件，不要混用两谱系的侧车/参数。

## 2.6 Phase Gate（家族）— 对通用 2G 补充

| # | 检查 | 期望 |
|---|---|---|
| G27.1 | 宿主 RAM 账 | 权重文件 < OS 可用 −6G（§2.2） |
| G27.2 | `load_completed` 日志 | `rss_mib` ≈ 权重 GiB×1.1；`gpu_device_used_mib` ≤ 声明 vram_gb×1024 |
| G27.3 | `/v1/models` | `id=qwen3.8-27b`、`context_length=262144`、模态含 image（mmproj 生效） |
| G27.4 | chat 算术/事实抽查 | 正确可读（temperature 0） |
| G27.5 | 200K needle（nonce 前缀，fresh） | 3/3 recall（10/50/90% 深度） |
| G27.6 | 与 flashnext 换载往返 | 双向 green，面板守卫拒绝面 409 实证 |

---

## HISTORICAL — ROCmFP4 / q38rocm 谱系

> 保留作史料与教训。该谱系的 unit/rollback 归档不再随包保留；恢复需按下列配方
> 重建引擎与 unit：14.5GB GGUF（sha `fb89c78d…`）经 Mac 中转再下载（CN 无直连 HF）
> → `q38-llama.service`（与 :8000 常驻互斥，先停常驻）。

- 引擎双轨：`stock` prebuilt v1.5.2（128K 日用）/ `ovf` engine-overflow 源码重建
  （**262K 生产必需**）；验收探针 `llama-server --list-devices` 必须同时列出 ROCm0+Vulkan0。
- 参数面（历史）：`-dev ROCm0 -ngl 999 -c 262144 -np 2 -t 16 --no-mmap
  --cont-batching --kv-unified -ctk q8_0 -ctv turbo4 --spec-type draft-mtp
  --spec-draft-n-max 4 --reasoning off`；TurboQuant KV + 混合注意力 → 262K ≈ 27G VRAM。
- 基线（历史）：prefill 365→283 t/s（1K→32K）、decode 32.3→25.2 t/s、MTP 接受率
  0.65-0.77、needle 3/3；历史审计含 32K cache 重放 0.14s。
- 坑（历史）：YaRN>262K 检索 MISS；`llama-server` 双二进制同名须 `/proc/<pid>/cmdline`
  验身份；SELinux `unlabeled_t` → `chcon -t bin_t`；`--reasoning off` 属该家族旗标。
