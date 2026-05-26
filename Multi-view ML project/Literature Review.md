# Human Pose Estimation

## [[wangDirectMultiviewMultiperson2021|MvP (2021)]]

![[source-notes/wangDirectMultiviewMultiperson2021/image-4-x100-y592.png]]

- Framework is fairly simple and direct
- Pass each view through a `ResNet 50` backbone to extract **feature only**
- then pass features and joint queries through a 6-layer transformer decoder
	- each layer does
	- self-attention between all joints
	- projective attention to fuse all the views
	- feed-forward network that generate 3D joint offset and confidence score (refines 3D joint position)
- **However**, too old, code base is very hard to build

---
## [[liaoMultipleViewGeometry2024|Geometric Multi-view Transformer]]

![[source-notes/liaoMultipleViewGeometry2024/image-3-x42-y574.png]]

- Each image first feed through a ResNet 50 to extract feature
- Features a `Appearance Module(AM)` and  `Geometry Module(GM)
	- AM: Dedicated to refine 2D pose estimation from image
		1. Projective Attention: Transformer that uses deformable attention to get sample points
		2. MLP: calculates 2D pixel offset and confidence score
	- GM: Dedicated to refine 3D pose estimation from triangulation (*not-learnable*) 
		- Takes in 2D positions and triangulates new 3D positions
- Query: Compositional Query
	- Appearance term: feature vector, tells ML what it is looking for
	- Geometric term: stores the 3D position of the joint

---
## Plan using MVGFormer

### 1. Input modality — RGB to RGB-D

| What | Where | Action | Notes |
|---|---|---|---|
| First conv layer | `MVGFormer/lib/models/pose_resnet.py` (~`:116`, `Conv2d(3, 64, ...)`) | MODIFY | Either change to `Conv2d(4, 64, ...)` for stacked RGBD, or keep two backbones (one RGB, one D) and fuse in decoder |
| Image loader | `MVGFormer/lib/dataset/JointsDataset.py:85-96` | MODIFY | Currently `cv2.imread(..., IMREAD_COLOR)`. Need to load depth alongside (depth file format / units / handling of invalid pixels is a design choice) |
| Image normalization | `JointsDataset.py:105-106` | MODIFY | Add depth normalization (e.g., divide by camera's max-range constant; mask invalid pixels to zero or to mean depth) |
| Pretrained ResNet | `pose_resnet.py:220-230` (state-dict load) | MODIFY | The pretrained backbone is 3-channel. For 4-channel: either (a) replicate channel-1 (red) weights to channel-4 and fine-tune, (b) initialize channel-4 randomly with small variance, or (c) keep backbone 3-channel and inject depth as a cross-attention key in the decoder |

### 2. Task — multi-person body joints to single-face landmarks

| What | Where | Action | Notes |
|---|---|---|---|
| `num_keypoints` | config + `multi_view_pose_transformer.py:147` | MODIFY | 15 (Panoptic) → face landmark count (68 / 51 / 478 — TBD, §5) |
| **Hardcoded `15` in decoder loop** | `MVGFormer/lib/models/dq_decoder.py:1141` (`query_num = round(query_num_mul_joints / 15)`) | MODIFY | Replace with `cfg.NETWORK.NUM_JOINTS`. Easy to miss — flagged because it's in code, not config |
| `num_instance` | config | MODIFY | 1024 (max persons) → 1 (single cropped face) |
| Query embedding scheme | `multi_view_pose_transformer.py:149, 159-184` | MODIFY | `'person_joint'` becomes effectively `'per_joint'` since `num_instance = 1`. Simpler and smaller |
| Classification head (object/no-object) | `multi_view_pose_transformer.py:193-195, 583-627` | DROP | No detection problem — every query is a known landmark |
| Hungarian matcher | `multi_view_pose_transformer.py:217-222`, `mvp/lib/models/matcher.py` | DROP | Single subject + known landmark indices = direct supervision per query |
| Cardinality loss | `multi_view_pose_transformer.py:630-650` | DROP | Always 1 face per crop |
| Query filtering before triangulation | `dq_decoder.py:889-908` (`outputs_class_prob`, `generate_valid_masks`) | MODIFY → simplify | Without classification, can drop the filter and triangulate all queries; or replace with a "visibility" filter based on per-view crop validity |

### 3. Per-view face crop integration

| What | Where | Action | Notes |
|---|---|---|---|
| Pre-loader face detection + crop | (NEW) wrap `JointsDataset.__getitem__` or add a transform | NEW | Detect face in each full view, select user-chosen face (across views), crop RGBD tile. Track the crop affine in `affine_trans` |
| Affine tracking | `JointsDataset.py:143-161` (`affine_trans`, `inv_affine_trans`) | REUSE / extend | Already designed for crop+resize affines; extend the affine to compose face-crop + resize |
| `IMAGE_SIZE` config | e.g. `MVGFormer/configs/panoptic/knn5-lr4-q1024-g8.yaml:43-45` | MODIFY | `[960, 512]` → tight face crop, e.g. `[256, 256]` or `[512, 512]` |
| Calibration update after crop | `MVGFormer/lib/mvn/utils/multiview.py:23-31` (`Camera.update_after_crop()`) | REUSE | Already handles principal-point shift; just feed it the face-crop affine |

### 4. Multi-person to single-person — 3D space reframing

| What | Where | Action | Notes |
|---|---|---|---|
| `MULTI_PERSON.SPACE_SIZE` | config (~`:81-94`) | REPLACE | `[8000, 8000, 2000]` mm (scene) → face region (~`[300, 300, 300]` mm centered on face) |
| `SPACE_CENTER` | config | REPLACE | World scene center → estimated face center (from detection / depth) per frame, OR a fixed origin in a face-aligned coordinate frame |
| `MAX_PEOPLE_NUM` | config (~`:94`) | DROP / set to 1 | |
| Reference-point MLP | `multi_view_pose_transformer.py:389` | REUSE | Output is normalized to `[0,1]^3` of the space — works regardless of space dimensions |

### 5. Geometry modules

| What | Where | Action | Notes |
|---|---|---|---|
| `ProjAttn` | `MVGFormer/lib/models/ops/modules/projattn.py` | REUSE | Geometry-agnostic; works on any 3D query |
| DLT triangulation | `MVGFormer/lib/mvn/utils/multiview.py` | REUSE | Same SVD-based approach |
| Structural triangulation `st` / `st-gt` | `MVGFormer/lib/structural/*` | REUSE — but redefine "bones" | Face landmarks have a meaningful structure (eyes pair, mouth contour, jaw chain); define `LIMBS_FACE` analogous to `LIMBS15` in `mvp/lib/core/loss.py:152-154` |
| Undistortion in decoder | `dq_decoder.py:119-204` | REUSE | Standard radial+tangential distortion model |

### 6. Camera calibration plumbing

| What | Where | Action | Notes |
|---|---|---|---|
| Calibration loader | `JointsDataset.py:186-220` | REUSE if rig output matches `(fx, fy, cx, cy, R, T)` schema | Write a small adapter from our rig's calibration format to this schema |
| Distortion coefficients | `JointsDataset.py:217` and `Camera` class | MODIFY | MVGFormer's distortion path exists; if our RGB-D rig is rectified, can pass zeros |
| Per-frame extrinsics | already supported | REUSE | Useful if cameras move (e.g., on a robot) |

### 7. Depth-channel exploitation — three options

The four-channel-input option (§4.1) is the most natural, but depth could also be injected later in the pipeline:

| Option | Where it attaches | Cost | Benefit |
|---|---|---|---|
| (a) **4th input channel to backbone** | `pose_resnet.py:116` | Loses some pretraining; need careful init | End-to-end learnable, no extra modules |
| (b) **Depth as geometric-consistency loss** | new term added in `dq_decoder.py` after triangulation, or in `core/loss.py` | Need to unproject depth maps and handle occlusions | Decouples feature extraction from depth; depth becomes a regularizer |
| (c) **Direct depth unprojection per landmark** | Post-decoder, in a new module that takes refined 2D landmarks + depth map and produces a 3D estimate | Extra computational stage; needs alignment of depth and color | Uses depth as a direct 3D signal, anchors scale absolutely |

**Recommended starting point:** option (a) with channel-replicated init, plus option (b) as an auxiliary loss with small weight (e.g., $\lambda_{\text{depth}} = 0.1$). Option (c) is a strong fallback if (a)+(b) underperform — and is also a great sanity-check baseline.

### 8. Loss / supervision

| What | Where | Action | Notes |
|---|---|---|---|
| `PerJointL1Loss` (3D) | `mvp/lib/core/loss.py:81-100` (and MVGFormer's equivalent) | REUSE | Identical for landmarks |
| `PerProjectionL1Loss2D` | `MVGFormer/lib/core/loss.py` (~`:245`) | REUSE | Critical for the iterative-refinement scheme |
| `PerBoneL1Loss` | `lib/core/loss.py` | MODIFY | Redefine `LIMBS` for face landmark connectivity (e.g., mouth contour, eye contour) |
| Classification loss `loss_ce` | `multi_view_pose_transformer.py` | DROP | See §4.2 |
| Cardinality loss | same | DROP | See §4.2 |
| Per-view visibility flag | `JointsDataset.py:110, 164-178` (`joints_2d_vis`, `joints_3d_vis`) | MODIFY → enhance | Set visibility = 0 per (view, landmark) when the landmark is self-occluded in that view (profile views occlude the off-side ear, jaw) |
| `weight_dict` | config | MODIFY | Remove `loss_ce`, `loss_cardinality`. Keep + tune `loss_pose_perjoint`, `loss_pose_perprojection_2d`, `loss_pose_perbone` |
| **NEW** 6-DoF loss | see §4.9 | NEW | Geodesic + L2 |

### 9. Head-model fitting — NEW additive stage

This stage does not exist in MVGFormer. Two options for `lib/models/head_pose_from_landmarks.py` (new file in our repo):

**Option A — closed-form rigid alignment.** Given predicted 3D landmarks $\hat{L} \in \mathbb{R}^{K \times 3}$ and template landmarks $L_0 \in \mathbb{R}^{K \times 3}$ from FLAME / BFM / a custom mean head, solve the orthogonal Procrustes problem for $(R, t, s)$:

$$\min_{R \in SO(3), t \in \mathbb{R}^3, s > 0} \sum_{k=1}^{K} \|\hat{L}_k - s R L_{0,k} - t\|_2^2$$

Closed-form via SVD of $\sum_k (\hat{L}_k - \bar{\hat{L}})(L_{0,k} - \bar{L_0})^\top$. Differentiable in PyTorch.

**Option B — learned MLP head.** $f_\theta: \mathbb{R}^{K \times 3} \to \mathrm{SE}(3)$, output rotation as 6D representation (Zhou et al. 2019) plus 3D translation, trained with the loss below.

**6-DoF loss:**

$$\mathcal{L}_{6\text{DoF}} = \lambda_R \cdot \text{geo}(R_{\text{pred}}, R_{\text{gt}}) + \lambda_t \cdot \|t_{\text{pred}} - t_{\text{gt}}\|_2$$

where the geodesic on $SO(3)$ is

$$\text{geo}(R_1, R_2) = \arccos\!\left(\frac{\mathrm{tr}(R_1^\top R_2) - 1}{2}\right)$$

Start with Option A (no extra params, immediate sanity check). Add Option B if Procrustes residuals are large relative to landmark noise.

**Integration point:** post-decoder, before `SetCriterion`. Append `outputs['pred_pose_6dof']` and add the loss term to `weight_dict`.

### 10. Biggest risks

1. **Loss of ResNet pretraining when adding the 4th channel.** Mitigation: replicate channel weights, freeze backbone for the first few epochs while the rest of the network learns, then unfreeze. Have a 3-channel-only baseline running in parallel as a sanity check.
2. **Crop affine propagation bugs corrupt the entire geometry.** A 1-pixel error in tracking the face-crop affine becomes a degree-scale error in 3D after triangulation. Mitigation: write a unit test that takes a synthetic 3D point, runs it through (project $\to$ crop $\to$ resize $\to$ inverse) and asserts round-trip recovery. Run this test continuously during development.
3. **Scale ambiguity with tight face crops and short-baseline rigs.** Face landmarks within a $\sim$200 mm cube observed from $\lesssim$0.5 m baselines have weakly-conditioned triangulation. Mitigation: depth channel (§4.7c) anchors scale; head-model prior (§4.9, scale $s$) regularizes; or constrain $s$ to a narrow range around a known mean.
4. **The hardcoded `15` in `dq_decoder.py:1141`** is the kind of thing a quick port will miss — explicitly listed here so it doesn't bite.
5. **Triangulation degenerate cases.** If a face landmark is visible in only 1 view (self-occlusion), DLT fails. Need per-landmark per-view confidences feeding into the triangulation weights, and need to handle the "<2 views visible" case explicitly (skip triangulation, fall back to the previous layer's 3D estimate, or use depth).
