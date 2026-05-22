# [Multiple View Geometry Transformers for 3D Human Pose Estimation](zotero://select/library/items/BT7758BP)

 **Authors:** Ziwei Liao, Jialiang Zhu, Chunyu Wang, Han Hu, Steven L. Waslander
 **Published:** 2024
 **Journal/Conference:** 2024 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)
**Citekey:** `liaoMultipleViewGeometry2024`

---
## Abstract
> [!abstract]
>  ```
>  In this work, we aim to improve the 3D reasoning ability of Transformers in multi-view 3D human pose estimation. Recent works have focused on end-to-end learning-based transformer designs, which struggle to resolve geometric information accurately, particularly during occlusion. Instead, we propose a novel hybrid model, MVGFormer, which has a series of geometric and appearance modules organized in an iterative manner. The geometry modules are learning-free and handle all viewpoint-dependent 3D tasks geometrically which notably improves the model’s generalization ability. The appearance modules are learnable and are dedicated to estimating 2D poses from image signals end-to-end which enables them to achieve accurate estimates even when occlusion occurs, leading to a model that is both accurate and generalizable to new cameras and geometries. We evaluate our approach for both indomain and out-of-domain settings, where our model consistently outperforms state-of-the-art methods, and especially does so by a significant margin in the out-of-domain setting. We will release the code and models: https: //github.com/XunshanMan/MVGFormer.
>  ```

---
## Summary

## Introduction
Estimating 3D human pose from multiple calibrated cameras is a long-standing problem with applications in motion capture, sports, and human-scene interaction, where correspondence across views and 2D-to-3D reasoning must be solved jointly. State-of-the-art accuracy on benchmarks like CMU Panoptic, Shelf, and Campus has been pushed forward by large neural networks that learn priors capable of handling severe occlusion. However, those learned priors overfit to the training camera rigs and fail when the test-time viewpoints, camera count, or scenes differ from training. The paper asks which sub-tasks should stay geometric and learning-free, and which should remain learned, to recover generalization without giving up occlusion robustness.

## Related Work
- **Geometric / triangulation-based multi-view pose** (Dong et al. [8, 9], Perez-Yus & Agudo [27], Chen et al. [7], Bartol et al. [1], Iskakov et al. [16]): detect 2D poses per view, solve cross-view associations with hand-crafted affinity matrices, then triangulate. They generalize across cameras but break under occlusion because 2D detections are unreliable; Bartol et al. only handle single-person scenes.
- **Volumetric methods** (VoxelPose [31], Faster VoxelPose [35], VoxelTrack [38], VTP [6], Graph3D [32], Lin et al. [20]): unproject 2D image features into a discretized 3D volume and apply 3D CNNs. Robust to occlusion but suffer ~30 mm quantization error, high memory cost, and a learned dependence on training camera angles that hurts generalization; voxel rotation augmentations [16] only partially fix this.
- **Transformers for 3D detection in autonomous driving** (DETR [5], BEVFormer [19], BEVFusion [23], BEVerse [39], PETR/PETRv2 [21, 22], Voxel Set Transformer [14], 3DETR [24]): extend DETR-style queries to multi-camera 3D detection, often via BEV representations. Not directly transferable to multi-person pose because of quantization and because position-encoded camera params overfit to seen rigs.
- **MvP [36]** (closest prior work): DETR-style end-to-end multi-view multi-person 3D pose with a RayConv that bakes camera parameters into image features. Strong in-domain, but its accuracy collapses to ~0 under unseen camera arrangements — the empirical motivation for MVGFormer's hybrid design.
- **Pose tracking with video / 4D association** [29, 37, 38]: out of scope; this paper handles single-frame inference and leaves tracking to future work.

## Methods & Architecture
**MVGFormer** is a DETR-style decoder stack that interleaves a learned Appearance Module (AM) and a learning-free Geometry Module (GM) over `N = 4` decoder layers. Inputs are `T` synchronized images with known projection matrices `Πt`; a frozen ResNet-50 (pre-trained for 2D pose on COCO) produces per-view feature maps `Mt`.

**Compositional queries.** Each query `Qk = (Fk, Pk)` represents one person. `Pk ∈ R^{J×3}` stores 3D joint positions, initialized by placing T-poses at `K = 1024` (64×64) uniformly sampled ground-plane centers within the capture volume. `Fk ∈ R^{J×L}` uses a hierarchical embedding `fkj = hk + gj` (instance embedding + joint embedding), as in MvP.

**Appearance Module (per joint, per view).** Project the current 3D joint `p` into view `t` with `ut = Πt[p;1]`, then apply Deformable-DETR–style projective attention around `ut` on `Mt` to get attention features `st`. An MLP `gθ(st)` outputs a 2D residual `Δut` and a per-view confidence `ct`; refined 2D position is `u'_t = ut + Δut`. `Δut` is supervised with projected ground-truth 2D poses; `ct` is supervised only indirectly via the downstream 3D loss.

**Geometry Module (per query).** With `{u'_t, ct}` over `T` views, compute `p' = Triangulate({u'_t}, {ct}, {Πt})` using differentiable algebraic triangulation [16]. The confidences `ct` down-weight occluded views, so triangulation acts as a denoiser. The geometry term is updated with `p'`; the appearance term is updated by fusing `{st}` — `f' = fγ(f + fα(Mean({st})))` — chosen over attention-based fusion in ablations.

**Query filtering.** An MLP classifier `fβ` scores each query from its appearance term; queries with score < `ε = 0.1` are dropped layer-by-layer, and NMS keeps the top-scoring query per cluster at output.

**Training.** Fully end-to-end and differentiable. KNN-based anchor matching with `W = 5` assigns each ground-truth pose to its 5 nearest initialized queries (multiple-to-one beats Hungarian, which did not converge here). Losses: L1 on 3D joints + L1 on per-view projected 2D joints for positive queries, plus binary cross-entropy on `fβ` for all queries, enforced at every decoder layer. Backbone frozen; rest trained 40 epochs at lr `4e-4`.

## Experiments & Results
Datasets: **CMU Panoptic** [17], **Shelf** [2], **Campus** [2]. Baselines: **VoxelPose** [31] (volumetric), **MvP** [36] (DETR-based), **Dong et al.** [8] (geometric two-stage). Metrics: AP25 / mAP / MPJPE on CMU Panoptic; PCP on Shelf and Campus.

**Cross-cameras (train CMU0 with 5 cams, test with 3/4/6/7 cams; Table 1).** Average AP25 / mAP — Dong et al. 13.3 / 41.6; VoxelPose 69.1 / 89.8; MvP 21.6 / 35.4; **Ours 83.3 / 94.1**. MvP collapses to 0.0 AP25 when cameras are added (6 or 7), because the new rays were never seen; MVGFormer is the only method that benefits from added unseen cameras.

**Cross-arrangements (train CMU0, test CMU1–CMU4; Table 2).** Average AP25 / mAP — Dong et al. 0.0 / 32.8; VoxelPose 29.4 / 78.5; **MvP 0.0 / 0.0** (complete failure on every arrangement); **Ours 74.7 / 90.6**. Best gains on CMU1 (86.8 AP25) and CMU4 (91.5 AP25).

**Cross-datasets, no fine-tune (Table 3, PCP).** Shelf — Dong 96.9, VoxelPose 69.8, MvP 8.7, **Ours 87.9**. Campus — Dong 96.3, VoxelPose 11.2, MvP 0.0, **Ours 58.1** (large outdoor appearance gap noted as the main limitation). With fine-tuning, Ours reaches 98.0 PCP on Shelf and 96.7 on Campus, matching or beating all baselines.

**In-domain (Table 4).** CMU Panoptic AP25 / MPJPE — VoxelPose 84.0 / 17.7 mm, MvP 92.3 / 15.8 mm, **Ours 92.3 / 16.0 mm**. Shelf PCP — **Ours 98.0** (best). Campus PCP — Ours 96.7 (ties VoxelPose).

**Ablations (Table 5).** AP25 climbs 77.9 → 86.8 → 90.7 → **92.3** → 92.0 as queries go 128 → 256 → 512 → 1024 → 1280. Decoder layers: 1 → 4 gives 5.2 → 79.2 → 88.6 → **92.3** AP25 (saturates beyond 4). KNN `W = 5` is optimal (86.0 → **92.3**); Hungarian one-to-one did not converge. Feature fusion: MLP **92.3** beats attention 90.4 and naive mean 36.2. Person-level embedding (**92.3**) beats joint-level (90.9) and human-level (91.0). NMS lifts AP25 from 20.0 → **92.3**. Inference: 0.21 s/frame on a V100 (vs. VoxelPose 0.29 s, MvP 0.17 s).

## Conclusion
MVGFormer is a hybrid coarse-to-fine Transformer for multi-view 3D human pose that explicitly decouples 2D appearance from 3D geometry: a learned Appearance Module refines 2D joint positions from image features, and a learning-free Geometry Module recovers 3D joints via differentiable triangulation, with the two iterating inside the decoder stack. This decoupling preserves the occlusion robustness of end-to-end learning while removing the camera-rig dependence that hobbles MvP and VoxelPose, and it sidesteps the quantization error of voxel methods. The model reaches state-of-the-art or comparable in-domain accuracy on CMU Panoptic, Shelf, and Campus, and a large generalization margin when tested on unseen camera counts, arrangements, and scenes.

## Impact
The headline contribution is empirical evidence that "keep the projection geometry hard-coded, learn only the appearance" is a viable design pattern for multi-view perception — not a compromise but an upgrade, since it both fits in-domain data and transfers across rigs. For practitioners building motion-capture or sports-tracking systems, this is the closest the field has come to "train once, deploy anywhere," which materially reduces the per-deployment data-collection burden that has limited learned multi-view methods. The approach is general — the authors suggest extending it to hand, face, and shape estimation — and the same decoupling principle plausibly transfers to autonomous-driving BEV pipelines that currently bake calibration into learned features (PETR, BEVFormer). Limitations remain: per-frame only (no temporal tracking), large appearance domain gaps still hurt (Campus zero-shot at 58.1 PCP), and inference is 0.21 s/frame, leaving real-time use as future work. Most importantly, the result reframes the generalization failures of MvP-style models as an architectural choice rather than a data problem, which should redirect work toward hybrid designs instead of larger end-to-end transformers.

## Notes & Highlights



> [!quote] Highlight (Page 710)
> Most closely related to our work, MvP [36] extends DETR for multi-view 3D human pose estimation. Similar to PETR [21, 22], it introduces a RayConv operation to integrate the camera parameters into the image features. Although it achieves good in-domain performance when cameras are the same for training and testing, it does not generalize well to different camera arrangements.



- **My Note:** Against MvP


