---
layout: post
title: "Effects of Rotation Matrix on AABB"
description: "About AABB And Rotation Matrix"
date: 2024-08-01
tags: AABB, Matrix, Graphics
comments: true
---


물체의 이동, 크기, 회전을 적용함에 따라 AABB가 이에 맞게 변환하는 것을 구현할 생각이다. AABB는 `center`와 `extents`를 통해서 구성되는데, center에서 경계면까지의 거리를 extents라고 한다. 

AABB는 Axis-Aligned Bounding Box라는 이름에 맞게 기저 축에 정렬된 충돌 박스이다. 따라서 물체가 회전하게 된다면, AABB는 회전을 하는 것이 아닌 회전에 맞게 크기가 변화된다. 이 부분에서 예상치 못한 문제점이 발생한다.
```cpp
inline void XM_CALLCONV BoundingBox::Transform(BoundingBox& Out, FXMMATRIX M) const noexcept
{
    // Load center and extents.
    XMVECTOR vCenter = XMLoadFloat3(&Center);
    XMVECTOR vExtents = XMLoadFloat3(&Extents);

    // Compute and transform the corners and find new min/max bounds.
    XMVECTOR Corner = XMVectorMultiplyAdd(vExtents, g_BoxOffset[0], vCenter);
    Corner = XMVector3Transform(Corner, M);

    XMVECTOR Min, Max;
    Min = Max = Corner;

    for (size_t i = 1; i < CORNER_COUNT; ++i)
    {
        Corner = XMVectorMultiplyAdd(vExtents, g_BoxOffset[i], vCenter);
        Corner = XMVector3Transform(Corner, M);

        Min = XMVectorMin(Min, Corner);
        Max = XMVectorMax(Max, Corner);
    }

    // Store center and extents.
    XMStoreFloat3(&Out.Center, XMVectorScale(XMVectorAdd(Min, Max), 0.5f));
    XMStoreFloat3(&Out.Extents, XMVectorScale(XMVectorSubtract(Max, Min), 0.5f));
}
```

$$\begin{bmatrix}x_{x}&x_{y}&x_{z}&1\end{bmatrix}\times
\begin{bmatrix}S_{x}&0&0&0\\0&S_{y}&0&0\\0&0&S_{z}&0\\T_{x}&T_{y}&T_{z}&0\\\end{bmatrix} \tag{1}$$
`Scale`과 `Translation`만 적용된 행렬과의 곱셈이다. 할 때는 문제가 발생하지 않는다. Scale이 커짐에 따라 AABB도 적절하게 크기가 커진다는 것을 간단하게 알 수 있다.

다만, 여기에 회전행렬이 추가될 경우에 문제가 발생한다. 회전행렬에 대해서 아래와 같이 가정해보자.
$$R =
\begin{bmatrix}R_{x}&R_{x}&R_{x}&0\\R_{y}&R_{y}&R_{y}&0\\R_{z}&R_{z}&R_{z}&0\\0&0&0&0\\\end{bmatrix} \tag{2}$$
$$\begin{bmatrix}x_{x}&x_{y}&x_{z}&1\end{bmatrix}\times
\begin{bmatrix}S_{x}R_{x}&S_{x}R_{x}&S_{x}R_{x}&0\\S_{y}R_{y}&S_{y}R_{y}&S_{y}R_{y}&0\\S_{z}R_{z}&S_{z}R_{z}&S_{z}R_{z}&0\\T_{x}&T_{y}&T_{z}&0\\\end{bmatrix} \tag{3}$$
행렬이 Row-major라는 가정하에 $S\times R\times T$를 하게 된다면 $(3)$과 같은 행렬이 만들어진다. 이렇게 된다면 계산의 결과에도 알 수 있듯이, $x_{x}$는 회전행렬에 대해서 $S_{y}R_{y}$과 $S_{z}R_{z}$의 영향도 받게 된다.

물론, 계산상에서 발생하는 문제점은 없고 `해당 영향`으로 인해 특정 도형이 문제가 발생한다는 점이다.

문제가 발생하는 도형은 바로 `구체`이다. 구체의 경우에는 회전값을 주더라도 겉으로 들어나는 변화가 없다. 하지만, AABB는 다르다. 회전 값이 추가됨에 따라 $(3)$의 계산식으로 인해 형태가 변하게 된다.
![AABB_Sphere](https://github.com/user-attachments/assets/8670a45b-02ea-48ef-b4fa-cad6fd7422bd)

이 부분을 해결하기 위해서 $S_{x}, S_{y}, S_{z}$과 같은 구체에 대해서는 $S\times T$만이 적용된 변환행렬을 곱하는 방법이 있다. 그러나, 완벽한 해결책은 아니다. [Video](https://www.youtube.com/watch?v=Zyz0hNHtj78&ab_channel=KoalaJung)

AABB를 사용하지 않고 OBB나 다른 경계 선 기법을 사용하는 방법도 있다.
