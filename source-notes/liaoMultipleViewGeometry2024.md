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

%% begin notes %%
## Summary
## Introduction
Multi-view 3D human pose estimation is critical for applications ranging from motion capture to autonomous systems, yet recent end-to-end transformer-based approaches fail to generalize to unseen camera configurations — their performance can degrade to near-zero when tested on cameras not seen during training. The core issue is that purely learned methods encode camera-specific geometric priors implicitly in network weights, making them brittle to viewpoint changes and prone to correspondence failures during occlusion. MVGFormer addresses this by explicitly decoupling what should be learned (2D appearance) from what should be computed geometrically (3D reasoning), combining both within a unified, end-to-end differentiable iterative refinement pipeline.

## Related Work
- **Geometric/triangulation-based methods** (Dong et al. [8], Bridgeman et al. [4], Perez-Yus & Agudo [27], Chen et al. [7]): Estimate per-view 2D poses, associate them across views, then triangulate. Generalize well to unseen cameras but break under severe occlusion due to absence of learned priors.
- **Volumetric representation methods** — VoxelPose [31] and variants (Faster VoxelPose [35], VoxelTrack [38], Chen et al. [6]): Project multi-view image features into a voxelized 3D space and apply 3D convolutions. Robust to occlusion but suffer large quantization errors (≈30 mm), high memory cost, and poor generalization to novel cameras due to learned camera-specific visual patterns in voxels.
- **DETR-based transformer methods** — MvP [36]: Extends DETR to multi-view 3D pose with a RayConv operation embedding camera parameters into image features. Achieves strong in-domain performance but essentially memorizes training-camera geometry; accuracy collapses to near-zero on unseen camera arrangements.
- **PETR / PETRv2** [21, 22]: Encodes 3D coordinate position embeddings into DETR for multi-view 3D object detection but requires large data to learn the 2D-to-3D relationship and overfits to training camera configurations.
- **BEV-based 3D detection** (BEVFormer [19], BEVFusion [23], BEVerse [39]): Construct volumetric bird's-eye-view representations for autonomous driving; not suitable for human pose due to quantization errors.
- **Temporal/video-based approaches** (TesseTrack [29], 4D Association [37]): Exploit temporal sequences for multi-person tracking; MVGFormer focuses on single-frame estimation.
- **Learnable triangulation** (Iskakov et al. [16]): Introduces differentiable algebraic triangulation, which MVGFormer directly builds upon for its geometry module.
- **Generalizable triangulation** (Bartol et al. [1]): Evaluates geometric triangulation generalization but only for single-person scenes; MVGFormer extends this to multi-person.

## Methods & Architecture
MVGFormer takes T multi-view images {I_t} as input, extracts features via a ResNet-50 backbone (pre-trained on COCO for 2D pose, then frozen), and maintains K compositional queries {Q_k} iteratively refined through N decoder layers.

**Compositional Query**: Each query Q_k = (F_k, P_k) has an appearance term F_k ∈ R^{J×L} (per-joint feature vectors) and a geometry term P_k ∈ R^{J×3} (3D joint positions). The appearance term uses a hierarchical embedding: K instance embeddings {h_k} plus J joint embeddings {g_j} are summed per joint (f_kj = h_k + g_j), reducing learnable parameters. The geometry term is initialized by placing T-poses at K=1024 uniformly sampled ground-plane centers.

**Appearance Module (AM)**: For each query, the 3D joint position p is projected into each view t via camera projection matrix Π_t to get 2D position u_t. Deformable attention samples features s_t from feature map M_t around u_t. An MLP g_θ predicts a 2D residual Δu_t and confidence score c_t from s_t. The refined 2D position is u'_t = u_t + Δu_t. A multi-view feature fusion MLP fα and fγ updates the appearance term f using the mean of {s_t}.

**Geometry Module (GM)**: Given the refined 2D positions {u'_t} and confidence scores {c_t} from AM, differentiable algebraic triangulation (following [16]) computes a new 3D joint position p'. This is learning-free and purely geometric. The updated 3D position is fed back to project into images in the next decoder layer, enabling coarse-to-fine refinement.

**Query Filtering**: An MLP classifier f_β predicts a validity score per query; queries below threshold ε=0.1 are filtered at each layer. Final outputs use NMS to remove redundant high-scoring queries around the same person.

**Training**: End-to-end with an anchor-based KNN matching strategy (W=5 nearest queries per ground-truth pose). Loss combines L1 3D pose loss, L1 projected 2D pose loss (at every decoder layer), and cross-entropy for the classifier. Trained for 40 epochs, lr=4e-4, 4 decoder layers with independent parameters.

## Experiments & Results
Evaluated on CMU Panoptic [17], Shelf [2], and Campus [2] under both in-domain and out-of-domain settings, against VoxelPose [31], MvP [36], and Dong et al. [8].

**In-domain (Table 4)**: CMU Panoptic — AP25 92.3, MPJPE 16.0 mm (matching MvP at 92.3/15.8 and surpassing VoxelPose at 84.0/17.7). Shelf PCP 98.0% (vs. VoxelPose 97.0%, MvP 97.4%). Campus PCP 96.7% comparable to baselines after finetuning.

**Cross-camera generalization (Table 1)** — trained on CMU0 (5 cameras), tested with K cameras removed/added: Average AP25 83.3 vs. VoxelPose 69.1, MvP 21.6, Dong et al. 13.3. MVGFormer is the only method that benefits from adding unseen cameras (AP25 94.7–95.1 at 6–7 cameras vs. MvP collapsing to 0.0).

**Cross-arrangement generalization (Table 2)** — trained on CMU0, tested on CMU1-4 with entirely different camera numbers and poses: Average AP25 74.7, mAP 90.6 vs. VoxelPose 29.4/78.5, MvP 0.0/0.0, Dong et al. 0.0/32.8. MvP accuracy collapses to zero on all unseen arrangements.

**Cross-dataset generalization without finetuning (Table 3)**: Shelf average PCP 87.9% vs. MvP 8.7%, VoxelPose 69.8%; Campus 58.1% vs. MvP 0.0%, VoxelPose 11.2%. After finetuning: Shelf 98.0%, Campus 96.7%.

**Ablation highlights**: Query count 1024 optimal; 4 decoder layers best; W=5 KNN assignment outperforms Hungarian matching; MLP feature fusion beats attention; NMS is essential (AP25 without NMS: 20.0 vs. 92.3 with NMS).

**Inference speed**: 0.21s per frame on V100 (vs. VoxelPose 0.29s, MvP 0.17s).

## Conclusion
MVGFormer demonstrates that explicitly decoupling 2D appearance estimation (learned) from 3D pose recovery (geometry-only triangulation) within an iterative transformer decoder achieves state-of-the-art in-domain accuracy while dramatically improving generalization to unseen camera numbers, arrangements, and datasets. The geometry module's learning-free triangulation is immune to quantization errors and camera-specific overfitting, while the appearance module's cross-view feature fusion handles occlusion robustly. The paper claims that this hybrid design can train once and deploy to new camera setups without retraining, a capability that pure learning-based methods fundamentally lack.

## Impact
MVGFormer directly challenges the prevailing assumption that end-to-end learned transformers are the best path forward for multi-view pose estimation — it shows that injecting classical geometric reasoning as a non-learned module inside a modern architecture is not a regression but an improvement, especially for real-world deployment where fixed training camera rigs are rarely available. The approach is practically significant for any multi-camera system that needs to be reconfigured or redeployed (sports analytics, surveillance, robotics), making "train once, deploy anywhere" a realistic goal. Its limitation is that it still requires known camera intrinsics and extrinsics at test time; extending to uncalibrated settings or dynamic camera rigs remains open. The iterative query-refinement paradigm with compositional geometry/appearance queries is likely to influence future work on multi-view keypoint estimation beyond the human body (hands, faces, objects).
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

---
> [!note] Highlight (Page 708)
> The geometry modules are learning-free and handle all viewpoint-dependent 3D tasks geometrically which notably improves the model’s generalization ability.%% begin JE57L372 %%%% end JE57L372 %%

---
> [!note] Highlight (Page 708)
> The appearance modules are learnable and are dedicated to estimating 2D poses from image signals end-to-end which enables them to achieve accurate estimates even when occlusion occurs%% begin H57ZBA5V %%%% end H57ZBA5V %%

---

![[source-notes/liaoMultipleViewGeometry2024/image-undefined-x305-y310.png]]
- **My Note:** Appearance Module (AM)
Geometric Module (GM)%% begin 4KTSTA8J %%%% end 4KTSTA8J %%

---
> [!info] Highlight (Page 708)
> What sub-tasks should be addressed geometrically in a learning-free style rather than by neural networks?
- **My Note:** The core question they try to address%% begin F8JWP9GK %%%% end F8JWP9GK %%

---
> [!danger] Highlight (Page 708)
> they struggle to handle severe occlusion due to  This CVPR paper is the Open Access version, provided by the Computer Vision Foundation.  Except for this watermark, it is identical to the accepted version; the final published version of the proceedings is available on IEEE Xplore. the lack of effective priors leading to correspondence failures
- **My Note:** for geometric approach%% begin 67M782YA %%%% end 67M782YA %%

---
> [!danger] Highlight (Page 709)
> learned approaches tend to obtain poor results on new scenes, especially when the testing camera configurations are not seen in training.%% begin XGVE4BXU %%%% end XGVE4BXU %%

---

![[source-notes/liaoMultipleViewGeometry2024/image-3-x42-y574.png]]
- **My Note:** Framework%% begin AZB6QAA5 %%%% end AZB6QAA5 %%

---
> [!note] Highlight (Page 710)
> Most closely related to our work, MvP [36] extends DETR for multi-view 3D human pose estimation. Similar to PETR [21, 22], it introduces a RayConv operation to integrate the camera parameters into the image features. Although it achieves good in-domain performance when cameras are the same for training and testing, it does not generalize well to different camera arrangements.
- **My Note:** Against MvP%% begin 97GZMPM7 %%%% end 97GZMPM7 %%

%% Import Date: 2026-05-25T19:20:43.483-04:00 %%
