---
layout: post
title: Acceleration Data Structures with NORI
description: Understanding Rendering
date: 2025-11-12
tags:
  - Rendering
  - Acceleration
  - Space-partitioning
  - KD-Tree
  - Octree
comments: true
use_math: true
---
2025/11/17 written.

# Reference
pbrt v3, https://www.pbrt.org/

# 0. Summary
![[summary0.png]]
Spatial partitioning approach is some problem that a triangle may overlap multiple spatial regions and thus may be tested for intersection multiple times as the ray passes. But, one property of primitive subdivision is each primitive appears in the hierarchy only once.

We do not consider `parallel computing` of Acceleration Data Structures. Because It's the next step. Thus, All rendering time data is computed by single CPU.

# 1. Octree
First, we talk about the theory of Octree. An Octree is the data structure that **implements 3D spatial partitioning to significantly accelerate** operations such as ray intersection.

![[octree.png]]

An octree has the parent node that is subdivided into eight children. Each of the children has leaves(e.g. triangles). How octree subdivide a 3d space recursively?, suppose **we have a bounding box(e.g. AABB) of object** and this bounding box is **uniformly! subdivided into eight smaller cubes.**

```cpp
struct Node {
	BoundingBox3f bbox;

	std::vector<uint32_t> triangles;
	std::unique_ptr<Node> children[8];

	bool isLeaf() const {
        for (int i = 0; i < 8; ++i) {
            if (children[i] != nullptr)
                return false;
        }
        return true;
	}
};
```

We can check whether triangle belongs to one of the eight sub-boxes. If the triangle belongs to second sub-box, we add the leaf(triangle) to second sub-box. Recursively, we get into second sub-box. *Once again*, this bounding box is uniformly subdivided into eight smaller cubes. The leaves of second sub-box *are re-distributed to new eight smaller cubes.* 

```cpp
void Octree::buildNode(Node* node,
    const std::vector<uint32_t>& tris,
    int depth) {

    // 1. condition of leaf
    if ((int)tris.size() <= m_maxLeafTris || depth >= m_maxDepth) {
        node->tris = tris;
        return;
    }

    // 2. subdivide the cubes
    subdivideNode(node, tris, depth);
}


void Octree::subdivideNode(Node* node,
    const std::vector<uint32_t>& triangles,
    int depth) {
    
    ... // this part make the uniformly eight cubes using node that has an bbox.
    
    BoundingBox3f childBBox[8];
    std::vector<uint32_t> childTriangles[8];

    for (uint32_t f : triangles) {
        const BoundingBox3f& tb = m_triangleBBoxes[f];
        for (int i = 0; i < 8; ++i) {
            if (cell[i].overlaps(tb)) {
                childTriangles[i].push_back(f);
                if (childBBox[i].isValid())
                    childBBox[i].expandBy(tb); // (min, max) of bbox is exapanded from tb to new tb.
                else
                    childBBox[i] = tb;
            }
        }
    }

    node->triangles.clear();

    for (int i = 0; i < 8; ++i) {
        if (!childTris[i].empty()) {
            node->children[i] = std::make_unique<Node>();
            node->children[i]->bbox = childBBox[i];
            buildNode(node->children[i].get(), childTriangles[i], depth + 1);
        }
    }
```

This process repeats until a certain condition is met, such as, the node reaches the maximum depth or the number of triangles in the cube is below threshold.

![[Octree2.png]]
We can use this approach for checking ray-intersection between triangle and ray. Suppose you shoot a ray. A naive approach is a ray checks all triangles, however, octree based approach is a ray does not check all triangle. This approach **accelerates ray traversal by allowing the ray to only visit the bounding boxes that it intersects.**

As a result, a naive approach has $O(N)$. octree based approach has $O(logN)$, but the worst case scenario has $O(log N + N) = O(N)$.

![[Octree3.png]]

| Maximum Depth<br>(Maximum leaves=8) | Second |
| :---------------------------------: | :----: |
|                  4                  |  4.7s  |
|                  6                  |  1.3s  |
|                  8                  | 0.772s |

We can know that the larger the octree has maximum depth, the more space that is not intersected with ray can be skipped, which accelerates culling performance. However, the more memory will be taken.

# 2. Background
Before we start studying `KD Tree` and `BVH(Bounding Volume Hierarchy)`, we know how KD tree and BVH make the tree. Typically, there are two approaches. First, we find the longest axis of the bounding box. Second, we find the lowest cost of ray traversal and ray intersection.

All methods basically want to be subdivided more efficiently from the parent bounding box to left and right children

The first method is naive approach. We just find the longest axis to split bounding box.  This approach of splitting is based on the underlying assumption that it leads to the most efficient spatial subdivision. Find the longest axis, sort the primitives (use `std::nth-element`) to find the middle one, and use its position as the pivot to split the parent bounding box into left and right.

The second method is SAH(Surface Area Heuristic). It's the same approach to split the bounding box into left and right. SAH find the lowest cost. So, we know how certain bounding box has lower cost that others.

$$\begin{align}
c(A, B)=t_{trav} + p_A\sum^{N_A}_{i=1}t_{isect}(a_i)+ p_A\sum^{N_B}_{i=1}t_{isect}(b_i)
\end{align}$$

$a_i$ and $b_i$ are the indices of primitives in the two children nodes. 
# 2.1. KD Tree

