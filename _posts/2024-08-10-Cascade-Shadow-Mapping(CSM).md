---
layout: post
title: "Stable Cascade Shadow Mapping(CSM)"
description: "Cascade Shadow Mapping and Camera flickering"
date: 2024-08-10
tags: Graphics Lighting Shadow
comments: true
use_math: true
---

## 참고
[https://cutecatgame.tistory.com/6](https://cutecatgame.tistory.com/6)
[https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps)
[https://www.slideshare.net/slideshow/implements-cascaded-shadow-maps-with-using-texture-array/67006667#2](https://www.slideshare.net/slideshow/implements-cascaded-shadow-maps-with-using-texture-array/67006667#2)
[https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus)

## Why use Cascade Shadow Mapping?
Directional Light에 대해서 공부를 해본 경험이 있을 것이다. 가장 기초적인 방식인 Shadow Mapping 방식을 통해 그림자를 만들려면 빛의 위치를 정하고 해당 위치에서 빛의 방향으로 View행렬과 Projection행렬을 만들어 Shadow Mapping을 진행하였다.

하지만, 이러한 방법은 장면이 엄청 커지게 되면서 해당 장면을 모두 담기에는 그림자 해상도가 부족해지는 문제점이 발생한다. 해당 문제점으로 장면이 어떻게 되냐면, 빛이 모든 물체를 담기 위해 물체를 Shadow Map에 물체를 작게 그리게 된다.

![해상도가낮은직교행렬그림자맵](https://github.com/user-attachments/assets/0a723bd2-749b-4987-9755-b46de45925ad)

~~그렇게 되면 빛이 조금만 움직여도 Shadow Map의 물체의 움직임은 상대적으로 커지게 된다. 이로 인한 영향으로 그림자의 떨림이 심해지는 현상이 발생하게 된다.~~

그림자의 떨림 현상은 다른 이유로 발생한다. 해당 문제는 Projection행렬에 Offset을 더해주는 것으로 문제를 해결할 수 있다. 기본적으로 해당 현상이 발생하는 이유는 Texture Pixel과 Shadow Map 해상도에 의해 좌표 변환 과정에서 생기는 부동소수점 오차 때문이다. [GDC 2005에서 소개되었던 문제](https://ubm-twvideo01.s3.amazonaws.com/o1/vault/gdc09/slides/100_Handout%203.pdf)이다.

```cpp
    // 카메라 위치 변화에 따른 그림자 떨림 방지를 위한 offset
    Vector3 shadowOrigin = Vector3(0.0f, 0.0f, 0.0f);
    shadowOrigin = Vector3::Transform(shadowOrigin, lightViewProjRow); // 그림자 좌표 공간에서의 원점
    shadowOrigin *= (SHADOW_MAP_SIZE / 2.0f); // 그림자 원점을 texture pixel 단위로 변환

    Vector3 roundedOrigin = XMVectorRound(shadowOrigin);
    Vector3 roundedOffset = roundedOrigin - shadowOrigin; // texel에 맞추기 위한 offset 계산

    roundedOffset *= (2.0f / SHADOW_MAP_SIZE); // 그림자 맵 좌표계로 다시 변환
    roundedOffset.z = 0.0f; // 깊이값 영향 무시
    
    // offset을 x, y값에 더하여 그림자가 texel의 중앙에 위치하도록 조정
    lightProjRow.m[3][0] += roundedOffset.x;
    lightProjRow.m[3][1] += roundedOffset.y;
```

제한적인 해상도를 가지고 그림자 품질을 늘리는 기법이 바로 `Cascade Shadow Mapping`이다.

## What is Cascade Shadow Mapping
현재 시야에 보이는 장면에 대해서 그림자를 만들기 위해 단계를 나누어서 Shadow Map을 만드는 것이라고 할 수 있다. Shadow Map을 나누는 기준은 Camera Frustum이 되며, 나눠진 Camera Frustum 안에 들어오는 물체에 대해서만 Shadow Map을 만든다.

### How to split up Camera Frustum
![시야절두체](https://github.com/user-attachments/assets/8e5a5aa9-47d9-4e62-9be8-b71d6e083fa8)

이렇게 카메라 Frustum을 3부분으로 나누고 나눠진 부분에 대해서 **화살표(빛의 방향)** 을 기준으로 View행렬과 Projection행렬을 만들어 Shadow Map을 만드는 것이다.

**사각형 영역**이 **Projection행렬의 영역**으로 해당 영역에 들어온 물체만 그려지게 된다. 후에 Shadow Map을 가져와 그림자 계산을 진행할 때 위의 사진처럼 **겹치는 부분**이 발생하게 될 것이다. 해당 부분은 **Clip Space의 Z값을 비교**해서 어떤 Shadow Map을 사용할지 정하여 Shadow Map Sampling을 진행할 것이다.

X, Z축이 이루는 2차원 평면에서 수평각도를 알고 near와 far값을 알면 그 사이에 있는 X값을 구할 수 있다.

#### 1번 방법
![cameraFrustum](https://github.com/user-attachments/assets/1fb2b67c-9332-4f88-8729-a1c22275d8b5)

$${tan({FOV\over{2}})} = {x_{1}\over n} \tag{1}$$

$$x_{1} = ntan({FOV \over 2}) \tag{2}$$

```cpp
// 카메라 역행렬
Matrix InvCamera = camera->GetView().Invert();

// 시야각을 이용하여 수직 시야각 도출
float tanHalfVecticalFov = tanf(DirectX::XMConvertToRadians(fov / 2.0f));
// 수직 시야각을 이용하여 수평 시야각 도출
float tanHalfHorizontalFov = tanHalfVecticalFov * aspectRatio;

// 절두체를 나누기 위한 각 부분 절두체의 끝 지점 선언
m_cascadeEnd[0] = nearZ;
m_cascadeEnd[1] = 6.0f;
m_cascadeEnd[2] = 18.0f;
m_cascadeEnd[3] = farZ;

// 3개의 절두체로 나누기 위해 3번 반복
for(uint32_t i = 0; i < 3; ++i) {
	float xn = m_cascadeEnd[i] * tanHalfHorizontalFov;
	float xf = m_cascadeEnd[i + 1] * tanHalfHorizontalFov;
	float yn = m_cascadeEnd[i] * tanHalfVecticalFov;
	float yf = m_cascadeEnd[i + 1] * tanHalfVecticalFov;
	
	// i번째 절두체의 각 좌표
	Vector4 frustumCorners[8] = {// near Face
                                   {xn, yn, m_cascadeEnd[i], 1.0f},
                                   {-xn, yn, m_cascadeEnd[i], 1.0f},
                                   {xn, -yn, m_cascadeEnd[i], 1.0f},
                                   {-xn, -yn, m_cascadeEnd[i], 1.0f},
                                   // far Face
                                   {xf, yf, m_cascadeEnd[i + 1], 1.0f},
                                   {-xf, yf, m_cascadeEnd[i + 1], 1.0f},
                                   {xf, -yf, m_cascadeEnd[i + 1], 1.0f},
                                   {-xf, -yf, m_cascadeEnd[i + 1], 1.0f}};

	Vector4 frustumCenter = Vector4(0.0f);
	for(uint32_t j = 0; j < 8; ++j) {
		frustumCorners[j] = frustumCorners[j] * InvCamera;
		frustumCenter += frustumCorners[j];
	}
	frustumCenter /= 8.0f;
	
	// 각 절두체의 중심을 구한뒤 중심을 기준,
	// i만큼의 frustumCenter를 구할수 있고 이를 이용해서 view행렬과 Proj행렬을 구하면된다.
	.
	.
}
```

코드에서 설명한 대로 Frustum을 나누기 위해서 각 Frustum의 경계면을 정의하는 Z값을 정의한다. 해당 부분의 간격은 임의로 정의해도 되지만, MSFT의 [Cascade Shadow Map 문서](https://learn.microsoft.com/en-us/windows/win32/dxtecharts/cascaded-shadow-maps)에서는 각 간격을 정하는 방법이 소개되어 있다.

#### 2번 방법(사용한 방법)
위와 같은 방법을 Projection행렬과 Invert View행렬을 곱하여 카메라 Frustum을 World 상으로 옮길 수 있다. 해당 경우에는 OpenGL과 DirectX의 계산이 조금 달라진다는 점에 주의하자.

```cpp
float camera_near = camera.Near();
float camera_far = camera.Far();
float fov = camera.Fov();
float ar = camera.AspectRatio();
float f = 1.0f / CASCADE_COUNT;
// 절두체를 몇 개로 나눌 것인가
for (uint32 i = 0; i < split_distances.size(); i++)
{
	float fi = (i + 1) * f;
	float l = camera_near * pow(camera_far / camera_near, fi);
	float u = camera_near + (camera_far - camera_near) * fi;
	split_distances[i] = l * split_lambda + u * (1.0f - split_lambda);
}
```

**1번 방법**과 보기에 다른 부분은 $nearZ = 0.0$, $farZ = 30.0$이라고 한다면, 1번 방법은 고정된 숫자를 통해서 절두체를 나누고 있다. **2번 방법**은 [균등분할과 로그분할이 결합된 방식](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-10-parallel-split-shadow-maps-programmable-gpus)이다.

1. 균등 분할(Uniform Split) : 계산이 간단하고 Frustum의 크기가 일정하기 때문에 구현이 용이하다.
2. 로그 분할(Logarithmic Split) : 카메라 Frustum을 비선형적으로 나누는 방식이다.

### Frustum Far값 정하기
1. $f$라는 변수를 이용해서 frustum을 나누는 비율을 정한다.
2. $l$은 `pow()`함수를 통해 `logarithmic split`방식을 구현하였다. $Near \times {Far \over Near}^{f_i}$ 를 통해서 가까운 거리에 더 높은 해상도를 할당(작은 Far -> 세밀한 mapping)하고 먼 거리에 낮은 해상도를 할당(높은 Far -> 세밀하지 못한 mapping)한다.
3. $u$는 $Near+(Far-Near) \times f_i$를 통해 계산된다. 즉, Camera의 $z$값을 균등분할(`Uniform Split`)하여  $Near$에 더해주는 것이다.
4. $splitDistance[i]=l \times lambda + u \times (1.0f-lambda)$를 통해 최종 $Far$값을 결정한다. 여기서 $lambda$는 고정값으로 어떤 분할을 중점으로 할지 결정하는 값이다.

```cpp
std::array<Matrix, CASCADE_COUNT> projectionMatrices{};
projectionMatrices[0] = XMMatrixPerspectiveFovLH(fov, ar, camera_near, split_distances[0]);
for (uint32 i = 1; i < projectionMatrices.size(); ++i)
	projectionMatrices[i] = XMMatrixPerspectiveFovLH(fov, ar, split_distances[i - 1], split_distances[i]);
```

$splitDistance$값을 따로 저장해서 쉐이더에 Constant Buffer의 data로 바인딩해주는데, 그 이유는 향후 Shadow 판정 시 $splitDistance$값과 $viewPosition$의 $z$값을 비교해서 어느 해상도의 Texture2D를 사용할지 정하게 된다.

```cpp
for (uint idx = 0; idx < CASCADE_COUNT; ++idx)
{
    if (viewPos.z < shadowData.splits[idx])
    {
	    ...
```

이렇게 구한 $Far$값을 통해, Projection 행렬을 도출한다. 왜냐하면 해당 방식은 DirectXMath함수를 이용하는 방법이다. `BoundingFrustum frustum(ProjRow);`를 통해 절두체를 도출하기 때문에 기존의 Camera Frustum을 몇개로 나눌지 정하고 해당 갯수만큼 Projection행렬을 위와 같이 만들면 된다.

### Frustum 만들기
```cpp
BoundingFrustum frustum(projection_matrix);
frustum.Transform(frustum, camera.View().Invert());
std::array<Vector3, BoundingFrustum::CORNER_COUNT> corners{};
frustum.GetCorners(corners.data());

Vector3 frustum_center(0, 0, 0);
for (Vector3 const& corner : corners)
{
	frustum_center = frustum_center + corner;
}
frustum_center /= static_cast<float>(corners.size());
```

### Code
CASCADE_COUNT 횟수만큼 Draw Call을 호출하는 것은 비효율적이라고 생각하였고, 참고 자료에서도 Geomertry Shader를 통해서 한번의 Draw Call로 효과적으로 Cascade Shadow mapping을 진행하였다. GS에서 CASCADE_COUNT만큼 Texture를 그리기 위해서 Texture2DArray를 선언해주었다. 

```cpp
textureArrayDesc.ArraySize = CASCADE_COUNT;

D3D11_DEPTH_STENCIL_VIEW_DESC cascadeDsvDesc{};
ZeroMemory(&cascadeDsvDesc, sizeof(cascadeDsvDesc));
cascadeDsvDesc.Format = DXGI_FORMAT_D32_FLOAT;
cascadeDsvDesc.ViewDimension = D3D11_DSV_DIMENSION_TEXTURE2DARRAY;
cascadeDsvDesc.Texture2DArray.ArraySize = CASCADE_COUNT;
cascadeDsvDesc.Texture2DArray.FirstArraySlice = 0;
cascadeDsvDesc.Texture2DArray.MipSlice = 0;

D3D11_SHADER_RESOURCE_VIEW_DESC texArraySrvDesc{};
ZeroMemory(&texArraySrvDesc, sizeof(texArraySrvDesc));
texArraySrvDesc.Format = DXGI_FORMAT_R32_FLOAT;
texArraySrvDesc.ViewDimension = D3D11_SRV_DIMENSION_TEXTURE2DARRAY;
texArraySrvDesc.Texture2DArray.ArraySize = CASCADE_COUNT;
texArraySrvDesc.Texture2DArray.MipLevels = 1;
```

```cpp
[maxvertexcount(3 * CASCADE_COUNT)]
void ShadowCascadeGS(triangle VSToGS input[3], inout TriangleStream<GSToPS> triStream)
{
    for (int face = 0; face < CASCADE_COUNT; ++face)
    {
        GSToPS output;
        output.layer = face;
        for (int i = 0; i < 3; ++i)
        {
            output.posProj = mul(input[i].posWorld, shadowData.shadowCascadeMapViewProj[face]);
            output.texcoord = input[i].texcoord;
            triStream.Append(output);
        }
        triStream.RestartStrip();
    }
}
```

### Light Frustum Culling
Cascade Shadow Mapping은 Frustum을 나누어서 Mapping을 진행하는 원리이다. 그렇다면 Culling을 어떻게 진행해야할지 의문이다. 각 Frustum마다 Culling을 진행하면 Draw Call이 많아져 좋지 않다고 생각한다. 그러나, Geometry Shader를 이용해서 한번의 Draw Call로 Shadow Map을 그린다면, CASCADE_COUNT만큼의 Culling을 누적해줘야 한다.

현재 나는 빛 한번 당 한번의 Light Frustum Culling 호출을 통해 물체들의 빛 충돌 여부를 체크하고 있다. 그렇기 때문에 이전에 기록했던 값을 무시하고 Culling을 진행해도 되었지만, CSM에서는 그렇게 하면 가장 마지막 Frustum의 Culling만 적용될 것이다.

Lighting을 끝내고서 AABB를 통한 빛 충돌 여부를 초기화 해주고 `LightFrustumCulling`함수를 Frustum 체크 여부를 누적시키는 형태로 변경하였다.

```cpp
// true 면 그대로 두고, false면 충돌 여부 체크
AABB.isLightVisible = AABB.isLightVisible ? true : lightBoundingBox.Intersects(aabb.boundingBox);

// Render
mesh.Draw(m_context);
aabb.isLightVisible = false;
```

### Shadow Mapping VS Cascade Shadow Mapping
| ![그냥SM](https://github.com/user-attachments/assets/51e776af-7a86-45f7-b2af-356e7e27ce56) | ![CSM](https://github.com/user-attachments/assets/e19e4bf7-c7c2-498e-95fb-b99d9060aa6f) |
| ------------- | ------------ |
