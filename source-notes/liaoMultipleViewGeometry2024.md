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

<!-- claude-summary-start -->
*Pending* 
<!-- claude-summary-end -->

## Notes & Highlights



> [!quote] Highlight (Page 710)
> Most closely related to our work, MvP [36] extends DETR for multi-view 3D human pose estimation. Similar to PETR [21, 22], it introduces a RayConv operation to integrate the camera parameters into the image features. Although it achieves good in-domain performance when cameras are the same for training and testing, it does not generalize well to different camera arrangements.





- **My Note:** Against MvP


