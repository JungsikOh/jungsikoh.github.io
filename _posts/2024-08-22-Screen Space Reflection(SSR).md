---
layout: post
title: "Screen Space Reflection (SSR)"
description: "using 3D Ray Marching in View Space"
date: 2024-08-22
tags: Graphics
comments: true
use_math: true
---

## 참고
[https://www.slideshare.net/xtozero/screen-space-reflection](https://www.slideshare.net/xtozero/screen-space-reflection)
[https://lettier.github.io/3d-game-shaders-for-beginners/screen-space-reflection.html](https://lettier.github.io/3d-game-shaders-for-beginners/screen-space-reflection.html)
[https://rito15.github.io/posts/ray-marching/](https://rito15.github.io/posts/ray-marching/)
[https://casual-effects.blogspot.com/2014/08/screen-space-ray-tracing.html](https://casual-effects.blogspot.com/2014/08/screen-space-ray-tracing.html)
[https://zznewclear13.github.io/posts/screen-space-reflection-en/](https://zznewclear13.github.io/posts/screen-space-reflection-en/)


SSR의 고질적인 문제, 화면에 그려져 있는 것을 반사하다보니 카메라가 바닥만 보고 있는 경우 반사가 제대로 작동하지 못함.
## What is Screen Space Reflection(SSR)
3D 환경에서 반사를 어떻게 표현해야 하는가? 에 대한 기본적인 답은 임의의 평면에 놓인 거울을 상상하고 빛이 거울을 향하고 거울과 같은 매끄러운 표면에 부딪힌 빛은 거울의 노말 벡터 방향에 대해서 입사각과 동일한 각도로 반사된다.

그렇게 되면, 물체는 대칭으로 거울에 물체의 상이 맺히게 된다. 3D 환경에서는 거울에 상이 맵히는 위치에 물체를 한번 더 그리는 식으로 구현할 수 있다.

다만, 해당 방법은 물체를 한번 더 그려야 하므로, 거울이 많을 경우 Draw Call 횟수가 획기적으로 증가하게 됩니다.

Screen Space Reflection은 Screen Space의 데이터를 통해서 반사를 계산한다. 이번 프레임에 그린 장면의 데이터를 재사용하면 반사를 위해서 물체를 여러번 그리지 않아도 되므로 장면의 복잡도에 영향을 주지 않는다.

### How to implement Screen Space Reflection
### Ray March
특정 픽셀의 위치의 반사를 계산해본다고 가정하자. 카메라에서 해당 픽셀까지의 광선이 표면에 반사되어 어느 물체에 부딪힐지 추적해야 한다. View Space에서 카메리의 위치는 $(0, 0, 0)$이므로 해당 픽셀로 광선이 나아가는 방향이 픽셀의 View Space 위치가 된다.

직관적으로, 빛의 광선이 장면의 어느 지점에 도달하고 현재 물체의 노말 벡터를 기준으로  위치 벡터의 반대 방향($\vec{R}$)으로 이동한다. 그런 다음 물체에 도달하여 다시 위치 벡터의 반대 방향으로 이동한 후, 카메라에 도달하여 현재 물체에 반사되 장면의 어떤 지점을 볼 수 있게 된다. 

SSR은 이 빛의 경로를 역방향으로 추적하는 과정이다. 이 과정에서 빛이 반사되어 현재의 물체에 도달한 장면의 반사된 영역을 찾으려고 시도한다. 아래와 같은 코드를 통해 화면의 모든 픽셀에 대해서 반사벡터를 구할 수 있다.

```cpp
float3 refelctVec = reflect(pixelToView, viewNormal);
```

반사 벡터를 알고 있으므로 반사벡터의 방향으로 Ray Marching을 통해 광선이 충돌하는 위치를 알아내고 해당 위치의 texel을 가져오면 된다.

그렇다면, 광선과 어떤 표면의 충돌을 어떻게 구할 수 있을까?

$$Ray : R(t) = P +td \ \ (t>=0) \tag{1}$$

여기서, $d$는 $R(0)$에서 $t$까지 전진한 누적거리를 저장하게 된다. 따라서 현재 위치 $P$에서 전진한 누적거리 $td$를 더한게 [광선의 방정식](https://rito15.github.io/posts/ray-marching/)이다. 참고로 지금 문제에서 $P$는 카메라의 위치인 $(0, 0, 0)$이다.
## Using 3D Ray Marching in View Space
한없이 광선을 전진시킬 수 없으므로, MAX_STEP을 설절하여, 광선의 진행방향으로 일정 거리만큼 이동시켜 가며 충돌점을 찾아낸다.

Deferred Lighting할 때 Depth값과 texcoord를 통해서 viewSpace의 좌표를 구하는 법을 공부했을 것이다. 이 방법을 이용해서 물체를 따로 그릴 필요 없이 그려진 물체의 View Space에서 좌표를 알 수 있다.

따라서 $\vec{R}$와 $fragViewSpace$를 통해서 광선의 방정식을 구현할 수 있게 되었다.

$$Ray(t) = fragViewSapce + t\vec{R} \tag{2}$$

$(2)$와 같이 식을 만들 수 있다. 반복문을 통해서 매번 $\vec{R}$만큼 진행하게 된다. 광선이 나아감에 따라 반사 광선이 닿은 물체에 대한 확인을 하여야 되므로, 광선의 현재 위치에 대해서 **깊이 값**을 가져와야 한다.

갖고 있는 정보는 $1.$ 물체의 view space coord와 $2.\ \vec{R}$가 있다. $1.$번을 이용하여 해당 좌표에서의 texcoord를 구해 물체의 깊이값을 가지고 올 수 있다. 먼저, projection 행렬을 곱하면, clip space에서의 좌표값을 얻을 수 있다. 여기서 `w`값으로 나누면 NDC값이 나오게 된다. 해당 값을 $[0, 1]$의 범위로 변환시켜주고 texcoord 좌표 형식에 맞게 y값을 뒤집어 주면 현재 광선에 위치의 texcoord를 얻을 수 있다.

```cpp
float4 projectedCoord = mul(float4(hitCoord, 1.0), frameData.proj);
projectedCoord.xy /= projectedCoord.w;
projectedCoord.xy = projectedCoord.xy * float2(0.5, -0.5) + float2(0.5, 0.5);

depth = DepthTex.SampleLevel(PointClampSampler, projectedCoord.xy, 0).r;
float3 samplePosition = GetViewSpacePosition(projectedCoord.xy, depth);
```

여기서 중요한 점은 Depth Texture를 샘플링하는 것이다. 선형보간은 다른 깊이 값들끼리의 두면이 교차한다고 인식하는 실수가 발생할 수 있다. 따라서, 이러한 교차점들을 제외하기 위해 `Point Clamp Sampler`를 활용해야 한다.

```cpp
float diff = hitCoord.z - samplePosition.z;
```

광선의 깊이 값과 광선 위치에 있는 물체의 깊이 값을 비교해 `-`면 광선이 교차하는 것이고 `+` 교차하지 않는 것으로 판단하게 된다. 여기서 더 정확하게 판단하기 위해 `Binary Search`가 사용된다. 왜냐하면 Color 값을 샘플링을 통해 갖고 오게되는데, 정확한 color 값을 갖고 오기 위함이다.

Binary Search는 광선의 진행방향에 대해 `*0.5f`를 해줌으로써 진행된다. Binary Search는 정해놓은 STEP만큼 진행되며, $Depth_{Ray} - Depth_{Frag}$가 음수면 물체가 더 깊이 있다는 것이므로 $P + \vec{R}$을 해준다. 해준 뒤, $0.5$를 $\vec{R}$에 다시 곱해준 뒤 광선을 뒤로 이동시켜서 더욱 더 정확한 위치를 찾아내는 것을 반복한다.

```cpp
if (diff <= 0.0)
{
    hitCoord += dir;
}
dir *= 0.5;
hitCoord -= dir;
```

이렇게 BINARY_STEP만큼 진행한 최종 위치에 대해여 최종적으로 깊이값의 차이를 구하고 해당 값이 설정한 두께 값 이내의 값이면 반사값을 그리는 것이 결정된다.

다만, SSR에는 문제점이 있는데 화면에 있는 정보만을 가져와서 그리다보니 여러가지 문제점이 발생한다. 먼저, 화면 끝에 있는 반사값에 대해 그 너머의 반사값을 알지 못하므로 똑같은 값을 계속 가져와 늘어지는 현상이 발생한다는 점이다.

![SSR에러1](https://github.com/user-attachments/assets/9dab5db0-d0e0-48f3-8ed8-4d2739c25a47)

이러한 문제점을 고치기 위해 할 수 있는 수단이 여러가지 있는데, 간단하게 생각할 수 있는 것은 반사 범위를 제한해 반사 값을 가져오지 못하는 구간에 대해서는 SSR을 진행시키지 않는 것이다.

![SSR에러1_1](https://github.com/user-attachments/assets/2256c0a9-b632-479a-aeac-ad7d37ba44e4)

하지만, 이러한 해결방법은 반사가 너무 **끊어져** 보인다는 단점이 있다. 그렇기 때문에 Screen Space에 대해서 Fade Out을 적용한다. 해당 방안의 원리는 반사값이 Screen Space에 끝자락에서 가져온 것이라면 Fade Out을 시키는 것이다.

```cpp
float2 coordsEdgeFactor = float2(1.0, 1.0) - pow(saturate(abs(coords.xy - float2(0.5, 0.5))) * 2.0, 4.0);
float screenEdgeFactor = saturate(min(coordsEdgeFactor.x, coordsEdgeFactor.y));
```

![SSR에러1_2](https://github.com/user-attachments/assets/33c07596-b0fc-4c86-b292-fbf04bb91b4e)

아직 많이 아쉽긴 하지만 훨씬 자연스러운 결과물을 볼 수 있다. 더욱 정확한 반사 값을 찾기 위해 DDA알고리즘을 이용해 Clip Space에서 2D Ray Marching을 하는 기술이 있다. 2D Ray Marching을 하는 이유는 View Space에서의 3D Ray Marching을 한 후 원근 투영을 적용하게 되면 먼거리에 있는 물체는 많은 정보가 필요하지 않아도 가까운 거리에 물체랑 똑같은 정보량을 갖고 오게 되므로 비효율적으로 동작하게 된다는 것이다. 참고한 자료에 해당 자료들이 있으므로 참고하길 바란다.

추가적으로, DX12나 Vulkan에서 Ray Tracing 기법을 통해 반사를 그리면 해당문제를 해결할 수 있다.
