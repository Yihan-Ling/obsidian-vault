# [Three-D Wide Faces (3DWF): Facial Landmark Detection and 3D Reconstruction over a New RGB–D Multi-Camera Dataset](zotero://select/library/items/SEINJTQX)

 **Authors:** Marcos Quintana, Sezer Karaoglu, Federico Alvarez, Jose Manuel Menendez, Theo Gevers
 **Published:** 2019
 **Journal/Conference:** Sensors
**Citekey:** `quintanaThreeDWideFaces2019`

---
## Abstract
> [!abstract]
>  ```
>  Latest advances of deep learning paradigm and 3D imaging systems have raised the necessity for more complete datasets that allow exploitation of facial features such as pose, gender or age. In our work, we propose a new facial dataset collected with an innovative RGB–D multi-camera setup whose optimization is presented and validated. 3DWF includes 3D raw and registered data collection for 92 persons from low-cost RGB–D sensing devices to commercial scanners with great accuracy. 3DWF provides a complete dataset with relevant and accurate visual information for different tasks related to facial properties such as face tracking or 3D face reconstruction by means of annotated density normalized 2K clouds and RGB–D streams. In addition, we validate the reliability of our proposal by an original data augmentation method from a massive set of face meshes for facial landmark detection in 2D domain, and by head pose classification through common Machine Learning techniques directed towards proving alignment of collected data.
>  ```

---

%% begin notes %%
## Summary
## Introduction

Facial analysis tasks — landmark detection, 3D face reconstruction, head pose estimation — are bottlenecked by the absence of datasets that simultaneously capture multi-viewpoint RGB-D streams, high-resolution 3D point clouds, and demographic annotations at scale. Existing collections (Multi-PIE, Florence, UHDB31) either use costly capture rigs, cover too few subjects, or lack the extreme-pose coverage needed to stress-test deep learning pipelines. 3DWF fills this gap with an affordable three-camera RGB-D system built from first-generation Asus Xtion sensors, capturing 92 subjects across ten discrete head-pose markers and producing both continuous RGB-D streams and density-normalized 2K point clouds.

## Related Work

| Prior work | Relevance to 3DWF |
|---|---|
| **Multi-PIE** (Gross et al., 2010) | Benchmark RGB face database with 337 subjects; defines pose/illumination variation but lacks depth |
| **300-W** (Sagonas et al., 2016) | Establishes landmark annotation standards and evaluation metrics used throughout |
| **Florence Dataset** (Bagdanov et al., 2011) | RGB-D hybrid dataset but only 53 subjects, costly 3dMD rig |
| **UHDB31** (Le & Kakadiaris, 2017) | Extends Florence with more poses/subjects but still high-cost setup |
| **Pandora** (Borghi et al., 2016) | Head/shoulder dataset for driving; only 20 subjects, single camera |
| **Feng et al. (2018)** | Dense 3D reconstruction from 2D images; 2000 images of 135 subjects, used as comparison for reconstruction quality |
| **VanillaCNN / Tweaked CNN** (Wu et al., 2018) | Pose-specific landmark regression using mid-network features; one of two tested architectures |
| **Recombinator Networks (RCN)** (Honari et al., 2016) | Coarse-to-fine feature aggregation with bidirectional branches; the other tested architecture |
| **Zhu & Ramanan (2012)** | Unified mixture-of-trees model for face detection, pose and landmark estimation; representative graphical model baseline |
| **Sun et al. (2013)** | Three-level cascaded deep CNN for landmark detection in a coarse-to-fine manner |
| **Fanelli et al. (2011)** | First notable depth-sensor-based head pose estimation via random regression forests |
| **Liu et al. (2016)** | CNN head pose estimation trained on synthetic rendered data; motivates the synthetic augmentation strategy |
| **PointNet** (Qi et al., 2017) / **PointCNN** (Li et al., 2018) | Deep learning on normalized point clouds; motivation for providing density-normalized 2K clouds |
| **3DMM** (Blanz & Vetter, 1999) | Statistical face morphable model; noted as a single-RGB baseline that can get trapped in local minima |

## Methods & Architecture

**Dataset capture pipeline**

- Three Asus Xtion RGB-D cameras (first-generation) synchronized via OpenNI 2 over independent USB buses.
- Optimal configuration (validated empirically): subject-to-frontal-camera distance 80 cm, luminous flux 250 lm, inter-camera angle 30°, light sources aimed at opposite side cameras.
- 92 subjects; each session yields 600–1200 RGB-D frames per camera across ten head-pose markers; supplementary HD ground-truth scan from a Faro Freestyle 3D Laser Scanner (point accuracy 0.5 mm).

**3D reconstruction pipeline** (Section 5)

1. **Registration** — affine transformation with 10 manually selected point correspondences per camera pair; RANSAC refines $R_{\text{left,right}}$ and $T_{\text{left,right}}$; left and right clouds are transformed into the frontal coordinate frame to form $Cl_{\text{Total}}$.
2. **Refinement** — Iterative Closest Point (ICP, point-to-point), 30 iterations, discarding the worst 90% candidate pairs:
$$R, t = \sum_{i=1}^{N} \arg\min_{R,T} \|(Rp_i + t) - q_i\|_2$$
3. **ROI extraction** — Dlib face detector on the frontal RGB image; VanillaCNN supplies 5 2D facial landmarks (left eye, right eye, nose, left mouth corner, right mouth corner); landmarks projected to 3D via Perspective Projection Model to crop $Cl_F'$.
4. **Noise filtering** — dual-criterion: (a) CIELAB color threshold using $\Delta E^*_{00}$ (CIEDE2000) around the mean ROI color within $\pm 0.75\sigma$; (b) depth confidence interval $[\hat{Z} \pm 2.25\sigma_Z]$.
5. **Uniform distribution** — voxel-grid downsampling with dynamic radius to produce density-normalized **2K-point** clouds, split into four quadrants of 512 points each.

**Data augmentation for landmark detection** (Section 4)

- Source: 3DUniversum (3DU) dataset — 300 3D face meshes captured by a rotating structured-light sensor.
- Raycasting projection: for each mesh, synthesize 2D images by sweeping pitch $\in [-30°, 30°]$, yaw $\in [-30°, 30°]$, roll $\in [-30°, 30°]$ in 5° steps and distance $\in [110, 160]$ cm in 10 cm steps (Euler angle parameterization).
- Each mesh annotated once; projection generates thousands of pose-augmented images automatically.
- Split: 245 subjects (81.67%) train / 40 (13.34%) val / 15 (5%) test.

**Landmark detection architectures**

| Architecture | Key design | Loss |
|---|---|---|
| **VanillaCNN** | 4 conv + 2 dense layers; mid-network clustering into pose-specific regressors; hyperbolic tangent activation; Adam optimizer | $L_2$ normalized by inter-ocular distance |
| **RCN** (Recombinator Networks) | Bidirectional multi-branch; 3–4 conv layers per branch; branches concatenated coarse-to-fine; per-landmark softmax | Negative log-likelihood + $\lambda\|W\|^2$ regularization |

**Head pose classification / validation** (Section 6)

- 3D facial landmarks from VanillaCNN (fine-tuned on AFLW) projected back to 3D via inverse Perspective Projection; Least-Squares Rigid Motion by SVD aligns markers 2–10 to marker 1 (frontal reference):
$$R = \arg\min_{R \in SO(3)} \sum_{i=1}^n w_i \|Rp_i - q_i\|^2 \quad \Rightarrow \quad R = VU^T$$
- Euler angles $(\phi, \theta, \psi)$ extracted from $R$ then classified by LDA and Gaussian Naive Bayes (GNB).

## Experiments & Results

**Facial landmark detection** (evaluated on AFLW and 3DU; metric: normalized inter-ocular distance error, threshold 0.1)

- Training details: learning rate $10^{-4}$ for both architectures; AFLW split: 9000 train / 3000 val / 1000 test; 3DU split: 35 train / 12 val subjects per viewpoint.
- **Best configuration:** RCN trained on 3DU augmented data achieves the lowest error rate on the 3DU test set (exact figure presented as histogram in Figure 8 — numeric value not printed in the extracted text).
- **Transfer result:** VanillaCNN pre-trained on 3DU then fine-tuned on AFLW achieves the best AFLW results among all combinations; mid-network features initialized on 3DU provide better global-minimum convergence for AFLW than training from scratch.
- **Worst configuration:** VanillaCNN trained and tested purely on 3DU yields the highest error; the pose-specific clustering in VanillaCNN is better suited to natural-image data like AFLW.

**Head pose classification** (LDA and GNB classifying 9 non-frontal markers; 80%/20% train/test split)

| Method | Training Accuracy | Testing Accuracy |
|---|---|---|
| LDA | 82% | 83% |
| GNB | 84% | 84% |

- Markers corresponding to lateral gaze (e.g., markers 2 and 5) are the hardest to classify; pitch-variation markers (e.g., marker 10) the easiest.
- Euler angle distributions across the ten markers are approximately uniform (Figure 9), validating the head-pose coverage of the capture protocol.

**3D reconstruction accuracy**

- Average minimum distance from $Cl_F'$ (marker 1, frontal) to Faro HD ground-truth $Cl_{HD}$: **16–23 mm**.
- Depth sensor error at sub-1 m range is 5–15 mm (per reference [44]); residual difference attributed to non-rigid facial deformation between capture sessions.

## Conclusion

The paper demonstrates that a three-camera first-generation RGB-D rig can produce a dataset (3DWF, 92 subjects, 10 poses, 2K normalized clouds) that is more complete than prior affordable facial 3D datasets. The proposed raycasting-based synthetic data augmentation from 3D meshes enables effective pre-training for landmark detectors: RCN benefits most on 3D-derived data, while VanillaCNN benefits most as a fine-tuning target on AFLW after 3DU pre-training. Head pose classification at 83–84% accuracy with simple LDA/GNB validates that the Euler angle distribution of the captured markers is consistent and discriminable.

## Impact

3DWF provides a practical template for building low-cost multi-view RGB-D facial datasets at a scale (92 subjects, multi-pose) that was previously achievable only with expensive commercial rigs. The 2K-normalized cloud format directly aligns with emerging point-cloud deep learning frameworks (PointNet, PointCNN), opening a concrete pathway for 3D facial attribute learning without mesh reconstruction overhead. However, the dataset remains relatively small by modern deep-learning standards, the 5-landmark annotation is sparse compared to the 68-point convention established by 300-W, and reconstruction accuracy in the 16–23 mm range limits utility for high-fidelity identity or expression capture. Its clearest near-term impact is as a pre-training or domain-adaptation resource for pose-robust landmark detectors operating under wide-angle or surveillance-like viewing conditions.
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-05-28T12:56:44.317-04:00 %%
