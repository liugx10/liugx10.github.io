---
layout: single
title: "AI-ISP 十二周学习计划：从成像物理到端侧部署"
date: 2026-10-08
categories: 技术
tags: ["ISP", "RAW", "图像处理", "学习计划"]
toc: true
toc_sticky: true
excerpt: "一份按真实工作链排序的 AI-ISP 学习计划：从传感器成像物理和手写 ISP 起步，经过 RAW-to-RGB 训练、3A 调优、多帧与生成式，最后落到量化压缩与 TensorRT 部署。12 周，每周 15–20 小时。"
---

学习 AI-ISP 最容易走偏的地方，是按「模型难度」排序去学——先啃一遍扩散模型，回头却说不清黑电平为什么按通道标定、RAW 域到底该在链路哪一步降噪。这份计划换一个排序依据：**按真实的工作链来**。

成像物理 → 传统 ISP 链路 → 数据与工具链 → 深度学习 RAW-to-RGB → 3A 与调优 → 多帧 / 视频 / 生成式进阶 → 端侧工程化 → 作品集输出。12 周，每周 15–20 小时。

两点前提说在前面：

- **硬件**：计划假设 24G 显存 + 64G 内存（例如 4090 + 64G），这个配置足够跑 RAW-to-RGB 级别的训练与部署验证。短板只在数据集磁盘空间——ZRR、MAI2022 这类数据集加起来体量可观，**建议预留 2–4TB NVMe**。
- **节奏**：15–20 小时/周是业余推进的强度。脱产学习可以压缩到 6–8 周，压缩的是等待训练的时间，不是动手的部分。

## 一、三条主线与验收标准

三条主线对应三种能力，缺一条都会在面试或落地上露馅。

| 主线 | 内容 | 三个月后的验收标准 |
|---|---|---|
| 物理与传统链路 | 传感器成像模型、噪声模型、Bayer/Quad Bayer、传统 ISP 各模块 | 能用 NumPy/C++ 手写一条可运行的 mini-ISP，输出与 rawpy/libraw 结果可比对 |
| 算法与训练 | RAW→sRGB 端到端/模块化网络、损失设计、数据构造 | 在公开数据集上复现一个 baseline 并超过它 0.5dB 以上，能解释每一处改动 |
| 工程与落地 | 量化/蒸馏/剪枝、ONNX→TensorRT、延迟与内存预算、指标评测 | 交付一个可部署模型 + 一份含 PSNR/SSIM/LPIPS/ΔE/延迟的评测报告 |

## 二、第 1 个月（W1–W4）：把「RAW 到图像」这条链路摸透

### W1 成像物理与传感器

**内容**：光电转换、量子效率、满阱容量、光子转移曲线（PTC）、散粒噪声与读出噪声、SNR 与 ISO 的关系、黑电平、PRNU/DSNU、微透镜与串扰、CFA（Bayer / 四合一方阵 / RGBW）、HDR 传感器模式（DOL-HDR、交错曝光）、卷帘快门。

**动手**：用 rawpy 读 20 张 DNG，打印 `black_level_per_channel`、`white_level`、`raw_pattern`、`color_matrix`、`cam_xyz_matrix` 以及 EXIF 里的 ISO/快门/增益；画直方图与 PTC 曲线，估算读出噪声。

**产出**：《我的第一份 RAW 元数据报告》。

### W2 传统 ISP 模块逐个击破

**内容**：按真实 pipeline 的顺序推进——BLC → DPC/BPC → LSC → RAW 域降噪 → AWB（RAW 域增益）→ Demosaic → CCM → 线性化/Gamma → 色调映射 → 2D/3D NR → Sharpen → 色彩增强/肤色保护 → YUV 转换。

**动手**：用 NumPy 实现 BLC、双线性/双三次/Hamilton-Adams demosaic、灰度世界 AWB、3×3 CCM、Gamma，再和 `rawpy.postprocess()` 的不同参数组合做对比图。

**关键认知**：**顺序不可乱**——DPC 必须在 demosaic 之前，AWB 要在 RAW 域做；同时每个模块的输入输出位深、处在线性域还是非线性域，都要能说清。

### W3 数据与工具链

**工具**：rawpy/libraw、dcraw、`colour-science`、kornia、OpenCV，MATLAB 可选（做仿真对照）。

**数据集**（按体量从小到大上手）：MIT-Adobe FiveK（DNG + 修图 GT，练色调与色彩）→ SID（低光 RAW，5094 对）→ Zurich RAW-to-RGB（ZRR，手机 RAW + 单反 GT，RAW-to-RGB 的主战场）→ Mobile AI 2022 UltraISP（IMX586 四合一，大规模）→ SIDD/DND（真实降噪）。

**动手**：写一套统一的 Dataset——uint16 存盘、patch 随机裁剪、flip/rot 时保持 CFA 相位一致（**这是 RAW 数据增强最容易踩的坑**），并用 LMDB/WebDataset 加速。

### W4 评测体系 + 第一个小项目

**内容**：PSNR/SSIM（保真）、LPIPS（感知）、ΔE2000（色彩）、MTF/SFR（锐度）、噪声 σ、无参考 NIQE；核心认知是「像素指标好 ≠ 观感好」。

**项目 1**：手写 ISP 并与 libraw 对齐，输出对比图与指标表，写成一篇技术笔记。

## 三、第 2 个月（W5–W8）：深度学习 RAW-to-RGB 主力训练

### W5 精读经典，建立方法地图

**必读**：DeepISP（TIP 2019，端到端开山）、HDRNet（局部色调映射 + 引导滤波）、ISP-Net / PyNet、综述 *ISP meets Deep Learning: A Survey on Deep Learning Methods for Image Signal Processing*（arXiv 2305.11994）、ICCV 2023 Tutorial《Understanding the In-Camera Rendering Pipeline and the role of AI/Deep Learning》。

**要能讲清三种范式的差异与取舍**：

- **模块化替换**：逐个模块上网络，可解释、易调参；
- **联合优化**：模块之间可微串联；
- **端到端**：RAW→sRGB 一把梭，效果上限高，但难调、难解释。

### W6 复现第一个 baseline

在 ZRR 或 MAI2022 上跑通一个 U-Net / NAFNet 类 baseline：输入 Bayer 4 通道（按相位拆分）或 1→3 通道，输出 sRGB。

**训练要点**：

- 线性域归一化 `(raw - black) / (white - black)`，按通道做；
- 用 log / 分段编码处理高动态范围；
- 损失用 L1 + 感知（VGG/LPIPS）+ 色度损失；
- bf16 混合精度、EMA、余弦退火。

**4090 显存参考**：512×512 patch、batch 8–16、约 20–40M 参数可以稳定跑满；1024×1024 需要 batch 2–4 加 gradient checkpointing；打开 `torch.compile` 与 channels_last 提速。64G 内存足够做预取，但务必 patch 化，不要整图进内存。

### W7 数据侧的深水区

这一块是拉开差距的地方。

- **噪声建模**：Poisson-Gaussian 传感器噪声模型，按 ISO/增益分档合成噪声（ELD 的思路）。
- **逆 ISP / unprocessing**：从 sRGB 反推 RAW（RAW-Adapter 的 unprocess 流程、ParamISP 用 EXIF 相机参数控制正/逆 ISP），用来缓解 RAW 配对数据稀缺。
- **对齐问题**：手机 RAW 与单反 GT 之间存在视差、色彩差异和快门时序差异，直接上 L1 监督会糊；了解光流 warp、全局颜色映射（GCM）等解法。

### W8 3A 与画质调优

AI-ISP 岗位的必考项。

- **AE**：测光分区统计、目标亮度、收敛与防闪烁（曝光时间取工频整数倍）、AE 与增益分配（先曝光后增益，以保 SNR）。
- **AWB**：灰度世界、完美反射、动态阈值、白点检测与色温估计，以及 AI AWB。
- **AF**：反差检测（高频评价值 + 爬山搜索）与相位检测（PDAF 离焦量）、追焦与预测。
- **调优流程**：色卡（ColorChecker）+ 分辨率卡（SFRplus）+ 灰阶卡，客观指标配主观评审；理解「参数配置文件下发到板端」的工程形态。

**项目 2**：训练一个 RAW-to-sRGB 模型并做消融实验报告——损失、归一化、patch 尺寸、噪声建模各一组对比。

## 四、第 3 个月（W9–W12）：进阶专题 + 端侧工程化 + 作品集

### W9 多帧与视频

真实产品的主战场。

**内容**：MFNR 多帧降噪、HDR 多帧合成（对齐 → 权重融合 → 色调映射）、时域 3DNR 与运动检测、光流/可变形对齐、ZSL 零快门延迟流程。

**数据集/参考**：DRV、SMOID。动手做 3–5 帧合成 demo，与单帧对比。

### W10 前沿方向

**选 1–2 个深挖，不要贪多。**

- **生成式**：DarkDiff（Apple + Purdue，把预训练 Stable Diffusion 改用于暗光 RAW 增强，用区域化交叉注意力防幻觉）、ExposureDiffusion。要理解「回归导致过平滑」与「生成导致幻觉」之间的权衡。
- **高效结构**：LW-ISP（异构蒸馏）、unrolled optimization（参数量降 30 倍以上）、FourierISP（频域解耦风格与结构）、RMFA-Net（隐式黑电平 + 三通道拆分 + Retinex 色调映射）。
- **跨设备**：camera-to-camera transfer、RAW 域风格迁移，让一个模型服务多颗 sensor。

### W11 模型压缩与部署

剪枝/蒸馏/低秩 → PTQ 与 QAT（RAW 域动态范围大，量化要按通道或分组做，注意黑电平与高光截断）→ ONNX 导出（onnxsim 简化、Netron 看图）→ TensorRT INT8/FP16 在 4090 上跑吞吐与延迟；同时了解移动端 NPU 工具链（高通 SNPE/Hexagon、联发科 NeuroPilot、瑞芯微 RKNN）的算子限制与常见 fallback。

**产出**：一张延迟/内存/精度三维权衡表——面试里这东西最能体现工程能力。

### W12 综合项目与输出

**项目 3（作品集核心）**：「一颗 sensor 的完整 AI-ISP 方案」——自采或公开 RAW → 传统 ISP 基线 → 深度模型（含噪声建模与数据增强）→ 压缩量化 → TensorRT 部署 → 完整评测报告（PSNR/SSIM/LPIPS/ΔE/延迟/内存）+ 对比图。

**同步输出**：GitHub 仓库（README 写清复现步骤与指标）、2–3 篇技术博客；可选参加 Mobile AI / AIM Challenge 类赛事刷榜。

## 五、几个高频坑

提前避开，能省掉大量返工。

1. **CFA 相位**：flip/rot/裁剪之后必须保持 Bayer 相位一致，否则色彩错乱、训练不收敛。
2. **线性域与感知域**：噪声模型、损失、指标都要明确在哪个域计算；GT 的 gamma/色调映射策略不一致，会让指标失去意义。
3. **黑电平与白电平**：不同通道、不同增益档的取值不同，用单一常数会导致暗部偏色与条纹。
4. **只看 PSNR**：低光场景 PSNR 高但细节糊，必须配 LPIPS + 主观评审。
5. **显存误判**：4K RAW 整图训练不现实，务必 patch 化 + 分块推理（注意接缝与 halo）。
6. **数据量**：ZRR/MAI2022 体量大，先用小规模子集打通全流程，再放大。

## 六、资源清单

够用即可，不要囤积。

**论文**：DeepISP、HDRNet、ISP 深度学习综述（arXiv 2305.11994）、MAI 2022 Learned Smartphone ISP Challenge 报告、RAW-Adapter、ParamISP（CVPR 2024）、FourierISP（AAAI 2024）、RMFA-Net、DarkDiff（2025）、LW-ISP。

**教程与课程**：ICCV 2023 Tutorial（In-Camera Rendering Pipeline + AI）、各厂商公开的 ISP tuning 文档、Gonzalez《数字图像处理》补滤波与色彩基础。

**代码库**：rawpy/libraw、colour-science、kornia、PyTorch + timm、diffusers（做生成式专题）、onnxruntime / TensorRT / onnxsim。

**数据集**：MIT-Adobe FiveK、SID、ELD、SIDD/DND、Zurich RAW-to-RGB、MAI 2022 UltraISP。

## 最后

这份计划里真正难的不是模型，是**把链路每一段的输入输出域说清楚**：什么在 RAW 域、什么在线性域、什么已经在感知域，黑电平怎么标定，相位怎么对齐。模型换一代就过时，这些不会。

三个月后的验收标准其实只有一条：给你一颗 sensor 和一份 RAW，能不能独立走完从标定到部署的全流程，并且说得清每一步为什么这么做。
