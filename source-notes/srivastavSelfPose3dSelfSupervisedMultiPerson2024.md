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

%% begin notes %%
## Summary
## Introduction
Estimating 3D poses of multiple people from a few calibrated cameras is a hard problem because the same person must be identified and matched across different viewpoints while also inferring depth. Existing learning-based methods achieve strong results but depend on 3D ground-truth poses that are expensive to collect, typically requiring dense camera domes. SelfPose3d eliminates this dependency entirely, requiring only multi-view RGB images and 2D pseudo poses from an off-the-shelf detector, making the approach practical for settings where 3D annotation is infeasible, such as operating rooms and outdoor sports venues.

## Related Work
- **VoxelPose (Tu et al., ECCV 2020)** — the volumetric, fully-supervised backbone that SelfPose3d directly extends; uses 3D ground-truth root joints for localization and full skeleton supervision.
- **MvP (Zhang et al., NeurIPS 2021)** — direct regression with transformers for multi-view multi-person pose; fully supervised, achieves 92.3 AP25 / 15.8 MPJPE on Panoptic.
- **Lin et al. (CVPR 2021)** — plane-sweep stereo approach; fully supervised, 92.1 AP25 / 16.8 MPJPE on Panoptic.
- **Wu et al. (ICCV 2021)** — graph-based multi-person 3D pose using multi-view images; fully supervised, 15.8 MPJPE on Panoptic.
- **TEMPO (Choudhury et al., ICCV 2023)** — efficient multi-view pose estimation, tracking, and forecasting; fully supervised, 14.7 MPJPE on Panoptic.
- **MvPose (Dong et al., CVPR 2019)** — optimization-based fast multi-person 3D pose from multiple views; 84.2 MPJPE on Panoptic, 96.3 avg PCP on Shelf.
- **ACTOR (Pirinen et al., NeurIPS 2019)** — self-supervised active triangulation using drones; 168.4 MPJPE on Panoptic.
- **Kocabas et al. (CVPR 2019)** — self-supervised single-person 3D pose using multi-view geometry constraints; motivates the geometric self-supervision paradigm.
- **Chen et al. (ECCV 2020)** — optimization-based crowded-scene multi-person 3D pose using multi-view geometry.
- **HRNet (Sun et al., CVPR 2019)** — used as the off-the-shelf 2D pseudo pose generator; w48 384x288 variant achieves 76.3 keypoint AP on COCO-val.
- **Mask R-CNN (He et al., ICCV 2017)** — used for person bounding box detection before HRNet.
- **SMPL (Loper et al., TOG 2015)** — body mesh model used to fit meshes onto estimated 3D poses for qualitative evaluation.

## Methods & Architecture
SelfPose3d extends VoxelPose without modifying its network structure. The pipeline has four learned modules: heatmap net2d (backbone for 2D joint heatmaps), attn net2d (ResNet-18 + deconv layers for soft attention heatmaps), root net (3D root localization), and pose net3d (3D pose estimation via soft-argmax).

**Pseudo 2D pose generation:** Mask R-CNN detects person bounding boxes; HRNet-w48 generates 2D joint heatmaps y* for each box. heatmap net2d is pretrained with these pseudo labels.

**Self-supervised 3D root localization:** A synthetic dataset Droot is created by randomly placing 3D points in world space, projecting them to each view with camera parameters, and rendering projected 2D points as root heatmaps. root net is trained with synthetic root loss l_root_syn (L2 between predicted and ground-truth synthetic root volumes). A root consistency loss l_root_C enforces invariance of root predictions under two random affine augmentations (t1_r,s and t2_r,s, parameterized by rotation and scale) applied to the multi-view input images.

**Self-supervised 3D pose estimation:** Given person proposals from root net, 3D feature volumes F1_i and F2_i are constructed (via unprojection of 2D heatmaps) for each affine-augmented image pair (x1, x2). pose net3d + soft-argmax produces bottleneck 3D poses Y1 and Y2. Cross-affine-view consistency: Y1 is projected into x2's image space (using t2_r,s) to generate 2D poses y2; Y2 is projected into x1's image space (using t1_r,s) to generate y1. These are rendered into heatmaps H1, H2 using a differentiable Gaussian rendering step (avoiding non-differentiable quantization). Affine transformations are also applied to pseudo 2D poses to produce y1*_2d, y2*_2d and heatmaps H1*, H2*. Pose heatmap loss l_pose_H (L2 between H1,H1* and H2,H2*) provides initial training; then L1 joint loss l_pose_J (via Hungarian matching) is added for fine-tuning. Combined: l_pose_3d = l_pose_H + λ·l_pose_J.

**Adaptive supervision attention:** Addresses noisy pseudo labels from occlusion. Soft attention (L2 supervision): attn net2d outputs attention heatmaps A; the attentive heatmap loss is l^attn_pose_H = (1/N) Σ A_i ⊗ (H_i − H_i*)^2. A regularization loss l_attn = L2(A, 1) prevents A from collapsing to zero. Hard attention (L1 supervision): for K camera views, the view with the highest per-view L1 loss is dropped; l^attn_pose_J averages only the remaining K−1 views. Final loss: l^attn_pose_3d = l^attn_pose_H + λ·l^attn_pose_J + σ·l_attn (λ=0.01, σ=0.1).

**Training:** heatmap net2d trained for 20 epochs with pseudo labels (Adam, lr 1e-4 → 1e-5 → 1e-6). root net trained 1 epoch. End-to-end with L2-only loss for 5 epochs (lr 1e-4), then adding L1 loss for 5 more epochs (lr 5e-5). Random affine augmentation: rotation ±45°, scale ±0.35; also rand-augment and rand-cutout.

## Experiments & Results
**Panoptic dataset** (5 HD cameras, 9 training sequences, metrics: AP at 25/50/100/150mm thresholds, Recall@500mm, MPJPE in mm):
- SelfPose3d (self-supervised): AP25=55.1, AP50=96.4, AP100=98.5, AP150=99.0, Recall@500=99.6, MPJPE=24.5mm
- VoxelPose (fully supervised): AP25=83.6, AP50=98.3, MPJPE=17.7mm
- MvP (fully supervised): AP25=92.3, MPJPE=15.8mm
- TEMPO (fully supervised): AP25=89.0, MPJPE=14.7mm
- MvPose (optimization-based): AP50=2.97, AP100=59.93, MPJPE=84.2mm
- ACTOR (optimization-based, self-supervised): MPJPE=168.4mm
- SelfPose3d with ground-truth 2D (upper bound probe): AP50=98.8, AP100=99.6, MPJPE=19.9mm

**Shelf dataset** (PCP metric, using pseudo 3D poses from Panoptic for training):
- SelfPose3d: Actor1=97.2, Actor2=90.3, Actor3=97.9, Average=95.1
- VoxelPose (fully supervised): Average=97.0; VoxelPose* (reproduced): Average=96.9
- MvPose (optimization-based): Average=96.9

**Campus dataset** (PCP metric):
- SelfPose3d: Actor1=92.5, Actor2=82.2, Actor3=89.2, Average=87.9
- VoxelPose (fully supervised): Average=96.7

**Ablations (Panoptic):** Cross-affine-view consistency + affine augmentations: AP50 improves from 86.0 (neither) → 83.3 (affine only) → 93.8 (both). L2+L1 losses together: AP50=96.4, MPJPE=24.5 vs L2-only: AP50=95.8, MPJPE=25.7. Both attention types together: AP25=55.1, MPJPE=24.5 vs neither: AP25=32.5, MPJPE=28.5. HRNet-w48 pseudo poses vs Keypoint R-CNN (R-101): AP50 93.8 vs 89.2, MPJPE 29.3 vs 31.9.

## Conclusion
SelfPose3d demonstrates that a learning-based multi-view multi-person 3D pose estimator can be trained without any 2D or 3D ground-truth annotations, using only multi-view images and off-the-shelf 2D pseudo poses. The approach reaches performance comparable to fully-supervised methods on Panoptic and competitive results on Shelf and Campus, while substantially outperforming prior optimization-based and self-supervised approaches. The adaptive supervision attention mechanism and cross-affine-view consistency objective are shown to be necessary components through ablation. The method also supports SMPL body mesh fitting on its outputs, producing geometrically plausible body shapes.

## Impact
SelfPose3d is directly relevant to domains where annotating 3D poses is prohibitively expensive or logistically impossible, most notably clinical settings such as operating rooms with multi-camera systems. By reducing the annotation requirement to only a calibrated camera rig and an off-the-shelf 2D detector, it opens a viable path to training strong 3D pose models on domain-specific in-situ video. The remaining gap to fully supervised methods (e.g., AP25 of 55.1 vs 83.6 for VoxelPose) suggests pseudo label quality is still a limiting factor, and improvements in 2D estimation or domain-adaptive detectors would directly translate to better 3D performance. The method's inability to handle entirely uncalibrated or moving cameras remains a limitation for truly in-the-wild deployment.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-05-22T21:04:06.167-04:00 %%
