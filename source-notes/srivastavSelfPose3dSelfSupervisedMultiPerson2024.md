# [SelfPose3d: Self-Supervised Multi-Person Multi-View 3d Pose Estimation](zotero://select/library/items/J2IQRCEZ)

 **Authors:** Vinkle Srivastav, Keqi Chen, Nicolas Padoy
 **Published:** 2024
 **Journal/Conference:** 2024 IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)
**Citekey:** `srivastavSelfPose3dSelfSupervisedMultiPerson2024`

---
## Abstract
> [!abstract]
>  ```
>  We present a new self-supervised approach, SelfPose3d, for estimating 3d poses of multiple persons from multiple camera views. Unlike current state-of-the-art fullysupervised methods, our approach does not require any 2d or 3d ground-truth poses and uses only the multiview input images from a calibrated camera setup and 2d pseudo poses generated from an off-the-shelf 2d human pose estimator. We propose two self-supervised learning objectives: self-supervised person localization in 3d space and self-supervised 3d pose estimation. We achieve selfsupervised 3d person localization by training the model on synthetically generated 3d points, serving as 3d person root positions, and on the projected root-heatmaps in all the views. We then model the 3d poses of all the localized persons with a bottleneck representation, map them onto all views obtaining 2d joints, and render them using 2d Gaussian heatmaps in an end-to-end differentiable manner. Afterwards, we use the corresponding 2d joints and heatmaps from the pseudo 2d poses for learning. To alleviate the intrinsic inaccuracy of the pseudo labels, we propose an adaptive supervision attention mechanism to guide the self-supervision. Our experiments and analysis on three public benchmark datasets, including Panoptic, Shelf, and Campus, show the effectiveness of our approach, which is comparable to fully-supervised methods. Code is available at https://github.com/CAMMApublic/SelfPose3D.
>  ```

---
## Summary

## Introduction
Multi-person, multi-view 3D human pose estimation from calibrated cameras matters for motion capture, surgical scene understanding, and social-behavior analysis, but the dominant learning-based methods (VoxelPose, MvP, plane-sweep stereo, etc.) all depend on 3D ground-truth poses captured in expensive dense-camera studios like CMU Panoptic. Geometric optimization-based methods avoid 3D labels but trail learned methods badly under occlusion and complex interactions. SelfPose3d asks whether a learned voxel pipeline can be trained with no 2D or 3D ground-truth poses at all — only multi-view images, known camera calibration, and 2D pseudo-poses from an off-the-shelf 2D detector — and still match fully-supervised accuracy.

## Related Work
- **Fully-supervised learning-based 3D pose** — VoxelPose [57] (the chosen backbone), Tessetrack [52], MvP [63], Wu et al. graph-based [59], Lin et al. plane-sweep stereo [39], TEMPO [15], Faster VoxelPose-style volumetric variants [64]; cross-view fusion / epipolar transformers [27, 51] and learnable triangulation [28] for the single-person case. All require 3D GT from dense studios [32], which SelfPose3d removes.
- **Optimization-based geometric methods** — MvPose / Dong et al. [18], Pirinen et al. ACTOR [49], Chen et al. [10], Kadkhodamohammadi & Padoy [33], Ershadi-Nasab et al. [20], 3DPS [2]. These solve cross-view correspondence with hand-crafted affinity matrices and triangulate; they need no 3D GT but trail learned methods badly under occlusion. SelfPose3d is benchmarked directly against ACTOR and MvPose.
- **Self-supervised single-person 3D pose** — multi-view geometric self-supervision (Kocabas et al. EpipolarPose [34]), video-constraint self-supervision (Kundu et al. [38]), adversarial 2D-to-3D lifting (Chen et al. [9], Drover et al. [19], Kudo et al. [36]). All are single-person; none handle multi-person cross-view correspondence — the gap this paper fills.
- **Self-supervised representation learning** (SimCLR [11], MoCo [26], DINO [7], BYOL [54]) is cited as the broader context motivating the approach, not directly compared.
- **Differentiable rendering / learning-by-projection** — DIB-R [12] for the projection-based supervisory paradigm; DSNT-style heatmap encoding [61] for differentiably rendering 2D joints into heatmaps online.
- **2D pose detectors used to produce pseudo-labels** — Mask R-CNN [25] for boxes, HRNet [55] for 2D keypoints; Keypoint R-CNN [25] is also tested.

## Methods & Architecture
**SelfPose3d** keeps the VoxelPose architecture (`heatmap_net2d` 2D backbone → unprojection to 3D feature volume → `root_net` for person localization → `pose_net3d` for per-person 3D joints) but replaces all 3D supervision with two self-supervised objectives plus an attention-weighted pseudo-label loss.

**Pseudo 2D pose generation.** Mask R-CNN detects person boxes; HRNet (w48, 384×288) regresses 2D joints inside each box. `heatmap_net2d` is pre-trained on these pseudo poses for 20 epochs with Adam, lr 1e-4 (→ 1e-5 at epoch 10, → 1e-6 at epoch 15).

**Self-supervised 3D root localization.** Build a synthetic dataset `D_root`: place random 3D points in world space, project them to each view using known camera parameters, render the projections as 2D Gaussian root-heatmaps. Unproject these synthetic root-heatmaps `H_root^syn` into a 3D volume `F_root^syn` (Eq. 1) and pass through `root_net` to predict `G^syn`; the synthetic-root loss `l_root_syn = L2(G^syn, G^syn*)` (Eq. 2) supervises localization. To regularize against real images, apply two random affine augmentations `(t1_{r,s}, t2_{r,s})` (rotation ∈ [−45°, 45°], scale ∈ [−0.35, 0.35]) to the real multi-view input `x0`, produce `x1, x2`, run all three through `heatmap_net2d` and `root_net` to get `G^0, G^1, G^2`, and enforce the root consistency loss `l_root_C = L2(G^0, G^1) + L2(G^0, G^2)` (Eq. 3). Only **root-heatmaps** (1 channel) are fed to `root_net`, not all-joint heatmaps (15 channels) as in VoxelPose — making it computationally cheaper. NMS + thresholding on `G^2` produces person proposals `{root_i}`.

**Self-supervised 3D pose estimation (cross-affine-view).** For each person proposal, build feature volumes `F_i^1, F_i^2` from the affine-augmented views, run `pose_net3d` + soft-argmax [8] to get bottleneck 3D poses `Y^1, Y^2 ∈ R^{P×J×3}`. **Cross-affine-view projection:** project `Y^1` into the `x2` image space (using `t2_{r,s}`) and `Y^2` into the `x1` image space, yielding 2D poses `y^1, y^2`. Render these into 2D heatmaps `H^1, H^2` using a differentiable Zhang-et-al. [61]-style floating-point encoding (no quantization, gradients flow). Pseudo 2D poses `y^*_{2d}` are affine-transformed to produce targets `y^{1*}_{2d}, y^{2*}_{2d}` and `H^{1*}, H^{2*}`. Losses: heatmap MSE `l_pose_H = L2(H^1, H^{1*}) + L2(H^2, H^{2*})` (Eq. 4) and joint L1 `l_pose_J = L1(y^1, y^{1*}_{2d}) + L1(y^2, y^{2*}_{2d})` after Hungarian matching [37] of predictions to pseudo-labels (Eq. 5). Combined: `l_pose_3d = l_pose_H + λ·l_pose_J` (Eq. 6).

**Adaptive supervision attention** handles pseudo-label noise from occlusion. **L2 / soft attention:** a ResNet-18 + deconv `attn_net2d` outputs per-view attention heatmaps `A`; the per-element product `A ⊗ (H − H*)^2` replaces plain MSE (Eq. 7). A regularizer `l_attn = L2(A, 1)` (Eq. 8) prevents `A` from collapsing to zero. **L1 / hard attention:** per multi-view sample, compute per-view L1 loss, drop the worst view, average the rest (Eqs. 9–10). Final objective: `l^attn_pose_3d = l^attn_pose_H + λ·l^attn_pose_J + σ·l_attn` (Eq. 11), with `λ = 0.01`, `σ = 0.1`.

**Training schedule.** (1) 20 epochs `heatmap_net2d` on pseudo poses; (2) 1 epoch `root_net`; (3) 5 epochs end-to-end with L2 only (lr 1e-4); (4) 5 epochs end-to-end with L1+L2 (lr 5e-5). Spatial augmentation via rand-augment [16] (contrast, color, sharpness, brightness jitter; auto-contrast; equalize) and rand-cutout [17] (20–40 px boxes). SMPL fitting [4, 41] is applied post-hoc for visualization only.

## Experiments & Results
**Datasets:** Panoptic [32] (9 training sequences, 5 HD cameras — IDs 3, 6, 12, 13, 23), Shelf [1], Campus [1]. **Metrics:** AP25/AP50/AP100/AP150, Recall@500, MPJPE (mm) on Panoptic; PCP per actor on Shelf/Campus.

**Panoptic (Table 1).** SelfPose3d gets **AP25 55.1 / AP50 96.4 / AP100 98.5 / AP150 99.0 / Recall@500 99.6 / MPJPE 24.5 mm**. The fully-supervised reference VoxelPose [57]: AP25 83.6 / AP50 98.3 / MPJPE 17.7. MvP [63]: AP25 92.3 / MPJPE 15.8. TEMPO [15]: MPJPE 14.7. Optimization-based MvPose [18]: AP25 0.0 / AP50 2.97 / MPJPE 84.2 — SelfPose3d crushes the optimization baselines by tens of AP points and improves MPJPE from 84.2 → 24.5. Vs. fully-supervised VoxelPose, the gap is ~1.9 AP50 and ~6.8 mm MPJPE.

**Shelf and Campus (Table 2, PCP).** Using pseudo 3D poses produced by running SelfPose3d on Panoptic, then training on Shelf/Campus: Shelf actors 97.2 / 90.3 / 97.9 (avg **95.1**); Campus actors 92.5 / 82.2 / 89.2 (avg **87.9**). Compares with fully-supervised VoxelPose Shelf avg 97.0 / Campus avg 96.7; MvP Shelf 97.4 / Campus 96.6; reproduced VoxelPose∗ Campus 90.9. Optimization-based MvPose: Shelf 96.9 / Campus 96.3; 3DPS [2]: Shelf 77.5 / Campus 84.5; Ershadi et al. [20]: Shelf 88.0 / Campus 90.6.

**Ablations on Panoptic.**
- *GT vs. pseudo 2D poses (Table 3):* swapping pseudo for GT 2D poses improves AP50 96.4 → 98.8 and MPJPE 24.5 → 19.9, showing pseudo-label noise (mostly occlusion-driven, see Figure 3) is the main residual gap.
- *Affine augs + cross-affine-view consistency (Table 4):* base 86.0 AP50; +affine augs alone 83.3; +cross-affine-view consistency 93.8 AP50 / MPJPE 34.7 → 29.3 — the consistency constraint is doing real work.
- *L1+L2 vs. solo losses (Table 5):* L1 alone doesn't converge (noise); L2 alone gets AP25 43.8; L1+L2 reaches AP25 55.1, MPJPE 24.5.
- *Attention ablation (Table 6):* no attention AP25 32.5; L1-only attention 37.9; L2-only attention 47.4; both 55.1.
- *2D detector quality (Table 7):* HRNet w48 (76.3 COCO AP) yields Panoptic AP50 93.8 / MPJPE 29.3 vs. Keypoint R-CNN R-101 (66.1 COCO AP) yielding 89.2 / 31.9 — pseudo-label quality propagates roughly linearly.

## Conclusion
SelfPose3d is the first self-supervised multi-view multi-person 3D pose estimator built on a learning-based volumetric backbone, replacing 3D ground-truth supervision with (i) synthetic-root supervision for the localization head, (ii) differentiable cross-affine-view projection of bottleneck 3D poses back into 2D heatmaps and joints for the pose head, and (iii) an adaptive attention mechanism that down-weights occluded pseudo-labels. It matches fully-supervised baselines at coarse thresholds (AP50/AP100/PCP) and decisively beats optimization-based methods at all thresholds across Panoptic, Shelf, and Campus, while needing only multi-view RGB, camera calibration, and an off-the-shelf 2D detector.

## Impact
The headline result is that the expensive bottleneck of multi-view 3D pose research — dense camera studios for 3D GT — can be eliminated for a large class of methods by combining synthetic-root pretraining with differentiable cross-view consistency on bottleneck 3D outputs. This is particularly valuable for the surgical / operating-room setting that motivates the lab (IHU Strasbourg, CAMMA), where collecting 3D GT is infeasible but multi-camera RGB rigs are common. The residual gap at fine thresholds (AP25 55.1 vs. fully-supervised 83.6) and the Campus drop (87.9 vs. 96.7 PCP) shows the method still inherits the failure modes of its 2D pseudo-detector — work on better occlusion-aware pseudo-labeling or temporal consistency would directly close the gap. More broadly, the "render bottleneck 3D back into 2D under augmentation and supervise there" template is a clean recipe that could transfer to other multi-view 3D tasks where dense labels are hard (hand pose, object pose, animal pose), and reframes self-supervised 3D pose as a problem of designing better projection consistency losses rather than chasing larger labeled datasets.

## Notes & Highlights

