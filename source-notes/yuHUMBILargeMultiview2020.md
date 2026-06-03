# [HUMBI: A Large Multiview Dataset of Human Body Expressions](zotero://select/library/items/X7PGPNYW)

 **Authors:** Zhixuan Yu, Jae Shin Yoon, In Kyu Lee, Prashanth Venkatesh, Jaesik Park, Jihun Yu, Hyun Soo Park
 **Published:** 2020
 **Journal/Conference:** 2020 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)
**Citekey:** `yuHUMBILargeMultiview2020`

---
## Abstract
> [!abstract]
>  ```
>  This paper presents a new large multiview dataset called HUMBI for human body expressions with natural clothing. The goal of HUMBI is to facilitate modeling view-speciﬁc appearance and geometry of gaze, face, hand, body, and garment from assorted people. 107 synchronized HD cameras are used to capture 772 distinctive subjects across gender, ethnicity, age, and physical condition. With the multiview image streams, we reconstruct high ﬁdelity body expressions using 3D mesh models, which allows representing view-speciﬁc appearance using their canonical atlas. We demonstrate that HUMBI is highly effective in learning and reconstructing a complete human model and is complementary to the existing datasets of human body expressions with limited views and subjects such as MPII-Gaze, Multi-PIE, Human3.6M, and Panoptic Studio datasets.
>  ```

---

%% begin notes %%
## Summary
## Introduction

Authentic telepresence requires photorealistic rendering of human body signals — gaze, facial expressions, and gestures — but this is difficult because appearance depends on viewpoint, texture, illumination, and geometry in complex ways. Existing appearance models are either built from dense scans of individual target subjects (not scalable to assorted people) or trained on datasets with too few views and subjects to generalize across identities. HUMBI pushes toward two extremes simultaneously — the number of synchronized camera views and the number of distinctive subjects — to enable generalizable, view-specific appearance and geometry modeling of the total human body.

## Related Work

The paper positions HUMBI against five families of prior datasets and methods:

**Gaze**
- **Columbia Gaze** [62] and **UT-Multiview** [64]: controlled environments with fixed head pose, 5–8 cameras, 50–56 subjects.
- **Eyediap** [43]: 1 depth + 1 HD camera, 16 subjects, allows head motion.
- **MPII-Gaze** [81]: 1 camera, 15 subjects, in-the-wild (laptops), 214K images.
- **RT-GENE** [18]: eye-tracking glasses, 17 subjects, free-ranging point of regard.

**Face**
- **3DMM** [10] / **BFM** [49]: 3D scanners of 200 subjects for morphable face models.
- **300W-LP** [84]: 3DMM fitted to 60K samples from multiple face alignment datasets.
- **Deep appearance model** [39]: view-dependent appearance via conditional VAE; HUMBI is designed to supply real multi-identity data for such models.

**Hand**
- Depth-only methods: **NYU Hand** [69], **HandNet** [74], **BigHand2.2M** [78] use depth cameras ± magnetic sensors for 1–10 subjects.
- Synthetic: **RHD** [86] (44K images, 20 models), **ObMan** [21] (141K pairs).
- Real multi-view: **STB** [80] (1 stereo-camera subject), **FreiHAND** [15] (8 cameras).

**Body**
- **Human EVA** [60]: 4–7 cameras, 4 subjects, marker + markerless.
- **Human3.6M** [23]: 4 HD cameras, 11 subjects, motion capture ground truth.
- **Panoptic Studio** [24, 27, 61]: 31 HD + 480 VGA cameras, ~100 subjects, social scenes with significant occlusion.
- **D-FAUST** [11] / **Dyna** [52]: 22 stereo pairs, 10 subjects, high-resolution body dynamics.

**Garment**
- **ClothCap** [51], **BUFF** [79], **3DPW** [71]: natural clothing on 5–10 subjects; limited subject and view diversity.

## Methods & Architecture

**Capture rig:** 107 synchronized HD cameras arranged on a dodecagon frame (2.5 m diameter). 69 cameras are uniformly distributed across two arc levels (0.8 m and 1.6 m, ~10° baseline between adjacent cameras). 38 additional cameras densify the frontal quadrant for face/gaze capture (~10 cm baseline). All cameras are calibrated with **COLMAP** [59] and upgraded to metric scale via physical baselines.

**Subjects:** 772 distinctive participants (50.7% female, 49.3% male), spanning a wide age range, diverse skin colors, and varied clothing styles. Activities are loosely guided by performance instructions.

**Unified representation:** each body expression is encoded as a triple:

$$\{K,\ M,\ A\}$$

where $K$ is a set of 3D keypoints (triangulated from 2D detections [14] via RANSAC + nonlinear reprojection refinement), $M = \{V, E\}$ is a 3D mesh, and $A : \mathbb{R}^2 \to [0,1]^3$ is a view-specific appearance map over a canonical UV atlas.

| Expression | Model | Size | Fitting function |
|---|---|---|---|
| Gaze | Surrey [22] eye region UV | – | head-coordinate $g \in S^2$ |
| Face | Surrey blend shape [22] | 3,448 V / 6,736 F | $M_\text{face} = f_\text{face}(K_\text{face}, I_\text{face})$, minimize reprojection error over shape/expression/illumination/texture |
| Hand | MANO [53] | 778 V / 1,538 F | $M_\text{hand} = f_\text{hand}(K_\text{hand})$, L2 parameter regularization + MLE of shape |
| Body | SMPL [40] | 4,129 V / 7,999 F | $M_\text{body} = f_\text{body}(K_\text{body}, O_\text{body})$, surface matched to outer surface of occupancy map |
| Garment | Template cloth mesh | 3,763–11,238 V per piece | $M_\text{cloth} = f_\text{cloth}(M_\text{body}, O_\text{body})$, Laplacian regularization [63] |

The body **occupancy map** $O_\text{body}$ is built via shape-from-silhouette using human body segmentation [37]. Body semantics (head, torso, arms, legs) are labeled by projecting body labels [76] onto the voxel grid.

Garment topology is manually selected per subject from six templates: tops (sleeveless, T-shirt, long-sleeve) and bottoms (short, medium, long).

## Experiments & Results

All cross-data evaluations follow the same protocol: train on one or more datasets, test on held-out sets, and compare models trained on existing datasets alone versus combined with HUMBI.

**Gaze (monocular 3D gaze prediction, mean angular error in degrees)**

Baseline network [81] trained/tested across MPII, UTMV, and HUMBI:
- HUMBI alone achieves cross-dataset error of **7.9°** (self), degrading by less than 1° on cross-data transfer — the smallest cross-data drop of any dataset tested.
- MPII+HUMBI outperforms MPII alone by **4.1°** on UTMV; UTMV+HUMBI outperforms UTMV alone by **13.9°** on MPII.
- HUMBI has the smallest average gaze bias (**5.98°** vs. 6.69°–14.04° for other datasets) and near-largest variance.

**Face (monocular 3D face mesh prediction, reprojection error in pixels)**

Network [77] trained on 3DDFA, HUMBI, and combined:
- 3DDFA+HUMBI improves over 3DDFA alone by **2.8 pixels** and over HUMBI alone by **4.9 pixels**.
- Adding HUMBI substantially reduces view-dependency of the learned model (Figure 7).

**Hand (3D hand keypoint prediction, AUC of PCK over 0–20 mm)**

Cross-data evaluation across STB, RHD, FreiHAND, HUMBI using detector [86]:
- HUMBI alone is more generalizable than the other three datasets by a margin of **0.02–0.16 AUC**.
- Adding HUMBI to any single dataset improves performance by **0.04–0.12 AUC**.

**Hand mesh prediction (reprojection error in pixels)**

Training ObMan vs HUMBI vs ObMan+HUMBI (network [77]):
- ObMan+HUMBI outperforms ObMan alone and HUMBI alone by **0.3** and **1.7 pixels** respectively, despite the synthetic-to-real domain gap.

**Body keypoint prediction (AUC of PCK over 0–150 mm)**

Cross-data: Human3.6M (H36M), MPI-INF-3DHP (MI3D), HUMBI:
- HUMBI is more generalizable than H36M and MI3D by **0.023** and **0.064 AUC**.
- Combining HUMBI with H36M or MI3D adds **+0.057** and **+0.078 AUC**.

**Body mesh prediction (reprojection error in pixels)**

UP-3D vs HUMBI vs UP-3D+HUMBI:
- UP-3D+HUMBI reduces error on the HUMBI test set by **13.5 pixels** vs UP-3D alone.

**Garment reconstruction (camera ablation)**

Reconstruction density saturates at ~90 cameras; accuracy saturates at ~60 cameras — confirming robustness even below the full 107-camera rig.

## Conclusion

HUMBI is a large-scale multiview dataset of human body expressions capturing 772 subjects across gender, ethnicity, age, and physical condition with 107 synchronized HD cameras. The paper shows that models trained on HUMBI generalize better across datasets than models trained on prior gaze, face, hand, and body datasets, and that HUMBI is systematically complementary to those datasets — combining HUMBI with any existing dataset consistently improves cross-data performance. The 3D mesh reconstructions with canonical UV appearance atlases enable view-specific appearance modeling directly from real data.

## Impact

HUMBI is one of the most practically relevant large-scale datasets for human appearance and geometry research: its unprecedented combination of subject diversity (772 real people, natural clothing) and dense view coverage (107 cameras) fills a gap that synthetic datasets and small-scale real captures both fail to address. For multi-view RGB-D pose and appearance pipelines, HUMBI is a natural pre-training or fine-tuning source — particularly for gaze, face, and hand modules where view-invariant geometric representations are critical. Its main limitation is that the body sub-dataset lacks pose diversity (confirmed by the body mesh results), meaning it is better suited as a complementary source than a standalone training set for body-pose tasks. The dataset's canonical UV appearance maps also open a direct path to neural rendering and novel-view synthesis of real people, making it relevant well beyond pose estimation.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-06-02T22:02:31.010-04:00 %%
