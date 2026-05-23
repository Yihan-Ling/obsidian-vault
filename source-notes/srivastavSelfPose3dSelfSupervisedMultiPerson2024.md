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
<!-- claude-summary-start -->
*Pending*
<!-- claude-summary-end -->
%% end notes %%

## Notes & Highlights%% begin obsidian-freeform-notes %%

### Freeform Notes%% end obsidian-freeform-notes %%

### From Zotero

%% Import Date: 2026-05-22T21:04:06.167-04:00 %%
