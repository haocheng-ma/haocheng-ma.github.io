---
layout: post
title: "Pre-Silicon Side-Channel Evaluation: From Physics to Diffusion"
date: 2026-05-10
description: 四篇论文梳理密码芯片硅前侧信道安全测评的发展：从版图级电磁仿真，到生成对抗网络、迁移学习与基于扩散模型的瞬态功耗仿真。
tags: security
lang: zh
hidden: true
permalink: /blog/2026/pre-silicon-side-channel-evaluation/zh/
giscus_comments: true
toc: true
---

_EMSim 源于我在天津大学的博士研究。后续的 EMSim+ 与 AIPS 由我指导的高雅主导，我作为共同作者参与。把四项工作放在一起看，可以看到一条清楚的演进路线：每项工作都在解决上一种方法留下的问题。_

## 为什么要开展硅前安全测评？

密码芯片运行时，逻辑单元的翻转活动会产生与处理数据相关的功耗和电磁辐射。侧信道分析（Side-Channel Analysis，SCA）利用这些物理信息恢复密钥等敏感数据，而不直接攻击密码算法本身。

安全测评通常采用两类方法。基于攻击的评估方法直接模拟攻击者恢复敏感信息；例如，相关电磁分析（Correlation Electromagnetic Analysis，CEMA）根据已知明文和猜测密钥构造假设中间值，经信息泄露模型映射为假设电磁辐射，再与实际电磁曲线计算皮尔逊相关系数。破解密钥所需的最小曲线数量（Measurement To Disclosure，MTD）可用于衡量攻击难度。基于信息泄露的评估方法则判断数据分组之间是否存在统计差异，例如测试向量泄露评估（TVLA）。TVLA 可能报告无法被实际攻击利用的泄露，因此本文涉及 EMSim 的测评结果时，以 CEMA 的攻击结果为主要依据。

传统安全测评集中在硅后阶段。如果首版芯片存在信息泄露，修改寄存器传输级（RTL）设计或物理版图后还要重新经历制造环节。硅前安全测评把这一反馈移到芯片设计阶段：从设计数据仿真电磁曲线或功耗曲线，采用与硅后测评相同的分析方法量化信息泄露风险，再据此修改设计。难点在于，仿真结果既要在时域和空间域上接近实际测量，又要满足高安全等级测评对大规模数据量的要求。

## 仿真的四个瓶颈

以一个 AES 核的迭代过程为例：先完成布局布线和初次测评，再加入掩码或修改电源网格，最后迁移到另一个工艺节点。每次改动之后，都需要重新量化信息泄露风险。仿真方法既要保留电磁曲线或功耗曲线的时域特征，也要在涉及局部电磁分析时保留空间分布，还要生成足够数量的曲线。这个过程主要受四个瓶颈限制。

1. **版图级电磁仿真的计算成本高。** 仿真需要从物理版图获得逻辑单元、金属互连、寄生参数和瞬态电流，再计算芯片表面任意观测点的磁场。物理信息提高了仿真精度，也增加了大规模生成电磁曲线的成本。

2. **模型简化后的电磁仿真仍需逐条计算。** EMSim 相比 ConvEM 提升了 32 倍的时间效率，但每增加一个明文或掩码值，仍要重新执行电流分析和电磁计算。

3. **学习模型存在逐设计训练的成本。** 为基准 AES 版图训练的模型，在版图、防护方案或制造工艺变化后，需要新的输入输出样本对。每次迭代都从头训练，会削弱快速生成大规模测评数据的优势。

4. **功耗侧信道测评需要瞬态功耗曲线。** 面向热分析或电压降分析的学习模型通常预测平均功耗，而侧信道分析依赖纳秒尺度、与数据相关的瞬态变化。Synopsys PrimeTime PX（PTPX）可以生成瞬态功耗曲线，但计算成本会随电路规模和曲线长度增长。

## EMSim：版图级电磁仿真

EMSim 从物理版图出发，主要包括数据准备、电流分析和电磁计算三个环节。数据准备环节由 RTL 到 GDS 的芯片设计流程建立版图数据库；电流分析环节获得物理版图上的瞬态电流分布；电磁计算环节再依据电磁场理论，将瞬态电流转化为任意观测点的磁场数据。与只使用 RTL 或门级网表的行为模型相比，物理版图还包含制造工艺、单元位置、金属布线尺寸和寄生参数，因此更接近实际芯片的电磁行为。

EMSim 的理论依据来自电流聚合效应和金属屏蔽效应。逻辑单元的翻转活动是电磁信息的来源，金属互连是辐射载体。逻辑单元产生的电流在片上电源网络中聚合，使顶层电源网格流过较大幅度的瞬态电流；较低金属层产生的电磁辐射则受到上层金属屏蔽。因此，EMSim 重点计算顶层电源网格的瞬态电流和磁场，在保留主导因素的同时降低计算成本。

具体采用三项技术。

**器件模型近似**将逻辑单元的翻转活动等效为单元级电流源激励。动态门级仿真先记录逻辑单元的翻转活动，单元功耗分析再根据 Liberty 库中的功耗查找表得到瞬态功耗，并除以电源电压得到瞬态电流。这样，深亚微米器件模型中的非线性方程求解被转换为分段线性的电流查找表。

**寄生网络约减**利用已经确定的单元级电流源，从寄生网络模型中排除信号互连线，保留获得顶层电源网格瞬态电流所需的片上电源网络。这样既减少了节点方程的计算量，也降低了 SPICE 仿真的收敛难度。

**GPU 并行计算**用于电磁计算环节。EMSim 将金属线划分为等面积的矩形子区域，离散求解各子区域对观测点的磁场贡献，并把叉乘矩阵和距离矩阵的运算表示为多维数组。CPU 模式使用 NumPy，GPU 模式使用基于 CUDA 的 CuPy；两者接口兼容，可以切换矩阵求解平台。

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_pipeline.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    图 1. EMSim 的版图级电磁仿真流程。器件模型近似和寄生网络约减降低电流分析成本，GPU 并行计算加速电磁计算。示意图依据 <a href="{{ '/assets/pdf/TIFS_2023_EMSim.pdf' | relative_url }}">Ma 等，TIFS 2023</a> 绘制。
</div>

EMSim 依据毕奥-萨伐尔定律计算观测点的磁感应强度。将第 $i$ 条金属线划分为矩形子区域后，论文公式（3-8）可写成以下离散形式：

$$
\mathbf{B}_{t,p} \;\approx\; \frac{\mu_0}{4\pi}
\sum_i \sum_j \sum_k
\frac{\mathbf{J}_{i,t} \times \hat{\mathbf{r}}_{i,j,k,p}}
{r_{i,j,k,p}^{2}}\,\Delta S_{i,j,k}
$$

其中，$\mathbf{J}_{i,t}$ 是第 $i$ 条金属线在时刻 $t$ 的电流密度，$\hat{\mathbf{r}}_{i,j,k,p}$ 和 $r_{i,j,k,p}$ 分别表示子区域到观测点 $p$ 的方向和距离，$\Delta S_{i,j,k}$ 是子区域面积。将所有子区域的贡献矢量相加，即得到观测点的近似磁感应强度。子区域划分越细，离散结果越接近面积分，但计算量也越大。实际测量时，变化的磁通量在近场探头线圈中感应出电压信号，因此 EMSim 还会把磁场结果转换为探头接收信号。

论文采用中芯国际 180 nm CMOS 工艺设计并制造 S-Box 和 AES 芯片，从仿真精度、测评准确度和计算成本三个方面验证 EMSim。仿真结果与实测数据的时域准确度高于 74%，空间域准确度高达 98%，对信息泄露风险的测评准确度为 93%；相对于 ConvEM，EMSim 的时间效率提升了 32 倍。这些结果适用于论文中的实验电路、工艺和近场测量设置，不能直接外推到任意版图与探头配置。

## EMSim+（GAN）：基于生成对抗网络的测评优化

EMSim 仍需逐条执行电流分析和电磁计算。EMSim+（ICCAD 2023）把芯片电磁仿真转换为配对图像翻译问题：先由 EMSim 生成少量输入输出样本对，训练生成对抗网络，再由生成器快速合成安全测评所需的大规模磁场数据。

模型输入由**单元电流分布**、**电源网格分布**和时间序列组成。单元电流分布将各逻辑单元的瞬态电流累加到对应版图网格；电源网格分布依据金属线与供电端口的位置描述顶层电源网格的阻抗特性。输出样本是 EMSim 仿真的空间磁场分布。在论文的测评优化流程中，这些数据经过归一化后用于模型训练。

模型以 Pix2Pix 为基础。U-Net 生成器提取单元电流分布和电源网格分布的空间特征，并与时间特征融合，输出合成磁场分布；判别器在对抗训练中区分真实磁场分布与合成磁场分布。进入风险量化环节后，只需使用训练好的生成器合成规定数量的磁场数据，再结合 CEMA 等安全评估方法量化信息泄露风险。

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_gan.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    图 2. EMSim+ 的生成对抗网络模型：生成器根据单元电流分布、电源网格分布和时间序列合成磁场分布，判别器区分合成数据与 EMSim 生成的真实样本。改绘自 <a href="{{ '/assets/pdf/ICCAD_2023_EMSim+.pdf' | relative_url }}">Gao 等，ICCAD 2023</a>。
</div>

生成器推理取代了大规模数据量下反复进行的电流分析和电磁计算，但模型训练仍需要 EMSim 提供输入输出样本对。测评数据量越大，前期数据准备和模型训练成本越容易被摊销。

论文评估了四种密码电路：AES、Kyber、带 AES 指令扩展的处理器，以及掩码 AES。在一百万条电磁曲线时，EMSim+ 相对 EMSim 的加速超过 **242×**。另一项独立实验使用一颗 180 nm AES-128 芯片的实测数据训练模型：合成磁场分布与实测磁场分布的 NCC 为 99.5%，SSIM 为 94.2%；破解密钥所需的最小曲线数量（MTD）在 EMSim+ 数据上约为 265，在实测数据上约为 173。两组数据都能恢复密钥，但不能据此认为它们的统计特性完全一致。

不过，训练后的模型仍与提供配对数据的设计绑定。

## EMSim+（GAN+TL）：在相关设计间迁移

如果每个修改后的 AES 版图都要从头训练，每次迭代都要重复支付前期成本。TIFS 2024 的扩展引入迁移学习，复用基准设计上已经训练好的模型。

首先在源设计上训练 GAN。面对目标设计时，复制预训练参数，冻结判别器，再用目标设计样本微调生成器。实验使用 500 个目标样本对，从头训练则使用 1,000 个样本对，训练时间大约减半。这个结果说明模型能在论文评估的设计族中复用，但不能推断任意电路都只需 500 个样本即可完成迁移。

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_tl.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    图 3. EMSim+ 迁移学习流程：先用源设计数据 $D_s$ 训练模型以合成源设计磁场分布 $T_s$，随后冻结判别器，用目标设计数据 $D_t$ 微调生成器以合成目标设计磁场分布 $T_t$。改绘自 <a href="{{ '/assets/pdf/TIFS_2024_EMSim+.pdf' | relative_url }}">Gao 等，TIFS 2024</a>。
</div>

源模型来自中芯国际 180 nm 工艺的 AES 设计。四个目标设计包括两个掩码 AES 变体（`AES_mask_1` 和 `AES_mask_2`）、增加电源网格金属条的 AES 版图（`AES_pg`），以及在中芯国际 55 nm 工艺下重新实现的 AES（`AES_55nm`）。以 EMSim 仿真的磁场分布为参考，迁移模型的 NCC 和结构相似性（SSIM）均超过 99.5%。对合成电磁曲线执行 CEMA，也能反映不同信息泄露模型的攻击结果：例如，一阶模型无法从 `AES_mask_1` 恢复密钥，而针对 S-Box 内部逻辑门的翻转计数模型可以。这里四个目标设计均以 EMSim 为参照；上一节的硅片实测对比只适用于基准 AES 实验。

迁移学习不会改变生成器的推理速度；它降低的是目标设计的数据准备和模型训练成本。计入目标样本生成与微调后，论文报告在十万条电磁曲线时相对 EMSim 加速 113.37–149.23×；在已有合适源模型的前提下，一百万条电磁曲线时的估算加速比为 282.04–483.98×。

前三项工作都面向电磁侧信道。AIPS 则把学习型仿真扩展到功耗侧信道。

## AIPS：用扩散模型生成瞬态功耗

AIPS（TCHES 2026）对数据相关逻辑翻转产生的瞬态功耗进行建模。PTPX 可以生成纳秒分辨率的参考功耗曲线，但成本会随设计规模和曲线长度增长。此前的学习型功耗估算方法大多面向平均功耗，无法保留侧信道分析所需的瞬态特征。

AIPS 以 PTPX 生成的瞬态功耗曲线为训练目标，条件特征来自两类设计数据。首先，将值变化转储（VCD）文件切分为固定时间窗口，并统计各逻辑单元的翻转次数；其次，从标准单元库提取功耗查找表数值和负载电容。AIPS 将这些特征与 PTPX 样本对齐，学习生成各时间窗口的瞬态功耗。

AIPS 采用扩散模型，是因为 GAN 训练可能不稳定，也可能只覆盖有限的输出模式。当少见的数据相关变化承载信息泄露时，这个缺点尤其突出。扩散模型将对抗训练换成固定的去噪目标，通常能覆盖更完整的输出分布。在 AIPS 中，翻转活动与标准单元库特征作为去噪条件，使每条合成功耗曲线与被仿真的工作负载对应。

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/aips_diffusion.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    图 4. AIPS 的扩散模型训练流程：VCD 翻转活动与标准单元库特征共同构成条件，模型学习把 $x_T$ 逐步去噪为 $x_0$，并以 PTPX 生成的瞬态功耗曲线为参考。改绘自 <a href="{{ '/assets/pdf/TCHES_2026_AIPS.pdf' | relative_url }}">Gao 等，TCHES 2026</a>。
</div>

条件反向扩散步骤为：

$$
p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{c}) \;=\; \mathcal{N}\!\left(\boldsymbol{\mu}_\theta(\mathbf{x}_t, t, \mathbf{c}),\; \boldsymbol{\Sigma}_\theta(\mathbf{x}_t, t, \mathbf{c})\right)
$$

在每个反向步骤中，模型根据当前功耗曲线 $\mathbf{x}_t$、扩散步骤 $t$ 和条件特征 $\mathbf{c}$，估计噪声更少的 $\mathbf{x}_{t-1}$ 的分布。重复这个过程，最终得到一条合成瞬态功耗曲线。由于扩散采样需要多次迭代，比较效率时既要计算推理成本，也要计算一次性的 PTPX 数据生成和模型训练成本。

论文分别在五个目标上评估 AIPS：

- **AES**：无掩码的基准密码核
- **Kyber**：NIST 选定的后量子密钥封装机制
- **两个掩码 AES 变体**：用于测试高阶侧信道分析
- **一个 RISC-V 核**：由通用处理器执行 AES，而非专用密码模块

在论文的实验设置下，AIPS 相比 GAN 基线提高了功耗曲线的相似性，并复现了基于 PTPX 数据得到的侧信道分析结果，包括针对 AES 和 Kyber 的一阶攻击、对掩码 AES 的高阶分析，以及在 RISC-V 核上的密钥字节恢复。这些结论只适用于论文采用的信息泄露模型和攻击方法，不能证明所有泄露阶数或实现细节都得到保留。效率方面，论文报告：

- **在一百万条功耗曲线时比 PTPX 快 4.14–42.44×**，包含数据准备和训练成本
- **训练成本摊销后，单条功耗曲线最高加速 10⁴×**
- **最小数据量实验使用约一千条或更少的训练曲线**
- **曲线越长，吞吐量优势越明显**

## 四项工作的关系

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/lineage.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    图 5. 四篇论文形成的研究路线：版图级电磁仿真 → 基于生成对抗网络的测评优化，逐设计训练 → 在相关设计间迁移，以及电磁侧信道 → 功耗侧信道（同时由生成对抗网络转向扩散模型）。
</div>

这条路线经历了三次变化。EMSim+ 用少量 EMSim 样本训练生成对抗网络，再合成大规模磁场数据；迁移学习在 AES 设计或制造工艺变化后复用已有模型；AIPS 则从电磁侧信道转向功耗侧信道，并由生成对抗网络合成磁场分布改为用扩散模型生成瞬态功耗曲线。

| 工作             | 侧信道类型 | 核心方法                                   | 适用范围             | 训练数据           | 相对参考流程的加速                                          | 验证对象                                            |
| ---------------- | ---------- | ------------------------------------------ | -------------------- | ------------------ | ----------------------------------------------------------- | --------------------------------------------------- |
| EMSim            | 电磁       | 器件模型近似 + 寄生网络约减 + GPU 并行计算 | 每个版图             | 不适用             | 相对 ConvEM 提升 32×                                        | 中芯国际 180 nm 的 S-Box 与 AES 芯片                |
| EMSim+（GAN）    | 电磁       | 条件生成对抗网络                           | 每个已训练设计       | 实验使用 1K 样本对 | 一百万条电磁曲线时相对 EMSim >242×                          | 四种密码电路；另含 180 nm AES 芯片实测实验          |
| EMSim+（GAN+TL） | 电磁       | 生成对抗网络 + 迁移学习                    | 相关设计与工艺节点   | 500 个目标样本对   | 十万条时 113–149×；一百万条时估算 282–484×                  | AES 源设计及四个 AES 衍生目标，覆盖 180 nm 与 55 nm |
| AIPS             | 功耗       | 条件扩散模型                               | 分别训练并评估各目标 | 约 1K 条或更少     | 一百万条功耗曲线时相对 PTPX 为 4.14–42.44×；纯推理最高 10⁴× | AES、Kyber、两个掩码 AES 与 RISC-V                  |

这些加速数字之外，方法的适用边界同样重要。学习模型依赖参考曲线以及用于验证的信息泄露模型和攻击方法；迁移学习只在以 AES 为中心的设计族中得到验证；AIPS 也是针对不同目标分别训练和测评，并非适用于所有设计的通用模型。硅前侧信道安全测评仍需在仿真精度、参考数据成本、模型迁移能力、侧信道类型和测评准确度之间取舍。

## 开放问题

**掩码和后量子设计的大规模测评。** AIPS 已覆盖 Kyber 和两个掩码 AES 变体，但这些实验没有证明它能覆盖更高阶掩码、ML-DSA 或 Falcon 等签名方案，以及完整的片上系统。这些场景仍需研究训练集规模、信息泄露阶数、背景翻转活动和扩散采样成本。

**形成安全测评与防护的闭环。** 安全测评可以定位信息泄露热点或验证攻击结果，却不会自动识别敏感信息的泄露路径，也不会直接给出防护方案。PathFinder（DAC 2022 · [PDF]({{ '/assets/pdf/DAC_2022_PathFinder.pdf' | relative_url }})) 和 Formal Path（TECS 2025 · [PDF]({{ '/assets/pdf/TECS_2025_Formal_Path.pdf' | relative_url }})) 是我们在安全溯源和逻辑级混淆方面的工作。如何把泄露路径识别、靶向增强与快速重新测评连接起来，仍需进一步研究。

## 代码

- [`github.com/jinyier/EMSim`](https://github.com/jinyier/EMSim) — EMSim 与 EMSim+ 的实现。
- [`github.com/jinyier/AIPS`](https://github.com/jinyier/AIPS) — AIPS 的实现。

## 参考文献

1. H. Ma, M. Panoff, J. He, Y. Zhao, Y. Jin. EMSim: A Fast Layout Level Electromagnetic Emanation Simulation Framework for High Accuracy Pre-Silicon Verification. _IEEE Transactions on Information Forensics and Security_, 2023. [PDF]({{ '/assets/pdf/TIFS_2023_EMSim.pdf' | relative_url }})
2. Y. Gao, H. Ma, J. Kong, J. He, Y. Zhao, Y. Jin. EMSim+: Accelerating Electromagnetic Security Evaluation with Generative Adversarial Network. _IEEE/ACM ICCAD_, 2023. [PDF]({{ '/assets/pdf/ICCAD_2023_EMSim+.pdf' | relative_url }})
3. Y. Gao, H. Ma, Q. Zhang, X. Song, Y. Jin, J. He, Y. Zhao. EMSim+: Accelerating Electromagnetic Security Evaluation With Generative Adversarial Network and Transfer Learning. _IEEE Transactions on Information Forensics and Security_, 2024. [PDF]({{ '/assets/pdf/TIFS_2024_EMSim+.pdf' | relative_url }})
4. Y. Gao, H. Ma, T. Zhang, J. He, Y. Zhao, M. Stojilović, Y. Jin. AIPS: AI-Based Power Simulation for Pre-Silicon Side-Channel Security Evaluation. _IACR Transactions on Cryptographic Hardware and Embedded Systems_, 2026. [PDF]({{ '/assets/pdf/TCHES_2026_AIPS.pdf' | relative_url }})
5. H. Ma. Research on Pre-Silicon Security Evaluation and Protection Techniques for Cryptographic Chip. Ph.D. Thesis, Tianjin University, 2023. [PDF]({{ '/assets/pdf/PhD_Thesis_2023.pdf' | relative_url }})
