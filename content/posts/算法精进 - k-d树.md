---
title: "算法精进之 k-d树"
date: 2019-05-19T21:19:46+08:00
draft: false
---

# 概述
k-d树（k-dimensional树的简称），是一种分割k维数据空间的数据结构。主要应用于多维空间关键数据的搜索（如：范围搜索和最近邻搜索）。K-D树是二进制空间分割树的特殊的情况。

# 算法原理
k-d tree是每个节点均为k维数值点的二叉树，其上的每个节点代表一个超平面，该超平面垂直于当前划分维度的坐标轴，并在该维度上将空间划分为两部分，一部分在其左子树，另一部分在其右子树。即若当前节点的划分维度为d，其左子树上所有点在d维的坐标值均小于当前值，右子树上所有点在d维的坐标值均大于等于当前值，本定义对其任意子节点均成立。

## 树的构建
一个平衡的k-d tree，其所有叶子节点到根节点的距离近似相等。但一个平衡的k-d tree对最近邻搜索、空间搜索等应用场景并非是最优的。

常规的k-d tree的构建过程为：循环依序取数据点的各维度来作为切分维度，取数据点在该维度的中值作为切分超平面，将中值左侧的数据点挂在其左子树，将中值右侧的数据点挂在其右子树。递归处理其子树，直至所有数据点挂载完毕。

### 切分维度选择优化
构建开始前，对比数据点在各维度的分布情况，数据点在某一维度坐标值的方差越大分布越分散，方差越小分布越集中。从方差大的维度开始切分可以取得很好的切分效果及平衡性。

### 中值选择优化
第一种，算法开始前，对原始数据点在所有维度进行一次排序，存储下来，然后在后续的中值选择中，无须每次都对其子集进行排序，提升了性能。

第二种，从原始数据点中随机选择固定数目的点，然后对其进行排序，每次从这些样本点中取中值，来作为分割超平面。该方式在实践中被证明可以取得很好性能及很好的平衡性。

### 例子
本文采用常规的构建方式，以二维平面点(x,y)的集合(2,3)，(5,4)，(9,6)，(4,7)，(8,1)，(7,2)为例结合下图来说明k-d tree的构建过程。

1. 构建根节点时，此时的切分维度为x，如上点集合在x维从小到大排序为(2,3)，(4,7)，(5,4)，(7,2)，(8,1)，(9,6)；其中值为(7,2)。（注：2,4,5,7,8,9在数学中的中值为(5 + 7)/2=6，但因该算法的中值需在点集合之内，所以本文中值计算用的是len(points)//2=3, points\[3\]=(7,2)）
2. (2,3)，(4,7)，(5,4)挂在(7,2)节点的左子树，(8,1)，(9,6)挂在(7,2)节点的右子树。
3. 构建(7,2)节点的左子树时，点集合(2,3)，(4,7)，(5,4)此时的切分维度为y，中值为(5,4)作为分割平面，(2,3)挂在其左子树，(4,7)挂在其右子树。
4. 构建(7,2)节点的右子树时，点集合(8,1)，(9,6)此时的切分维度也为y，中值为(9,6)作为分割平面，(8,1)挂在其左子树。至此k-d tree构建完成。
![k-d树](http://ww2.sinaimg.cn/large/006tNc79ly1g36y5azze7j30d008zdgl.jpg)

上述的构建过程结合下图可以看出，构建一个k-d tree即是将一个二维平面逐步划分的过程。
![](http://ww4.sinaimg.cn/large/006tNc79ly1g36y5vgq1hj30ci0cn74z.jpg)

### 实现
```python
def kd_tree(points, depth):
    if 0 == len(points):
        return None
    cutting_dim = depth % len(points[0])
    medium_index = len(points) // 2
    points.sort(key=itemgetter(cutting_dim))
    node = Node(points[medium_index])
    node.left = kd_tree(points[:medium_index], depth + 1)
    node.right = kd_tree(points[medium_index + 1:], depth + 1)
    return node
```

## 寻找d维最小坐标值点
1. 若当前节点的切分维度是d
因其右子树节点均大于等于当前节点在d维的坐标值，所以可以忽略其右子树，仅在其左子树进行搜索。若无左子树，当前节点即是最小坐标值节点。

2. 若当前节点的切分维度不是d
需在其左子树与右子树分别进行递归搜索。
如下为寻找d维最小坐标值点代码：
```python
def findmin(n, depth, cutting_dim, min):
    if min is None:
        min = n.location
    if n is None:
        return min
    current_cutting_dim = depth % len(min)
    if n.location[cutting_dim] < min[cutting_dim]:
        min = n.location
    if cutting_dim == current_cutting_dim:
            return findmin(n.left, depth + 1, cutting_dim, min)
    else:
        leftmin = findmin(n.left, depth + 1, cutting_dim, min)
        rightmin = findmin(n.right, depth + 1, cutting_dim, min)
        if leftmin[cutting_dim] > rightmin[cutting_dim]:
            return rightmin
        else:
            return leftmin
```

## 新增节点
从根节点出发，若待插入节点在当前节点切分维度的坐标值小于当前节点在该维度的坐标值时，在其左子树插入；若大于等于当前节点在该维度的坐标值时，在其右子树插入。递归遍历，直至叶子节点。
如下为新增节点代码：

```python
def insert(n, point, depth):
    if n is None:
        return Node(point)
    cutting_dim = depth % len(point)
    if point[cutting_dim] < n.location[cutting_dim]:
        if n.left is None:
            n.left = Node(point)
        else:
            insert(n.left, point, depth + 1)
    else:
        if n.right is None:
            n.right = Node(point)
        else:
            insert(n.right, point, depth + 1)
```
多次新增节点可能引起树的不平衡。不平衡性超过某一阈值时，需进行再平衡。

## 删除节点
给定点p，查询数据集中与其距离最近点的过程即为最近邻搜索。

如在上文构建好的k-d tree上搜索(3,5)的最近邻时，本文结合如下左右两图对二维空间的最近邻搜索过程作分析。
![](http://ww2.sinaimg.cn/large/006tNc79ly1g36yahglo5j30d00blwg2.jpg)

1. 首先从根节点(7,2)出发，将当前最近邻设为(7,2)，对该k-d tree作深度优先遍历。以(3,5)为圆心，其到(7,2)的距离为半径画圆（多维空间为超球面），可以看出(8,1)右侧的区域与该圆不相交，所以(8,1)的右子树全部忽略。
2. 接着走到(7,2)左子树根节点(5,4)，与原最近邻对比距离后，更新当前最近邻为(5,4)。以(3,5)为圆心，其到(5,4)的距离为半径画圆，发现(7,2)右侧的区域与该圆不相交，忽略该侧所有节点，这样(7,2)的整个右子树被标记为已忽略。
3. 遍历完(5,4)的左右叶子节点，发现与当前最优距离相等，不更新最近邻。所以(3,5)的最近邻为(5,4)。

如下为最近邻搜索代码：

```python
def NN(Point Q, kdTree T, int cd, Rect BB):
  // if this bounding box is too far, do nothing
  if T == NULL or distance(Q, BB) > best_dist:
    return   
  
  // if this point is better than the best: 
  dist = distance(Q, T.data)if dist < best_dist:       
    best = T.data      
    best_dist = dist   
  // visit subtrees is most promising order:
  if Q[cd] < T.data[cd]:      
    NN(Q, T.left, next_cd, BB.trimLeft(cd, t.data))      
    NN(Q, T.right, next_cd, BB.trimRight(cd, t.data))
  else:     
    NN(Q, T.right, next_cd, BB.trimRight(cd, t.data))      
    NN(Q, T.left, next_cd, BB.trimLeft(cd, t.data))
```



# 复杂度分析

| 操作       | 平均复杂度 | 最坏复杂度 |
| ---------- | ---------- | ---------- |
| 新增节点   | O(logn)    | O(n)       |
| 删除节点   | O(logn)    | O(n)       |
| 最近邻搜索 | O(logn)    | O(n)       |