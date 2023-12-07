---
title: Chamfer Distace
created: 2023-02-03 18:17:04, 星期五
modified: 2023-11-22 16:52:46, 星期三
tags: [点云, metric, pytorch3d]
---
#点云 #metric #pytorch3d

用于点云距离度量，比如重建出来的 SDF 结果

### 理论计算

找到两种算法：

$$
d_{CD}(S_1,S_2)=
\frac{1}{S_1}\sum\limits_{x\in S_1}\min\limits_{y\in S_2}\left\|x-y\right\|^2_2+
\frac{1}{S_2}\sum\limits_{y\in S_2}\min\limits_{x\in S_1}\left\|y-x\right\|^2_2
$$

叫做倒角距离

还有一个说法是叫 EMD，Earth Move Distance，搬运区域块的距离

计算的是重建结果与 GT 之间的差别，好像只有数据集才有 GT，自己拍的图片没有 GT 可以比较

volsdf 评估 mesh 结果时用到，作者说使用 pytorch3d 中的库函数计算的，见：
[pytorch3d.loss — PyTorch3D documentation](https://pytorch3d.readthedocs.io/en/latest/modules/loss.html#pytorch3d.loss.chamfer_distance)

这里还有其他 loss，也是评估 mesh 啥的，可以看下

### 语言描述

其中一个点集是参考点集，另一个点集是待评估点集。它的计算方法是对于每个参考点，找到距其最近的待评估点，计算它们之间的距离，并将这些距离求和，然后对于待评估点集中的每个点，同样找到距离其最近的参考点，计算它们之间的距离，并将这些距离求和，最终将两个和相加得到 Chamfer distance。

### pytorch3d 方法

```python
import argparse
import numpy as np
import torch
from pytorch3d.io import IO
from pytorch3d.ops import sample_points_from_meshes
from pytorch3d.loss import chamfer_distance
from icecream import ic

parser = argparse.ArgumentParser(description='Calculate chamfer distance between two point clouds')
parser.add_argument('obj1', type=str, help='path to the first OBJ file')
parser.add_argument('obj2', type=str, help='path to the second OBJ file')
args = parser.parse_args()


sample_points = 100000


device=torch.device("cuda:0")

# 创建PyTorch3D网格对象
meshes1 = IO().load_mesh(args.obj1, device=device)
meshes2 = IO().load_mesh(args.obj2, device=device)

# ic(type(meshes1)) <class 'pytorch3d.structures.meshes.Meshes'>


# 从第一个网格中采样100k个点
sample_mesh1 = sample_points_from_meshes(meshes1, sample_points)
# 从第二个网格中采样100k个点
sample_mesh2 = sample_points_from_meshes(meshes2, sample_points)

# 计算Chamfer距离
loss_chamfer, _ = chamfer_distance(sample_mesh1, sample_mesh2)

print(f"Chamfer Distance: {loss_chamfer.item()}")
```

【注】[[pytorch3d使用]] 安装还挺麻烦的

### chamferdist 库方法

似乎不准，大多论文用的 pytorch3d 计算尺度。
这个值很多时候算出来很大，10^2-10^3 数量级

```python
import torch
import trimesh
import argparse
from chamferdist import ChamferDistance



parser = argparse.ArgumentParser(description='Convert OBJ files to tensor format')
parser.add_argument('obj1', type=str, help='path to the first OBJ file')
parser.add_argument('obj2', type=str, help='path to the second OBJ file')
args = parser.parse_args()

# 使用trimesh库加载OBJ文件
mesh1 = trimesh.load(args.obj1)
vertices1 = mesh1.vertices

mesh2 = trimesh.load(args.obj2)
vertices2 = mesh2.vertices

# 创建source与target cloud
source_cloud = torch.from_numpy(vertices1).unsqueeze(0).cuda()
target_cloud = torch.from_numpy(vertices2).unsqueeze(0).cuda()




# Create two random pointclouds
# (Batchsize x Number of points x Number of dims)
source_cloud = torch.randn(1, 100, 3).cuda()
target_cloud = torch.randn(1, 50, 3).cuda()
source_cloud.requires_grad = True

# Initialize Chamfer distance module
chamferDist = ChamferDistance()
# Compute Chamfer distance
dist_forward = chamferDist(source_cloud, target_cloud)
print("Forward Chamfer distance:", dist_forward.detach().cpu().item())

# Chamfer distance depends on the direction in which it is computed (as the
# nearest neighbour varies, in each direction). One can either flip the order
# of the arguments, or simply use the "reverse" flag.
dist_backward = chamferDist(source_cloud, target_cloud, reverse=True)
print("Backward Chamfer distance:", dist_backward.detach().cpu().item())
# Or, if you rather prefer, flip the order of the arguments.
dist_backward = chamferDist(target_cloud, source_cloud)
print("Backward Chamfer distance:", dist_backward.detach().cpu().item())

# To get a symmetric measure, the simplest way is to average both the "forward"
# and "backward" distances. This is done by the "bidirectional" flag.
cdist = 0.5 * chamferDist(source_cloud, target_cloud, bidirectional=True)
cdist = 0.5 * chamferDist(target_cloud, source_cloud, bidirectional=True)
print("Bi-directional Chamfer distance:", cdist.detach().cpu().item())

# As a sanity check, chamfer distance between a pointcloud and itself must be
# zero.
dist_self = chamferDist(source_cloud, source_cloud)
print("Chamfer distance (self):", dist_self.detach().cpu().item())
dist_self = chamferDist(target_cloud, target_cloud)
print("Chamfer distance (self):", dist_self.detach().cpu().item())

# Backprop using this loss!
cdist.backward()
print(
    "Gradient norm wrt bidirectional Chamfer distance:",
    source_cloud.grad.norm().detach().cpu().item(),
)
```

### 朴素：cdist 遍历计算

加速可采用 [[numba加速]] 方案。

```python
def compute_cd(pts1, pts2):
    sample_size = 100000
    sample_pts1 = pts1[np.random.choice(len(pts1), sample_size, replace=False)]
    sample_pts2 = pts2[np.random.choice(len(pts2), sample_size, replace=False)]
    pts1 = sample_pts1
    pts2 = sample_pts2

    chamfer_distance1 = 0.0

    for p in pts1:
        pp = np.min(cdist([p], pts2), axis=1)
        chamfer_distance1 += pp[0]

    print("full:", chamfer_distance1, "; len:", len(pts1))
    c1 = chamfer_distance1 / len(pts1)

    chamfer_distance2 = 0.0
    for p in pts2:
        pp = np.min(cdist([p], pts1), axis=1)
        chamfer_distance2 += pp[0]

    print("full:", chamfer_distance2, "; len:", len(pts2))
    c2 = chamfer_distance2 / len(pts2)

    print("full:", chamfer_distance1 + chamfer_distance2, "; len:", len(pts1) + len(pts2))
    cd = (chamfer_distance1 + chamfer_distance2) / (len(pts1) + len(pts2))

    return cd

def mesh_cd(gt_path, pred_path):
    print("Loading mesh: ", gt_path)
    print("Loading mesh: ", pred_path)

    pts1 = trimesh.load(gt_path).vertices
    pts2 = trimesh.load(pred_path).vertices

    cd_score = compute_cd(pts1, pts2)
    print(cd_score)
    ic(cd_score)
```
