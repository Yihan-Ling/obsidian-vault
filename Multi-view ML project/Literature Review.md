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