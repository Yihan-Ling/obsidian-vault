# [Direct Multi-view Multi-person 3D Pose Estimation](zotero://select/library/items/MPZJJ69S)

 **Authors:** tao wang, Jianfeng Zhang, Yujun Cai, Shuicheng Yan, Jiashi Feng
 **Published:** 2021
 **Journal/Conference:** Advances in Neural Information Processing Systems
**Citekey:** `wangDirectMultiviewMultiperson2021`

---
## Abstract
> [!abstract]
>  ```
>  
>  ```

---

%% begin notes %%
## Summary

## Introduction
Multi-view multi-person 3D pose estimation — localizing 3D skeleton joints for every person in a scene from synchronized cameras — is foundational for surveillance, sports broadcast, gaming, and mixed reality. Prior art either runs a costly multi-stage reconstruction pipeline (2D detection → cross-view matching → triangulation) or builds an explicit 3D feature volume (heatmap back-projection → per-person regression), both of which scale linearly in cost with the number of people and depend on fragile intermediate representations. MvP shows it is possible to collapse the entire pipeline into a single direct-regression stage with a transformer, achieving higher accuracy and roughly 2× faster inference.

## Related Work
- **Reconstruction-based multi-person methods** (Belagiannis et al. [1,2]; Ershadi-Nasab et al. [9]; Dong et al. [6]; Chen et al. [4]; Huang et al. [14]; Lin & Lee [26]; Kadkhodamohammadi & Padoy [21]): detect 2D poses per view, then aggregate via 3D pictorial structures, CRF, or triangulation; accurate but computationally expensive and person-count-dependent.
- **Volumetric approach — VoxelPose** (Tu et al. [40]): back-projects heatmaps into a 3D feature volume, runs per-person proposal + regression; the primary direct baseline MvP improves upon by 9.8 AP25 while being ~2× faster.
- **Single-person multi-view methods** (Iskakov et al. [16] learnable triangulation; Remelli et al. [37] camera-disentangled representation; He et al. [13] epipolar transformers; Qiu et al. [36] cross-view fusion; Pavlakos et al. [34] pictorial structures): strong geometric priors but not designed for multi-person.
- **Monocular 3D pose** (Martinez et al. [29]; Mehta et al. [30]; Sun et al. [38]; Zhou et al. [49]; Zhang et al. [46]; Gong et al. [10]): ill-posed without multi-view; positioned as motivation for the multi-view setting.
- **Transformer-based detection — DETR** (Carion et al. [3]); **Deformable DETR** (Zhu et al. [51]): provide the query-embedding + Hungarian matching paradigm that MvP adapts to 3D pose; MvP's projective attention extends deformable attention with explicit 3D-to-2D geometric projection.
- **Deformable convolutions** (Dai et al. [5]; Zhu et al. [50]): inspire the adaptive deformable sampling in projective attention.
- **ViT / Transformers for vision** (Dosovitskiy et al. [7,8]; Jiang et al. [18]; Wang et al. [42]): demonstrate transformer effectiveness on vision tasks, motivating MvP's architecture.
- **Body mesh recovery — SMPL** (Loper et al. [28]); **HMR** (Kanazawa et al. [22]); **BMP** (Zhang et al. [47]); **Coherent reconstruction** (Jiang et al. [17]): provide the mesh extension baseline and adversarial loss design.
- **2D pose backbone** (Xiao et al. [45] SimpleBaseline on ResNet-50 [12]): used as the multi-view feature extractor.

## Methods & Architecture
MvP is a transformer decoder that maps learnable joint query embeddings and multi-view CNN features directly to 3D joint coordinates and confidence scores for all persons simultaneously.

**Feature extraction:** A SimpleBaseline/ResNet-50 backbone produces high-resolution feature maps {Z_v} for each camera view v. RayConv then concatenates per-pixel camera ray directions R_v (derived from intrinsic/extrinsic parameters) channel-wise to Z_v and applies a standard convolution to yield Ẑ_v = Conv(Concat(Z_v, R_v)), injecting view-dependent 3D positional geometry into every spatial feature.

**Hierarchical query embeddings:** Rather than one independent vector per joint-per-person (M = NJ vectors of size C), MvP maintains N person-level queries {h_n} and J joint-level queries {$l_j$}. The query for joint j of person n is q^n_j = h_n + l_j, reducing parameters from NJC to (N+J)C and enabling joint-level knowledge sharing. An input-dependent adaptation augments each query with a globally pooled scene feature g = Concat(Pool(Z_1),…,Pool(Z_V))W^g so that q^n_j = g + h_n + l_j.

**Transformer decoder:** L=6 stacked decoder layers each apply: (1) self-attention between all NJ joint queries (captures person-joint structural relations); (2) projective attention for multi-view feature fusion; (3) a feed-forward network that outputs 3D joint offset and confidence score. Each layer refines the 3D positions from the previous layer (multi-layer progressive regression).

**Projective attention:** Given current 3D joint estimate y and query feature q, the 2D anchor p_v = Π(y, C_v) is computed via perspective projection for each view. K=4 deformable sampling points are placed around each anchor with learned offsets Δp. View-specific features f_v are aggregated with learned attention weights a = Softmax((q + Z_v(p))W_a) and offsets Δp = (q + Z_v(p))W_p. The final fused feature is PAttention(q, y, {Z_v}) = Concat(f_1,…,f_V)W_P.

**Training — grouped Hungarian matching:** M=NJ predictions are grouped into N consecutive per-person poses {Y_n}. A bipartite matching minimizes L_match = −p_i + L1(Y*_n, Y_σ(n)) via the Hungarian algorithm. The Hungarian loss combines focal loss for confidence and L1 for 3D joints and their 2D projections in all views, summed across all L decoder layers.

**SMPL extension:** Per-person features (average-pooled joint features) feed a small FFN predicting SMPL parameters, trained with L1 on 3D/2D joints and an adversarial loss.

## Experiments & Results
**Panoptic dataset** (5 HD cameras, AP and MPJPE metrics):
- MvP: AP25 = 92.3%, AP50 = 96.6%, AP100 = 97.5%, MPJPE = 15.8 mm, inference = 170 ms
- VoxelPose [40]: AP25 = 84.0%, MPJPE = 17.8 mm, inference = 320 ms
- MvP improvement over VoxelPose: +9.8 AP25, −2.0 mm MPJPE, ~1.9× faster

**Inference time scaling:** MvP is constant at ~170 ms regardless of person count (185 ms for 100 persons); VoxelPose scales linearly (260 ms at 4 persons → 1156 ms at 20 persons).

**Shelf dataset** (PCP, indoor): MvP average PCP = 97.4 vs VoxelPose 97.0; best on all three actors.

**Campus dataset** (PCP, outdoor): MvP average PCP = 96.6 vs VoxelPose 96.7; comparable without any intermediate task.

**Ablations (all on Panoptic):**
- Removing RayConv: AP25 drops 87.5 (−4.8), MPJPE rises to 17.4 (+1.6)
- Per-joint query (no hierarchy): AP25 = 67.4, MPJPE = 41.2; adding hierarchy: AP25 = 82.5, MPJPE = 19.5; adding adaptation: AP25 = 92.3, MPJPE = 15.8
- Decoder layers 2→6→7: MPJPE 49.6 → 15.8 → 15.9 (6 layers optimal)
- Cameras 1→5: AP25 4.7 → 92.3 (monotonically increasing)
- Deformable points K=1→4→8: AP25 88.6 → 92.3 → 84.4 (K=4 optimal)

## Conclusion
MvP demonstrates that multi-view multi-person 3D pose estimation can be solved as a single-stage direct regression problem using a transformer decoder, eliminating the need for heatmap back-projection, cross-view matching, or per-person volumetric processing. The combination of hierarchical query embeddings (with input-adaptive scene conditioning), geometrically-guided projective attention, and RayConv yields state-of-the-art accuracy on Panoptic while halving inference time versus VoxelPose. The authors acknowledge that MvP requires sufficient training data (learns 3D geometry implicitly) and struggles with cross-camera generalization to novel viewpoints.

## Impact
MvP establishes that query-based transformer decoding, originally proven for 2D object detection (DETR), transfers naturally and powerfully to the 3D multi-view domain when coupled with explicit camera geometry. The projective attention and RayConv operations are generic multi-view primitives that should benefit other 3D vision tasks (depth estimation, 3D object detection, dense mesh recovery). The constant-time scaling with person count is practically significant for crowded-scene applications. The main open problems — data efficiency and cross-camera generalization — are well-defined and actively studied, making MvP a solid foundation for follow-on work on generalizable multi-view 3D understanding.


%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Obsidian Notes

<!-- Personal notes not tied to a specific highlight. Survives re-imports. -->
hhhhhhh
%% end obsidian-freeform-notes %%

### From Zotero

---

![[source-notes/wangDirectMultiviewMultiperson2021/image-4-x100-y592.png]]
- **My Note:** Pipeline%% begin 6GTGC29L %%

%% end 6GTGC29L %%

---
> [!note] Highlight (Page 5)
> It adopts a convolution neural network, designed for 2D pose estimation
- **My Note:** CNN for each 2D image view%% begin M8Y9YBVS %%

%% end M8Y9YBVS %%

%% Import Date: 2026-05-22T21:33:31.576-04:00 %%
