# [FaceScape: A Large-Scale High Quality 3D Face Dataset and Detailed Riggable 3D Face Prediction](zotero://select/library/items/SCCRLU4C)

 **Authors:** Haotian Yang, Hao Zhu, Yanru Wang, Mingkai Huang, Qiu Shen, Ruigang Yang, Xun Cao
 **Published:** 2020
 **Journal/Conference:** 2020 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)
**Citekey:** `yangFaceScapeLargeScaleHigh2020`

---
## Abstract
> [!abstract]
>  ```
>  In this paper, we present a large-scale detailed 3D face dataset, FaceScape, and propose a novel algorithm that is able to predict elaborate riggable 3D face models from a single image input. FaceScape dataset provides 18,760 textured 3D faces, captured from 938 subjects and each with 20 speciﬁc expressions. The 3D models contain the pore-level facial geometry that is also processed to be topologically uniformed. These ﬁne 3D facial models can be represented as a 3D morphable model for rough shapes and displacement maps for detailed geometry. Taking advantage of the large-scale and high-accuracy dataset, a novel algorithm is further proposed to learn the expression-speciﬁc dynamic details using a deep neural network. The learned relationship serves as the foundation of our 3D face prediction system from a single image input. Different than the previous methods, our predicted 3D models are riggable with highly detailed geometry under different expressions. The unprecedented dataset and code will be released to public for research purpose†.
>  ```

---

%% begin notes %%
## Summary
## Introduction

High-quality 3D face reconstruction underpins face tracking, recognition, synthesis, and animation, all of which depend critically on large-scale, geometrically detailed 3D training data. Existing 3D face datasets are limited in both scale and resolution: active sensors (Kinect, structured light) lack fine geometric detail, while sparse multi-view systems produce unstable reconstructions. FaceScape addresses both deficits simultaneously — providing the largest dataset of any prior work and capturing geometry down to pore and wrinkle level.

## Related Work

| Category | Prior approaches | Limitation addressed by FaceScape |
|---|---|---|
| **3D face datasets (model-fitting)** | Basel Face Model [Booth et al., CVPR 2016], Large Scale 3DMM [Booth et al., IJCV 2018], 3DDFA [Zhu et al., CVPR 2016] | Fitted 3DMM shapes lack detailed geometry and accuracy |
| **3D face datasets (depth/scanner)** | BU-3DFE [Yin et al., FG 2006], BU-4DFE [Yin et al., FG 2008], BJUT-3D [Baocai et al., 2009], Bosphorus [Savran et al., 2008], FaceWarehouse [Cao et al., TVCG 2013], 4DFAB [Cheng et al., CVPR 2018] | Low spatial resolution, limited expression range |
| **3D face datasets (multi-view)** | D3DFACS [Cosker et al., ICCV 2011], BP4D-Spontaneous [Zhang et al., 2014] | Only 6–3 cameras; unstable reconstruction; small subject count |
| **3DMM parametric modelling** | Bilinear model [Vlasic et al., ToG 2005], FaceWarehouse [Cao et al.], FLAME [Li et al., ToG 2017], nonlinear 3DMMs [Tran & Liu, CVPR 2018/2019], MeshGAN [Cheng et al., 2019] | Insufficient representation power for wrinkle-level details |
| **Single-view detail prediction** | Richardson et al. CVPR 2017 (multi-layer depth refinement), Sela et al. ICCV 2017 (image-to-image), Sengupta et al. CVPR 2018 (SfSNet), Extreme3D [Tran et al., CVPR 2018] (bump map), DFDN [Chen et al., 2019] (cond. GAN + displacement), Huynh et al. CVPR 2018 (mesoscopic geometry) | Predicted detail is static — not riggable across expressions |

FaceScape is the first dataset enabling expression-dependent dynamic detail learning. The predicted models uniquely support rigging to arbitrary expressions with plausible geometric detail — a capability absent from all prior single-view methods.

## Methods & Architecture

**Dataset construction pipeline**

1. **Capture:** A dense 68-DSLR camera array (30 × 8K front cameras, 38 × 4K side cameras; shutters synced within 5 ms) captures 938 subjects (ages 16–70), each performing 20 FACS-guided expressions — yielding 18,760 raw meshes with $\approx 2$M vertices and 4M triangle faces.
2. **Topologically uniformed (T.U.) model:** Raw meshes are registered to a template via Procrustes alignment of 3D landmarks then Non-rigid ICP (NICP). For non-neutral expressions, deformation transfer initialises the registered template before NICP refines it. The resulting T.U. base shapes share identical topology across all subjects and expressions.
3. **Displacement maps:** A two-layer representation encodes fine detail not captured by the coarse base mesh. Per pixel, the signed distance from the base mesh surface to the raw scan is stored in UV space. Compression from the raw mesh to base + displacement achieves $\approx 2\%$ of original data size with mean absolute error $< 0.3$ mm.
4. **Bilinear model:** Tucker decomposition of a rank-3 tensor (26317 vertices $\times$ 52 expressions $\times$ 938 identities) yields a core tensor $C_r$ and low-dimensional identity/expression components. New shapes are generated as:
$$V = C_r \times w_{\text{exp}} \times w_{\text{id}}$$

**Single-image prediction pipeline (three stages)**

**Stage 1 — Base model fitting.** Parameters $w_{\text{id}}$, $w_{\text{exp}}$, and $w_{\text{alb}}$ are estimated by minimising:
$$E = E_{\text{lan}} + \lambda_1 E_{\text{pixel}} + \lambda_2 E_{\text{id}} + \lambda_3 E_{\text{exp}} + \lambda_4 E_{\text{alb}}$$
where $E_{\text{lan}}$ is weak-perspective landmark alignment, $E_{\text{pixel}}$ is Lambertian pixel-level consistency using spherical harmonics illumination, and $E_{\text{id}}, E_{\text{exp}}, E_{\text{alb}}$ are Gaussian regularisers. Individual-specific blendshapes $B_i$ are then extracted via:
$$B_i = C_r \times \hat{w}_{\text{exp}}^{(i)} \times w_{\text{id}}, \quad 0 \le i \le 51$$

**Stage 2 — Displacement map prediction.** A pix2pixHD network (generator $G$ with three multi-scale LSGAN discriminators $D_1, D_2, D_3$) takes as input the deforming map (vertex displacement in UV space from neutral to target expression) stacked with the UV texture, and predicts a $1024 \times 1024$ displacement map for each of the 20 key expressions. Training loss:
$$\min_G \left( \max_{D_1,D_2,D_3} \sum_{k} \mathcal{L}_{\text{adv}}(G, D_k) + \lambda \sum_{k} \mathcal{L}_{\text{FM}}(G, D_k) \right)$$
The deforming map provides expression-motion cues that the static texture alone cannot supply — an ablation confirms its necessity.

**Stage 3 — Dynamic detail synthesis.** For an arbitrary target expression with blendshape weight $\alpha$, the final displacement map $F$ is synthesized as a weighted blend of the 20 predicted expression displacement maps $\hat{F}_i$:
$$F = M_0 \odot \hat{F}_0 + \sum_{i=1}^{19} M_i \odot \hat{F}_i$$
Weight mask $M_i$ for each expression is derived from activation masks $A_j(p) = \|e_j(p) - e_0(p)\|_2$ (per-vertex motion magnitude) linearly combined by blendshape weights $\alpha$, ensuring locally-active regions receive appropriate blending.

## Experiments & Results

**Bilinear model fitting accuracy** (test set: 50 held-out subjects, 20 expressions each = 1000 meshes):

| Model | Params | Advantage |
|---|---|---|
| FaceWarehouse (FW) | 50 identity, 47 expression | outperformed by FaceScape at same param count |
| FLAME | 300 identity, 100 expression | outperformed by FaceScape (300 id, 52 exp) |
| **FaceScape (ours)** | 50 id, 52 exp | lowest cumulative reconstruction error |

FaceScape bilinear model achieves visually more mid-scale detail than both FW and FLAME (qualitative comparison in Figure 3).

**3D face prediction error** (point-to-plane, mm):

| Method | Mean error | Variance |
|---|---|---|
| 3DDFA [Zhu et al.] (source exp.) | 2.17 | 3.23 |
| Extreme3D [Tran et al.] (source exp.) | 2.06 | 2.55 |
| DFDN [Chen et al.] (source exp.) | 2.19 | 3.20 |
| **Ours (source exp.)** | **1.22** | **1.17** |
| **Ours (all exp.)** | **1.39** | **2.33** |

**Ablation study:**
- *W/O dynamic detail:* replacing per-expression displacement maps with a single source-expression map eliminates expression-driven wrinkles from rigged results.
- *W/O deforming map:* replacing the motion deforming map with one-hot expression encoding removes expression-specific geometric detail from the predicted displacement maps.

## Conclusion

FaceScape is claimed to be the largest and highest-quality 3D face dataset, outperforming all prior datasets in subject count (938), geometric resolution ($\approx 2$M-vertex pore-level detail), expression count (20), and image resolution (4K–8K). The paper claims to be the first to predict a detailed and riggable 3D face model from a single image, with the reconstructed model supporting per-expression dynamic detail synthesis for arbitrary blendshape weights. The bilinear model trained on FaceScape outperforms FaceWarehouse and FLAME on held-out reconstruction accuracy.

## Impact

FaceScape fills a critical gap between parametric 3DMM methods (coarse but animatable) and detail-prediction methods (sharp but static): it enables detailed, animatable face reconstruction from a single RGB image. The dataset will benefit any learning-based 3D face method that requires high-fidelity ground-truth geometry — including face reenactment, neural radiance field avatars, and expression synthesis. The main limitation is geographic/demographic bias (subjects mostly from Asia, captured in controlled lab conditions), which may reduce generalisation to unconstrained faces. The capture infrastructure is also expensive and inaccessible, limiting reproducibility and extension of the dataset itself.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-06-02T22:02:20.326-04:00 %%
