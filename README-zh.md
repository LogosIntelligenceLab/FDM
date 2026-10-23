<div align="center">

<h1>Forward Dynamics Models</h1>

**论文《Forward Dynamics Models》（FDM）的官方实现。**

[📄 论文](https://arxiv.org/abs/2610.13117) · [🌐 项目主页](https://logosintelligencelab.github.io/FDM/) · [English](README.md)

[概述](#overview) · [论文结果](#results-reported-in-the-paper) · [安装](#installation) · [训练](#training) · [评测](#evaluation-and-inference)

</div>

<a id="overview"></a>

## ✨ 概述

FDM 通过**从动作到未来的学习（action-to-future learning）**来学习机器人策略。训练时，生成专家读取动作专家的特征，预测未来观测。未来预测损失沿这些特征反向传播，让策略学习动作的后果。动作预测不能读取未来 token，包括通过上下文形成的间接路径。

部署时，策略根据当前观测和任务指令预测动作，跳过未来生成，同时保留动作专家所需的上下文处理。

<p align="center">
  <img src="docs/images/pipeline.png" alt="FDM 流程：上下文专家和动作专家预测动作，仅在训练时使用的生成专家通过动作特征提供未来监督。" width="100%">
</p>

<p align="center"><em>训练时，未来监督塑造动作特征；推理时，仅使用上下文专家和动作专家。</em></p>

同一原理支持两类骨干模型：

| 论文模型 | 框架名称 | 骨干模型 | 未来监督 |
| --- | --- | --- | --- |
| FDM w/ VLM | `Pi05Causal` | π₀.₅ / PaliGemma | 直接回归未来图像的 Wan VAE 潜变量 |
| FDM w/ VGM | `WanMoTCausal` | Wan2.2-TI2V-5B | 对未来视频潜变量进行流匹配 |

本实现使用 `FDM` Python 包。仓库还包含用于相关工作流的其他 Pi05 和 WanMoT 变体；上表中的两个因果框架实现了 FDM。

<a id="results-reported-in-the-paper"></a>

## 🏆 论文报告的结果

以下成功率（%）摘自论文，并非对本次源码发布重新测得的结果。

| 模型 | LIBERO Spatial | Object | Goal | Long | 平均 |
| --- | ---: | ---: | ---: | ---: | ---: |
| FDM w/ VLM | 99.2 | 100.0 | 99.8 | 98.6 | **99.4** |
| FDM w/ VGM | 98.8 | 100.0 | 99.2 | 94.2 | **98.1** |

| 模型 | RoboTwin 2.0 Clean | Randomized | 平均 |
| --- | ---: | ---: | ---: |
| FDM w/ VLM | 81.54 | 80.86 | **81.20** |
| FDM w/ VGM | 88.84 | 89.62 | **89.23** |

真实机器人实验使用双臂 ARX-X5，覆盖五项操作任务。论文报告，相较于参与比较的 VLA 和 WAM 方法，成功率的绝对提升最高为 13 个百分点。训练使用 8 张 NVIDIA H100 GPU，推理使用 1 张 H100。

<a id="installation"></a>

## 🛠️ 安装

在仓库根目录执行命令。本文档使用的环境为 Linux、Python 3.10、PyTorch 2.7.0 和 CUDA 12.8。

```bash
git clone https://github.com/LogosIntelligenceLab/FDM.git
cd FDM
conda create -n fdm python=3.10 -y
conda activate fdm
pip install torch==2.7.0 torchvision==0.22.0 torchaudio==2.7.0 \
  --index-url https://download.pytorch.org/whl/cu128
pip install flash-attn --no-build-isolation --no-cache-dir
pip install -r requirements.txt
pip install -e .
```

按照 [INSTALL.md](docs/INSTALL.md) 中的源码依赖说明，安装提交版本为 `d602e8169cbad9e93a4a3b3ee1dd8b332af7ebf8` 的 LeRobot，并编译 decord。请将外部仓库克隆到 FDM 仓库旁的目录中。依赖列表也包含真实机器人部署所需的软件包；只有对应的部署流程才需要硬件驱动。部分沿用的指南涉及本次发布之外的可选环境和本地测试。

<a id="data-and-pretrained-weights"></a>

## 📦 数据与预训练权重

本次发布包含源码和配置模板，不包含数据集、训练后的 FDM checkpoint、文本缓存和实验日志，也尚未在此提供公开的 FDM checkpoint 下载链接。

**数据集。** 训练读取 LeRobot 数据集。请在相应的数据注册文件中配置数据集根目录、相机键名、动作维度和归一化方式：

- [LIBERO 数据注册](examples/LIBERO/train_files/data_registry/data_config.py)
- [RoboTwin 数据注册](examples/Robotwin/train_files/data_registry/data_config.py)
- [ARX 数据注册](examples/ARX/train_files/data_registry/data_config.py)

对于下方示例，`libero_all_multi` 要求预处理后的数据集位于 `playground/Datasets/LEROBOT_LIBERO_DATA`。实际的数据集根目录由注册文件定义；修改某个通用的根目录字段并不会替换所有已注册路径。

**FDM w/ VLM。** 准备兼容 RLinf/OpenPI_RLinf 的 π₀.₅ PyTorch checkpoint 目录（包含 `model.safetensors`）、PaliGemma tokenizer，以及 Wan2.2 的 48 通道 VAE。加载器也支持旧版 OpenPI PyTorch 权重布局。设置以下变量：

```bash
export PI05_MODEL_PATH=/path/to/pi05_base_pytorch
export PI05_TOKENIZER_PATH=/path/to/paligemma_tokenizer.model
export WAN_VAE_PATH=/path/to/Wan2.2_VAE.pth
```

**FDM w/ VGM。** 在 [fdm_wanmot_causal.yaml](FDM/config/training/vla/fdm_wanmot_causal.yaml) 中设置 DiT、VAE、UMT5 编码器和 tokenizer 的路径。默认目录布局使用 `playground/Pretrained_models/Wan2.2-TI2V-5B` 和 `playground/Pretrained_models/umt5-xxl`。训练默认使用缓存的文本嵌入：

```bash
export TEXT_EMBEDDING_CACHE_DIR=playground/cache/text_embeds/libero
PYTHONPATH=. python scripts/vla/precompute_text_embeds.py \
  --config_yaml FDM/config/training/vla/fdm_wanmot_causal.yaml \
  --datasets.vla_data.data_mix libero_all_multi
```

训练和推理服务应使用一致的编码器路径、提示词模板和缓存设置。若要使用随机初始化的动作专家启动新训练，请传入 `--framework.action_model.model_path null`；否则，基础 YAML 配置会指向一个未随本次发布提供的本地初始化 checkpoint。

<a id="training"></a>

## 🚀 训练

### 在 LIBERO 上训练 FDM w/ VLM

准备好数据，并设置上述三个权重与 tokenizer 路径变量后，运行：

```bash
NPROC_PER_NODE=8 bash examples/LIBERO/train_files/train_pi05-causal.sh \
  libero_all_multi fdm_vlm \
  --framework.model.num_images 2 \
  --trainer.resume_from_checkpoint null
```

该启动脚本使用 15 步动作预测时域，禁用本体感知状态输入，以 128 的全局批量大小训练 30,000 步。生成分支使用原始帧偏移量为 `[-15, 0, 15]` 的过去、当前和未来帧，并采用潜变量 MSE 目标。`Pi05Causal` 要求禁用 FAST 交叉熵、子任务训练和 RTC 训练延迟，所提供的配置已作相应设置。详见[实现细节](docs/Pi05_causal.md)。

### 在 LIBERO 上训练 FDM w/ VGM

准备好权重和文本缓存后，运行：

```bash
NPROC_PER_NODE=8 bash examples/LIBERO/train_files/train_wanmot-causal.sh \
  libero_all_multi fdm_vgm \
  --framework.action_model.model_path null \
  --datasets.vla_data.disable_state true \
  --trainer.resume_from_checkpoint null
```

基础训练配置使用 32 步动作预测时域、4 的未来帧采样步长，以及 128 的全局批量大小，共执行 22,000 个优化器更新步。请保持 `framework.video_model.config.video_attention_mask_mode=first_frame_causal`，确保动作可读取的当前观测特征无法读取未来 token。

### RoboTwin 与真实机器人

| 环境 | FDM w/ VLM 启动脚本 | FDM w/ VGM 启动脚本 |
| --- | --- | --- |
| RoboTwin | [train_pi05-causal.sh](examples/Robotwin/train_files/train_pi05-causal.sh) | [train_WanMoTCausal.sh](examples/Robotwin/train_files/train_WanMoTCausal.sh) |
| ARX | [train_pi05-causal.sh](examples/ARX/train_files/train_pi05-causal.sh) | [train_wanmot-causal.sh](examples/ARX/train_files/train_wanmot-causal.sh) |

启动脚本接受数据混合配置、可选的运行名称后缀，以及末尾追加的配置覆盖参数。`NPROC_PER_NODE` 控制每个节点的进程数。请根据可用硬件调整单设备批量大小和全局批量大小。输出默认保存到 `results/Checkpoints/vla`。

**复现说明。** 基础 YAML 和启动脚本是配置起点，部分默认值与论文不同。对于 RoboTwin，论文中的两个模型都禁用了本体感知状态输入；请传入 `--datasets.vla_data.disable_state true`，并将 VLM 的动作维度和状态维度均设为 14，使用三个相机视角。对于论文中的真实机器人设置，请使用 20 维双臂动作表示、禁用状态输入，并采用各任务独立的 Z-score 归一化；ARX 模板也支持其他动作布局。比较结果前，请核对数据集表示、相机布局、预测时域和保存的运行配置。重新随机初始化不一定与论文实验中保存的初始化完全一致。

<a id="evaluation-and-inference"></a>

## 🎯 评测与推理

两个 FDM 框架都通过 `predict_action` 提供仅动作推理，并通过 `predict_video` 提供可选的未来预测。Pi05Causal 的动作推理只需要当前帧，不调用 VAE。WanMoTCausal 保留视频骨干对当前观测的处理，但不生成未来视频。

按照 [LIBERO 指南](examples/LIBERO/README.md) 安装 LIBERO 及其评测依赖。在模型环境中启动策略服务器：

```bash
GPU_ID=0 PORT=6694 bash examples/LIBERO/eval_files/run_policy_server.sh \
  /path/to/run/final_model/pytorch_model.pt
```

在另一个终端中激活 LIBERO 环境，然后运行：

```bash
PORT=6694 bash examples/LIBERO/eval_files/eval_libero.sh \
  /path/to/run/final_model/pytorch_model.pt libero_spatial
```

对 `libero_object`、`libero_goal` 和 `libero_10` 重复评测。请将 checkpoint 对应的运行配置和 `dataset_statistics.json` 与 checkpoint 一起保存在预期的运行目录中。保存的配置中的路径必须能在运行推理服务的机器上正确访问。

默认 LIBERO 评测对每项任务执行 50 个 episode，使用随机种子 42、10 个去噪步，并在每次重新规划前执行 10 个动作。动作执行时域与模型预测时域不同。其他对应工作流见[评测协议](examples/eval_protocol.md)、[RoboTwin 指南](examples/Robotwin/README.md)和 [ARX 指南](examples/ARX/README.md)。

<a id="code-structure"></a>

## 🗂️ 代码结构

```text
FDM/model/framework/VLM4A/Pi05_causal.py   # FDM w/ VLM
FDM/model/modules/pi05/causal.py           # VLM 因果专家
FDM/model/framework/WM4A/WanMoT_causal.py  # FDM w/ VGM
FDM/model/modules/wan_mot/                 # 视频/动作专家与 MoT
FDM/config/training/vla/                   # 模型与训练配置
FDM/dataloader/                            # LeRobot 与视频数据流水线
FDM/training/                              # 分布式训练
examples/{LIBERO,Robotwin,ARX}/            # 数据注册与任务工作流
deployment/                                # 策略服务与机器人部署
```

<a id="citation"></a>

## 📖 引用

```bibtex
@misc{ma2026forward,
  title   = {Forward Dynamics Models},
  author  = {Ma, Zipei and Jiang, Junzhe and Zhang, Jiahui and Zhang, Juntong
             and Gu, Chun and Zhang, Bozhou and Deng, Jiankang and Zhu, Xiatian
             and Zhang, Li},
  year    = {2026},
  eprint  = {2610.13117},
  archivePrefix = {arXiv},
  primaryClass = {cs.RO},
  doi     = {10.48550/arXiv.2610.13117},
  url     = {https://arxiv.org/abs/2610.13117}
}
```

<a id="acknowledgments-and-license"></a>

## 🤝 致谢与许可证

FDM 基于 [starVLA](https://github.com/starVLA/starVLA) 构建，并使用了 OpenPI/RLinf、Wan、LeRobot、LIBERO 和 RoboTwin 的组件与工作流。感谢这些项目的作者和贡献者。[LICENSE](LICENSE) 标明了 FDM 作者和贡献者，并保留上游 starVLA 的版权声明；随仓库提供的第三方组件声明见 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。预训练权重和数据集分别遵循各自的许可证。
