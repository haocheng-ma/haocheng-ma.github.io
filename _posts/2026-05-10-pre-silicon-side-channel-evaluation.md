---
layout: post
title: "Pre-Silicon Side-Channel Evaluation: From Physics to Diffusion"
date: 2026-05-10
description: From layout-level EM simulation to GANs, transfer learning, and diffusion-based power modeling.
tags: security
giscus_comments: true
toc: true
---

_EMSim grew out of my Ph.D. work at Tianjin University. Ya Gao, whose research I supervise, led the later AI-accelerated work on EMSim+ and AIPS; I am a co-author. Seen together, the four projects follow a clear progression, with each one tackling a limitation of the previous approach._

## Why pre-silicon security evaluation?

When a cryptographic chip operates, logic-cell transitions produce power consumption and electromagnetic (EM) emanations that depend on the data being processed. Side-channel analysis (SCA) uses this physical information to recover secrets such as cryptographic keys without attacking the algorithm itself.

Security evaluation methods fall into two broad classes. Attack-based methods reproduce an attacker's attempt to recover sensitive information. Correlation electromagnetic analysis (CEMA), for example, derives hypothetical intermediate values from known plaintexts and key guesses, maps them to hypothetical EM emanations with a leakage model, and computes their Pearson correlation with measured EM traces. Measurement to Disclosure (MTD), the minimum number of traces required to recover the key, measures attack difficulty. Leakage-based methods instead test whether two data populations differ statistically; Test Vector Leakage Assessment (TVLA) is the best-known example. Because TVLA can report leakage that a practical attack does not exploit, the EMSim results discussed here rely primarily on attack-based CEMA.

Conventional security evaluation takes place after fabrication. If first silicon leaks, changes to the register-transfer-level (RTL) design or physical layout require another manufacturing cycle. Pre-silicon security evaluation moves this feedback into the design stage: simulate EM or power traces from design data, apply the same analysis methods used after fabrication, and revise the design before tape-out. The simulated data must match measurements in the relevant time and spatial domains while scaling to the trace counts required at higher security levels.

## The simulation challenge

Take an AES core through a typical sequence of revisions: layout and initial evaluation, the addition of masking or a modified power grid, and eventually migration to another process node. Each change requires a new assessment of information leakage. The simulator must preserve the temporal features of EM or power traces, retain the spatial distribution needed for localized EM analysis, and generate enough traces for the selected security level. Four bottlenecks dominate this process.

1. **Layout-level EM simulation is computationally expensive.** It derives logic cells, metal interconnect, parasitics, and transient currents from the physical layout, then calculates the magnetic field at observation points above the die. This physical information improves simulation accuracy but raises the cost of generating large EM datasets.

2. **A simplified physical simulation still runs once per trace.** EMSim is 32× faster than ConvEM, but every additional plaintext or mask value still requires current analysis and EM computation.

3. **A learned model incurs a per-design training cost.** A model trained for the baseline AES layout needs new input-output sample pairs when the layout, countermeasure, or fabrication process changes. Training from scratch at every iteration reduces the benefit of rapid large-scale data generation.

4. **Power side-channel evaluation needs transient power traces.** Learned models for thermal or voltage-drop analysis usually predict average power, whereas SCA depends on nanosecond-scale, data-dependent transients. Synopsys PrimeTime PX (PTPX) can generate transient power traces, but its cost grows with circuit size and trace length.

## EMSim — layout-level EM simulation

EMSim starts from the physical layout and consists of three stages: data preparation, current analysis, and EM computation. Data preparation builds a layout database through the RTL-to-GDS design flow. Current analysis derives the transient current distribution across the physical layout. EM computation then converts those currents into magnetic-field data at arbitrary observation points. Unlike a behavioral model based only on RTL or a gate-level netlist, the physical layout includes the fabrication process, cell locations, metal geometry, and parasitic parameters needed to approximate the behavior of fabricated silicon.

The simplifications in EMSim follow from two physical effects: current aggregation and metal shielding. Logic-cell transitions are the source of EM information, while the metal interconnect is the radiation carrier. Cell currents aggregate in the on-chip power network, producing relatively large transient currents in the top-level power grid. EM emanations from lower metal layers are attenuated by the upper layers. EMSim therefore focuses on the transient currents and magnetic field of the top-level power grid, retaining the dominant contributors while reducing computation.

It uses three techniques.

**Device-model approximation** represents logic-cell transitions as cell-level current-source excitations. Dynamic gate-level simulation first records cell transitions. Cell-level power analysis then obtains transient power from the lookup tables in the Liberty library and divides it by the supply voltage to derive transient current. This replaces the nonlinear equations of deep-submicron device models with piecewise-linear current lookup tables.

**Parasitic-network reduction** uses those predetermined cell-level current sources to remove signal interconnects from the parasitic-network model while retaining the on-chip power network needed to calculate transient currents in the top-level power grid. The smaller system reduces both nodal-analysis cost and convergence problems in SPICE simulation.

**GPU parallelization** accelerates EM computation. EMSim divides each metal wire into equal-area rectangular subregions, discretizes their magnetic-field contributions at each observation point, and expresses the cross-product and distance-matrix operations as multidimensional arrays. CPU mode uses NumPy; GPU mode uses CUDA-backed CuPy. Their compatible interfaces allow the matrix solver to switch between the two platforms.

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_pipeline.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 1. The EMSim layout-level EM simulation flow. Device-model approximation and parasitic-network reduction lower the cost of current analysis; GPU parallelization accelerates EM computation. Schematic based on <a href="{{ '/assets/pdf/TIFS_2023_EMSim.pdf' | relative_url }}">Ma et al., TIFS 2023</a>.
</div>

EMSim applies the Biot-Savart law to calculate the magnetic flux density at an observation point. After the $i$th metal wire is divided into rectangular subregions, Equation (3-8) of the thesis can be written in the following discrete form:

$$
\mathbf{B}_{t,p} \;\approx\; \frac{\mu_0}{4\pi}
\sum_i \sum_j \sum_k
\frac{\mathbf{J}_{i,t} \times \hat{\mathbf{r}}_{i,j,k,p}}
{r_{i,j,k,p}^{2}}\,\Delta S_{i,j,k}
$$

Here, $\mathbf{J}_{i,t}$ is the current density of metal wire $i$ at time $t$; $\hat{\mathbf{r}}_{i,j,k,p}$ and $r_{i,j,k,p}$ are the direction and distance from a subregion to observation point $p$; and $\Delta S_{i,j,k}$ is the subregion area. Summing the vector contribution of every subregion gives an approximation of the magnetic flux density at the observation point. Finer subdivision more closely approximates the surface integral but increases computation. In a measurement, changing magnetic flux induces a voltage in the near-field probe coil, so EMSim also converts its field result into a probe output signal.

The thesis validates EMSim with S-Box and AES chips designed and fabricated in SMIC 180 nm CMOS, considering simulation accuracy, security-evaluation accuracy, and computational cost. The simulated and measured data achieved more than 74% time-domain accuracy and up to 98% spatial accuracy; the reported accuracy in evaluating information-leakage risk was 93%. EMSim was 32× faster than ConvEM. These results apply to the experimental circuits, process, and near-field measurement setup in the thesis and should not be extrapolated to arbitrary layouts or probe configurations.

## EMSim+ (GAN) — GAN-based evaluation acceleration

EMSim still performs current analysis and EM computation for every trace. EMSim+ (ICCAD 2023) reformulates chip-level EM simulation as a paired image-to-image translation problem: EMSim first produces a small set of input-output sample pairs, those pairs train a generative adversarial network, and the trained generator synthesizes the large magnetic-field dataset required for security evaluation.

The model input comprises a **cell-current distribution**, a **power-grid distribution**, and a time sequence. The cell-current distribution accumulates the transient current of each logic cell into its corresponding layout tile. The power-grid distribution uses the locations of metal wires and supply ports to describe the impedance characteristics of the top-level power grid. The output sample is the spatial magnetic-field distribution simulated by EMSim. These data are normalized before model training.

The model is based on Pix2Pix. Its U-Net generator extracts spatial features from the cell-current and power-grid distributions, fuses them with the time feature, and outputs a synthetic magnetic-field distribution. During adversarial training, the discriminator distinguishes simulated magnetic-field distributions from synthetic ones. During risk quantification, the trained generator synthesizes the required number of magnetic-field samples, which are then analyzed with CEMA or another security-evaluation method.

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_gan.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 2. The EMSim+ generative adversarial network. The generator synthesizes a magnetic-field distribution from the cell-current distribution, power-grid distribution, and time sequence; the discriminator distinguishes synthetic data from samples generated by EMSim. Adapted from <a href="{{ '/assets/pdf/ICCAD_2023_EMSim+.pdf' | relative_url }}">Gao et al., ICCAD 2023</a>.
</div>

Generator inference replaces repeated current analysis and EM computation at large dataset sizes, but model training still requires EMSim input-output sample pairs. The larger the evaluation dataset, the easier it is to amortize data preparation and training.

For the four evaluated cryptographic circuits — AES, Kyber, a processor with an AES instruction-set extension, and a masked AES implementation — the paper reports a speedup of more than **242×** over EMSim at one million EM traces. In a separate experiment, the model was trained with measurements from a fabricated 180 nm AES-128 chip. The synthetic and measured magnetic-field distributions reached 99.5% NCC and 94.2% SSIM; MTD was about 265 traces for EMSim+ and 173 for the measurements. Both datasets allowed key recovery, but that does not make their statistical properties identical.

The trained model, however, remains tied to the design that supplied the paired data.

## EMSim+ (GAN+TL) — transfer across related designs

Training from scratch gives every modified AES layout the same startup cost. The TIFS 2024 extension uses transfer learning to reuse a model trained on the baseline design.

The GAN is first trained on a source design. For a target design, its pretrained parameters are copied, the discriminator is frozen, and the generator is fine-tuned with target-design samples. The experiment used 500 target pairs rather than the 1,000 used for training from scratch, cutting training time by roughly half. This demonstrates reuse within the evaluated design family; it does not show that 500 samples will suffice for an arbitrary circuit.

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/emsim_tl.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 3. The EMSim+ transfer-learning flow: train on source-design data $D_s$ to synthesize source magnetic-field distributions $T_s$, then freeze the discriminator and fine-tune the generator on target-design data $D_t$ to synthesize target distributions $T_t$. Adapted from <a href="{{ '/assets/pdf/TIFS_2024_EMSim+.pdf' | relative_url }}">Gao et al., TIFS 2024</a>.
</div>

The source model was trained on an AES design in SMIC 180 nm. The four target designs were two masked AES variants (`AES_mask_1` and `AES_mask_2`), an AES layout with added power-grid strips (`AES_pg`), and the AES benchmark reimplemented in SMIC 55 nm (`AES_55nm`). Against magnetic-field distributions simulated by EMSim, the transferred models exceeded 99.5% in both NCC and structural similarity (SSIM). CEMA on the synthetic EM traces also reflected the results of different leakage models: for example, a first-order model did not reveal the key in `AES_mask_1`, whereas a toggle-count model targeting internal S-Box gates did. All four target-design comparisons use EMSim as the reference; the silicon comparison in the previous section applies only to the baseline AES experiment.

Transfer learning does not change generator inference speed; it reduces the cost of target-design data preparation and model training. Including target-sample generation and fine-tuning, the paper reports a 113.37–149.23× speedup over EMSim at 100,000 EM traces. With a suitable pretrained source model, the estimated speedup at one million traces is 282.04–483.98×.

The three EMSim projects address the EM side channel. AIPS extends learned simulation to the power side channel.

## AIPS — diffusion-based power simulation

AIPS (TCHES 2026) models the transient power caused by data-dependent logic transitions. PTPX can generate reference power traces at nanosecond resolution, but its cost grows with design size and trace length. Earlier learned power estimators largely targeted average power and therefore missed the transient features required for SCA.

AIPS uses transient power traces generated by PTPX as training targets. Its conditioning features come from two design sources. A value change dump (VCD) is divided into fixed time windows and converted into per-cell toggle counts. The standard-cell library supplies power lookup-table values and load capacitance. AIPS aligns these features with PTPX samples and learns to generate the transient power for each time window.

AIPS uses diffusion because GAN training can be unstable or collapse onto a limited set of output modes. This is especially problematic when rare, data-dependent variations carry information leakage. Diffusion replaces adversarial training with a fixed denoising objective and tends to cover the output distribution more broadly. Toggle activity and standard-cell library features condition the denoising process, tying each synthetic power trace to the simulated workload.

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/aips_diffusion.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 4. The AIPS diffusion-model training flow. VCD toggle activity and standard-cell library features provide the condition; the model learns to denoise $x_T$ into $x_0$ using transient power traces generated by PTPX as the reference. Adapted from <a href="{{ '/assets/pdf/TCHES_2026_AIPS.pdf' | relative_url }}">Gao et al., TCHES 2026</a>.
</div>

The conditional reverse-diffusion step is:

$$
p_\theta(\mathbf{x}_{t-1} \mid \mathbf{x}_t, \mathbf{c}) \;=\; \mathcal{N}\!\left(\boldsymbol{\mu}_\theta(\mathbf{x}_t, t, \mathbf{c}),\; \boldsymbol{\Sigma}_\theta(\mathbf{x}_t, t, \mathbf{c})\right)
$$

At each reverse step, the model uses the current power trace $\mathbf{x}_t$, diffusion step $t$, and conditioning features $\mathbf{c}$ to estimate a distribution for the less noisy $\mathbf{x}_{t-1}$. Repeating the process produces a synthetic transient power trace. Because diffusion sampling is iterative, a fair cost comparison must include inference as well as the one-time cost of PTPX data generation and model training.

The paper evaluates AIPS separately on five targets:

- **AES** (the baseline cipher, unmasked)
- **Kyber** (a NIST-selected post-quantum key-encapsulation mechanism)
- **Two masked AES variants** (used to evaluate higher-order SCA)
- **A RISC-V core** (a general-purpose processor executing AES, not a dedicated crypto block)

Across the tested settings, AIPS improved power-trace similarity over the GAN baseline and reproduced the SCA results obtained with PTPX data, including first-order attacks on AES and Kyber, higher-order analysis of masked AES, and key-byte recovery on the RISC-V core. These conclusions apply to the leakage models and attack methods used in the paper; they do not establish that every leakage order or implementation detail is preserved. On efficiency, the paper reports:

- **4.14–42.44× faster than PTPX at 1M traces**, including data preparation and training
- **Up to 10⁴× per-trace speedup once training is amortized**
- **About 1K or fewer training traces** in the reported minimum-data experiments
- **A larger throughput advantage as trace length increases**

## Putting the four projects together

{% include figure.liquid path="assets/img/2026-05-10-pre-silicon-side-channel-evaluation/lineage.png" class="img-fluid rounded z-depth-0 mx-auto d-block" zoomable=true %}

<div class="caption">
    Figure 5. The four-paper lineage: layout-level EM simulation → GAN-based evaluation acceleration, per-design training → transfer across related designs, and the EM side channel → the power side channel (with GAN → diffusion in the final step).
</div>

Three changes define the progression. EMSim+ trains a GAN with a small EMSim dataset and then synthesizes a much larger magnetic-field dataset. Transfer learning reuses the model when the AES design or fabrication process changes. AIPS moves from the EM side channel to the power side channel, and from GAN-synthesized magnetic-field distributions to diffusion-generated transient power traces.

| Paper           | Side channel | Core method                                                                    | Scope                        | Training data            | Speedup vs. reference flow                             | Validated on                                            |
| --------------- | ------------ | ------------------------------------------------------------------------------ | ---------------------------- | ------------------------ | ------------------------------------------------------ | ------------------------------------------------------- |
| EMSim           | EM           | device-model approximation + parasitic-network reduction + GPU parallelization | per layout                   | n/a                      | 32× over ConvEM                                        | S-Box and AES chips in SMIC 180 nm                      |
| EMSim+ (GAN)    | EM           | conditional GAN                                                                | per trained design           | 1K pairs in experiments  | >242× over EMSim at 1M traces                          | Four crypto circuits; separate 180 nm AES silicon study |
| EMSim+ (GAN+TL) | EM           | GAN + transfer learning                                                        | related designs/nodes        | 500 target pairs         | 113–149× at 100K; estimated 282–484× at 1M             | AES source + four AES-derived targets, 180 nm and 55 nm |
| AIPS            | Power        | conditional diffusion                                                          | trained/evaluated per target | about 1K traces or fewer | 4.14–42.44× over PTPX at 1M; up to 10⁴× inference-only | AES, Kyber, masked AES (×2), and RISC-V                 |

The limitations are just as important as the speedups. The learned models depend on reference traces and on the leakage models and attack methods used for validation. Transfer was demonstrated within an AES-centered design family, and AIPS was trained and evaluated separately for each target rather than as a universal model. Pre-silicon side-channel security evaluation still requires trade-offs among simulation accuracy, reference-data cost, model transferability, side-channel coverage, and security-evaluation accuracy.

## Open problems

**Masked and post-quantum designs at scale.** AIPS covers Kyber and two masked AES variants, but the experiments do not establish coverage for higher masking orders, signature schemes such as ML-DSA or Falcon, or complete systems-on-chip. Those settings raise open questions about training-set size, leakage orders, background switching activity, and diffusion sampling cost.

**Closing the loop between evaluation and protection.** Security evaluation can locate leakage hotspots or validate attack results, but it does not automatically identify the paths along which sensitive information leaks or produce a countermeasure. PathFinder (DAC 2022 · [PDF]({{ '/assets/pdf/DAC_2022_PathFinder.pdf' | relative_url }})) and Formal Path (TECS 2025 · [PDF]({{ '/assets/pdf/TECS_2025_Formal_Path.pdf' | relative_url }})) are our work on leakage-path identification and logic-level obfuscation. Connecting leakage-path analysis and targeted hardening to rapid reevaluation remains an open problem.

## Code

- [`github.com/jinyier/EMSim`](https://github.com/jinyier/EMSim) — EMSim and EMSim+ implementations.
- [`github.com/jinyier/AIPS`](https://github.com/jinyier/AIPS) — AIPS implementation.

## References

1. H. Ma, M. Panoff, J. He, Y. Zhao, Y. Jin. EMSim: A Fast Layout Level Electromagnetic Emanation Simulation Framework for High Accuracy Pre-Silicon Verification. _IEEE Transactions on Information Forensics and Security_, 2023. [PDF]({{ '/assets/pdf/TIFS_2023_EMSim.pdf' | relative_url }})
2. Y. Gao, H. Ma, J. Kong, J. He, Y. Zhao, Y. Jin. EMSim+: Accelerating Electromagnetic Security Evaluation with Generative Adversarial Network. _IEEE/ACM ICCAD_, 2023. [PDF]({{ '/assets/pdf/ICCAD_2023_EMSim+.pdf' | relative_url }})
3. Y. Gao, H. Ma, Q. Zhang, X. Song, Y. Jin, J. He, Y. Zhao. EMSim+: Accelerating Electromagnetic Security Evaluation With Generative Adversarial Network and Transfer Learning. _IEEE Transactions on Information Forensics and Security_, 2024. [PDF]({{ '/assets/pdf/TIFS_2024_EMSim+.pdf' | relative_url }})
4. Y. Gao, H. Ma, T. Zhang, J. He, Y. Zhao, M. Stojilović, Y. Jin. AIPS: AI-Based Power Simulation for Pre-Silicon Side-Channel Security Evaluation. _IACR Transactions on Cryptographic Hardware and Embedded Systems_, 2026. [PDF]({{ '/assets/pdf/TCHES_2026_AIPS.pdf' | relative_url }})
5. H. Ma. Research on Pre-Silicon Security Evaluation and Protection Techniques for Cryptographic Chip. Ph.D. Thesis, Tianjin University, 2023. [PDF]({{ '/assets/pdf/PhD_Thesis_2023.pdf' | relative_url }})
