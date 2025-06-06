好的，这是一份关于GR-1的技术报告，详细阐述了其技术原理、实现细节，并参考了基于GR-1的进一步研究。

# GR-1 技术报告

## 摘要

GR-1 是一种为多任务语言条件视觉机器人操作设计的 GPT 风格模型。它通过大规模视频生成式预训练学习有用的表征，然后针对机器人数据进行微调。GR-1 以语言指令、观测图像序列和机器人状态序列作为输入，端到端地预测机器人动作和未来图像。实验表明，GR-1 在具有挑战性的 CALVIN 基准测试和真实机器人上均取得了显著成果，尤其在零样本未见场景泛化方面表现突出。

## 1. GR-1 技术原理与实现细节

这部分构成本报告的主要内容，详细介绍 GR-1 的核心技术。

### 1.1 问题定义

GR-1 的研究涉及两个主要阶段的问题定义：

* **视频生成式预训练 (Video Generative Pre-Training):**
    在此阶段，GR-1 被训练来预测未来的视频帧 ($o_{t+\Delta t}$)。输入包括视频的语言描述 ($l$) 和一系列过去的视频帧 ($o_{t-h:t}$)。预训练的数据集由视频及其对应的语言描述对 ($v=\{l,o_{1},o_{2},...,o_{T}\}$组成。

* **多任务视觉机器人操作 (Multi-Task Visual Robot Manipulation):**
    在机器人操作任务中，GR-1 学习将语言指令 ($l$)、观测图像序列 ($o_{t-h:t}$) 和机器人状态序列 ($s_{t-h:t}$) 映射到机器人动作 ($a_t$) 和未来图像预测 ($o_{t+\Delta t}$)。语言指令描述了具体任务（例如，“向左滑动红色方块”）。观测序列包含视觉图像，状态序列包括机器人末端执行器的姿态和夹爪的二元状态。机器人数据集由专家轨迹组成，每条轨迹包含语言指令以及观测、状态和动作序列 ($\tau=\{l,o_{1},s_{1},a_{1},o_{2},s_{2},a_{2},...,o_{T},s_{T},a_{T}\}$)。

### 1.2 模型架构

GR-1 采用了一个简单的 GPT 风格的 Transformer 架构，能够处理不同模态的输入，并输出未来图像和机器人动作。

#### 1.2.1 输入处理

* **语言输入 (Language Input):** 语言指令 ($l$) 使用一个冻结的 CLIP (Radford et al., 2021) 文本编码器进行编码。
* **视觉输入 (Visual Input):** 视觉观测 ($o$) 由一个使用 MAE (He et al., 2022) 预训练过的 Vision Transformer (ViT)进行编码。输出的 CLS 令牌 ($z_{o}^{CLS}$) 提供全局图像表示，而输出的补丁令牌 ($z_{o}^{p_{1zi}}$) 作为局部表示。这些局部表示会进一步通过一个 Perceiver Resampler (Jaegle et al., 2021) 来减少令牌数量。
* **机器人状态输入 (Robot State Input):** 机器人状态 ($s$) 包括机器人末端执行器的6D姿态 ($s_{arm}\in SE(3)$) 和二元夹爪状态 ($s_{gripper}\in\{0,1\}$)。这些状态通过线性层进行编码。
* **嵌入对齐 (Embedding Alignment):** 在送入因果 Transformer 之前，来自所有模态的嵌入都通过线性层进行维度对齐。
* **特殊令牌 (Special Tokens):**
    * 为了预测动作，模型学习一个动作预测令牌嵌入，称为 `[ACT]`。
    * 为了预测视频，模型学习几个观测预测令牌嵌入，称为 `[OBS]`，用于预测未来帧。
* **令牌序列化 (Token Sequencing):**
    * **视频生成式预训练期间:** 令牌序列为 `(l, $o_{t-h}$, [OBS], l, $o_{t-h+1}$, [OBS], ..., l, $o_{t}$, [OBS])`。
    * **机器人数据微调期间:** 令牌序列为 `(l, $s_{t-h}$, $o_{t-h}$, [OBS], [ACT], l, $s_{t-h+1}$, $o_{t-h+1}$, [OBS], [ACT], ..., l, $s_{t}$, $o_{t}$, [OBS], [ACT])`。
* **时间信息 (Temporal Information):** 模型学习相对时间步长嵌入，并将其添加到令牌中以注入时间信息。在给定时间步长的所有模态共享相同的时间嵌入。语言令牌在每个时间步长重复出现，以防止其被其他模态信息淹没。

#### 1.2.2 网络机制

GR-1 利用了生成式预训练模型中典型的**因果注意力机制 (Causal Attention Mechanism)**。然而，所有的 `[ACT]` 和 `[OBS]` 令牌都被掩码。这意味着在预训练期间，一个令牌可以关注所有先前的令牌，除了 `[OBS]` 令牌。在微调期间，一个令牌可以关注所有先前的令牌，除了 `[ACT]` 和 `[OBS]` 令牌。

#### 1.2.3 输出生成

* **视频预测解码器 (Video Prediction Decoder):** 一个由自注意力模块和多层感知机 (MLP) 组成的 Transformer 解码器连接到对应于 `[OBS]` 令牌和掩码令牌的输出。每个掩码令牌是一个共享的、可学习的嵌入，并结合了位置编码。每个掩码令牌的输出重建预测未来图像的一个补丁。视频预测的损失函数 ($L_{video}$) 是像素空间中重建图像与原始图像之间的均方误差 (MSE)。
* **动作预测解码器 (Action Prediction Decoder):** 来自 `[ACT]` 令牌的输出通过线性层来预测机械臂和夹爪的动作。
    * 对于连续的机械臂动作，使用 Smooth-L1 损失 ($L_{arm}$) 进行训练。
    * 对于二元的夹爪动作，使用二元交叉熵 (BCE) 损失 ($L_{gripper}$) 进行训练。

### 1.3 训练工作流

GR-1 的训练过程分为两个阶段：

1.  **预训练 (Pre-training):**
    * **数据集:** Ego4D 数据集 (Grauman et al., 2022)，包含大规模的人类与物体交互视频。从中收集了约 80 万个视频片段（总计 800 万帧），每个片段时长3秒。
    * **目标:** GR-1 被训练来预测在给定视频语言描述和过去帧序列的情况下，$t+\Delta t$ 时刻的视频帧。具体来说，是预测采样帧序列中的下一帧图像，即在预训练中 $\Delta t=1$。
    * **帧采样策略:** 从 Ego4D 数据集的视频片段中等间隔采样帧。连续帧之间的持续时间为 1/3 秒，以确保足够的视觉差异。
    * **损失函数:** 网络使用因果视频预测损失 ($L_{video}$) 进行优化。
    * **冻结模块:** CLIP 文本编码器和 MAE 图像编码器在预训练期间保持冻结。

2.  **机器人数据微调 (Robot Data Fine-tuning):**
    * **数据处理:** 机器人数据相对于预训练中使用的采样视频帧序列更为密集。因此，在机器人数据上进行微调时，$\Delta t$ 设置为 3。GR-1 被训练来预测来自静态摄像头和夹爪摄像头的捕获图像。
    * **损失函数:** GR-1 通过结合因果行为克隆损失和视频预测损失进行端到端优化。组合损失函数为：$L_{finetune} = L_{arm} + L_{gripper} + L_{video}$。
    * **冻结模块:** CLIP 文本编码器和 MAE 图像编码器在微调期间也保持冻zeń。

### 1.4 实现细节 (网络参数与训练配置)

根据论文附录 A.1 及相关章节，关键实现细节如下：

| 参数类别             | 配置                                                       |
| :------------------- | :--------------------------------------------------------- |
| **因果Transformer** |                                                            |
| 层数                 | 12                                                         |
| 注意力头             | 12                                                         |
| 隐藏层维度           | 384                                                        |
| **参数量** |                                                            |
| 总计                 | 1.95亿                                                     |
| 可训练参数           | 4600万                                                     |
| **动作预测输出层** | 3层 MLP，最后一层为双头输出 (机械臂动作和夹爪动作)             |
| **视频预测输出层** | Transformer (由自注意力模块和线性层组成)                     |
| 视频预测目标         | 归一化的补丁级别 (Normalized patch-wise)                   |
| 图像增强             | 随机平移增强 (Random shift augmentation)                     |
| 输入序列长度         | 10                                                         |
| **优化器** | AdamW                                                      |
| **学习率调度** | 余弦衰减 (Cosine decay)                                    |
| Dropout              | 0.1                                                        |
| **预训练超参数** |                                                            |
| 批量大小             | 1024                                                       |
| 学习率               | $3.6 \times 10^{-4}$                                         |
| 预热周期             | 5                                                          |
| 训练周期             | 50                                                         |
| 帧预测 $\Delta t$    | 1 (对于预训练中等间隔采样的帧)                             |
| **机器人数据微调超参数** |                                                            |
| 批量大小             | 512                                                        |
| 学习率               | $1 \times 10^{-3}$                                         |
| 预热周期             | 1                                                          |
| 训练周期             | 20                                                         |
| 帧预测 $\Delta t$    | 3 (对于机器人数据中的连续步骤)                             |

### 1.5 实验设置与主要结果

#### 1.5.1 CALVIN 基准测试实验

* **任务类型:** CALVIN 基准包含34种不同的、通过语言指令指定的机器人操作任务，例如操作滑动门、抽屉、不同颜色的方块、LED灯和灯泡等。
* **环境描述 (参考图4):** CALVIN 基准包含四个不同的环境 (Env A, B, C, D)，它们在桌子颜色和物体配置（如滑动门、LED、灯泡、开关、按钮的位置）上有所不同。
* **动作预测细节:** GR-1 预测机械臂的 XYZ 位置增量和欧拉角增量，以及二元夹爪动作。
* **数据集划分:**
    * **ABCD→D:** 模型使用来自所有四个环境 (A, B, C, D) 的数据进行训练，并在环境 D 中进行评估。
    * **ABC→D:** 模型使用来自环境 A, B, C 的数据进行训练，并在训练期间未见过的环境 D 中进行评估。
    * 训练数据集包含超过2万条专家轨迹，配有语言指令标签。这仅占CALVIN数据集（包含24小时的遥操作非导向游戏数据）中1%的已标注数据。
* **评估指标 (参考表1):**
    * **连续完成任务数 (Tasks completed in a row):** 衡量连续完成1、2、3、4、5个任务的成功率。
    * **平均长度 (Avg. Len.):** 通过对所有评估序列中连续完成5个任务的平均数量进行计算，综合衡量长期任务能力。

* **CALVIN 主要结果 (参考表1):**
    * **多任务学习 (ABCD→D):** GR-1 显著优于基线方法。
        * 完成1个任务的成功率: 94.9% (对比最佳基线 HULC MT-R3M 的 88.9%)。
        * 平均长度: 4.21 (对比最佳基线 HULC MT-R3M 的 3.06)。
    * **零样本未见场景泛化 (ABC→D):** GR-1 表现出显著提升。
        * 完成1个任务的成功率: 85.4% (对比最佳基线 RT-1 的 53.3%)。
        * 平均长度: 3.06 (对比最佳基线 MT-R3M 的 0.93)。
    * **数据效率 (ABCD→D 数据的10%):** GR-1 展示了高数据效率。
        * 完成1个任务的成功率: 77.8% (对比最佳基线 HULC 的 66.8%)。
        * 平均长度: 2.00 (对比最佳基线 HULC 的 1.11)。
    * **零样本未见语言泛化 (unseen lang):** GR-1 优于所有基线。
        * 完成1个任务的成功率: 76.4% (对比最佳基线 HULC 的 71.5%)。
        * 平均长度: 2.17 (对比最佳基线 HULC 的 1.82)。

#### 1.5.2 真实机器人实验

* **机器人平台:** 7自由度的 Kinova Gen2 机械臂。
* **相机配置:** 一个 RealSense 相机安装在末端执行器上，一个 Kinect Azure 相机提供场景的静态视图。
* **任务类型 (参考图5):**
    * **物体运输:** 将物体（如甜椒、茄子、西兰花、番茄、黄桃）在盘子和桌子之间运输。
    * **关节物体操作:** 操作抽屉（打开和关闭）。
* **已见/未见物体/类别评估设置 (参考表2):**
    * **已见物体:** 运输训练数据中出现过的物体（茄子、西兰花、甜椒）。评估还包括两个有干扰物的场景（番茄、玉米、黄桃）和背景变化（木板、碗）以评估鲁棒性。
    * **未见实例:** 运输一组训练中未见过的新茄子、西兰花和甜椒实例。
    * **未见类别:** 运输其类别在机器人训练数据中完全未见过的新物体（番茄、黄桃）。

* **真实机器人主要结果 (参考表2):**
    * **物体运输:** GR-1 在所有三种设置中均优于基线方法。
        * 已见物体: 成功率 79% (对比 RT-1 的 27% 和 MT-R3M 的 15%)。
        * 未见实例: 成功率 30% (对比 RT-1 的 13% 和 MT-R3M 的 13%)。
        * 未见类别: 成功率 73% (对比 RT-1 的 0% 和 MT-R3M 的 10%)。
    * **关节物体操作:** GR-1 显著优于基线。
        * 成功率: 75% (对比 RT-1 的 35% 和 MT-R3M 的 30%)。

#### 1.5.3 关键性能指标

* **成功率与性能:** GR-1 在 CALVIN 基准的各种挑战性设置（包括多任务学习、零样本未见场景泛化、数据效率和零样本未见语言泛化）中持续获得更高的成功率和平均长度。在真实机器人实验中，GR-1 也表现出卓越的性能，尤其是在泛化到未见物体实例和类别方面。
* **视频预测质量 (参考图6及4.3节):** GR-1 能够准确重建 CALVIN 和真实机器人数据上的未来帧，尽管一些微小细节（如被遮挡的物体）可能会丢失。这表明视频预测信号为动作预测提供了强有力的指导机制。

## 2. 基于GR-1的进一步研究

自 GR-1 (arXiv:2312.13139, ICLR 2024) 发表以来，寻找直接且可验证的、明确扩展其核心视频生成式预训练方法用于机器人操作的后续研究，在现有工具条件下具有一定挑战性。然而，GR-1 的部分作者在相关领域持续进行研究，其工作为视觉语言动作模型和通用机器人策略的发展指明了方向。

以下是截至2023年12月之后发表的、与GR-1研究方向紧密相关的最新研究工作：

1.  **"Towards Generalist Robot Policies: What Matters in Building Vision-Language-Action Models"**
    * **作者:** Xinghang Li, Peiyan Li, Minghuan Liu, Dong Wang, Jirong Liu, Bingyi Kang, Xiao Ma, Tao Kong, Hanbo Zhang, Huaping Liu. (其中 Xinghang Li, Minghuan Liu, Tao Kong 为 GR-1 作者)
    * **arXiv ID:** arXiv:2412.14058 (请注意，此ID对应的提交日期可能在未来，但论文已在线可查阅)
    * **链接:** [https://arxiv.org/abs/2412.14058](https://arxiv.org/abs/2412.14058)
    * **简介:** 该研究深入探讨了构建通用视觉语言动作 (VLA) 模型的关键设计选择和重要因素。它旨在为未来VLA模型的设计提供详细的指导手册，并提出了一个名为 "RoboVLMs" 的框架。论文中提及了对大规模视觉语言预训练在通用机器人策略中的作用的开放性问题，并可能在CALVIN等基准上进行了评估。虽然这篇论文不一定直接扩展GR-1的视频生成模块，但它关注的是如何构建更通用的机器人智能体，这与GR-1的目标一脉相承。

2.  **"SInViG: A Self-Evolving Interactive Visual Agent for Human-Robot Interaction"**
    * **作者:** Jiafeng Xu, Hanyang Zhang, Xinghang Li, Huaping Liu, Xinchao Lan, Tao Kong. (其中 Xinghang Li, Tao Kong 为 GR-1 作者)
    * **arXiv ID:** arXiv:2402.11792 (2024年2月)
    * **链接:** [https://arxiv.org/abs/2402.11792](https://arxiv.org/abs/2402.11792)
    * **简介:** SInViG 提出了一种自进化的交互式视觉基座代理，用于人机交互场景。该工作聚焦于通过对话来解决语言模糊性，提升机器人在复杂指令下的理解和执行能力。虽然其核心技术点与GR-1的生成式视频预训练有所不同，但它代表了GR-1作者在提升机器人语言理解和交互能力方面的相关探索。

由于工具限制，未能深入分析上述论文是否直接引用或在技术上延续了GR-1的视频生成式预训练框架。然而，这些工作无疑代表了GR-1研究团队在机器人学习、多模态理解和通用智能体方向上的持续努力和最新进展。未来可能会有更多研究直接受到GR-1在视频生成预训练方面的启发。

## 3. 结论

GR-1 通过其创新的大规模视频生成式预训练和GPT风格的统一架构，在视觉机器人操作领域取得了显著的性能提升，特别是在任务泛化和数据效率方面。它为机器人学习领域展示了大型预训练模型的巨大潜力。虽然直接的、明确的技术扩展性后续研究尚不明确，但其核心作者团队仍在积极推动相关领域（如通用机器人策略、人机交互）的发展，预示着该方向未来广阔的研究前景。