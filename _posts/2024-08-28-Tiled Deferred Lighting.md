---
layout: post
title: "Tiled Deferred Lighting"
description: "using Compute Shader"
date: 2024-08-28
tags: Graphics, Lighting
comments: true
use_math: true
---

## 참고
1. [https://www.intel.com/content/www/us/en/developer/articles/technical/deferred-rendering-for-current-and-future-rendering-pipelines.html](https://www.intel.com/content/www/us/en/developer/articles/technical/deferred-rendering-for-current-and-future-rendering-pipelines.html)
2. [https://www.aortiz.me/2018/12/21/CG.html#clustered-shading](https://www.aortiz.me/2018/12/21/CG.html#clustered-shading)
3. [https://lalyns.tistory.com/entry/%ED%88%AC%EC%98%81-%ED%96%89%EB%A0%AC-%EC%9C%A0%EB%8F%84%ED%95%98%EA%B8%B0](https://lalyns.tistory.com/entry/%ED%88%AC%EC%98%81-%ED%96%89%EB%A0%AC-%EC%9C%A0%EB%8F%84%ED%95%98%EA%B8%B0)
4. [https://espace.library.uq.edu.au/data/UQ_385844/tiled_shading_preprint.pdf?Expires=1724480510&Key-Pair-Id=APKAJKNBJ4MJBJNC6NLQ&Signature=azuf2T41aQCxHudsWD99a0A2TFmw4rf2fec6iHDjH7xCOKzmeubW8oxteweGcuro6sw-gm~XuVj75gZQdtAJfKC9rJFRu8fgVgJUMrGM6wCik-WBj6xeF-FZ8EyjsACNuhQxt2o0ANF-AnwSR-3E7ElulJkiuAbaY1myr7jiSFRWjkjviEDDjUVEKBync0I8mxVMZHnMQ9zU4jXRF-pFTYbUbXH6ttbOQ1oCdYEU7pdoWzbsgA95P1JCpnfZApiAzi144NkC8YCHd4AZ0fIs09aVXIEswX7NO~g7G3Mnk-YPudTsoE1lhgYMLvHe0aZadG5018rCjE8wipjBIvZz9A__](https://espace.library.uq.edu.au/data/UQ_385844/tiled_shading_preprint.pdf?Expires=1724480510&Key-Pair-Id=APKAJKNBJ4MJBJNC6NLQ&Signature=azuf2T41aQCxHudsWD99a0A2TFmw4rf2fec6iHDjH7xCOKzmeubW8oxteweGcuro6sw-gm~XuVj75gZQdtAJfKC9rJFRu8fgVgJUMrGM6wCik-WBj6xeF-FZ8EyjsACNuhQxt2o0ANF-AnwSR-3E7ElulJkiuAbaY1myr7jiSFRWjkjviEDDjUVEKBync0I8mxVMZHnMQ9zU4jXRF-pFTYbUbXH6ttbOQ1oCdYEU7pdoWzbsgA95P1JCpnfZApiAzi144NkC8YCHd4AZ0fIs09aVXIEswX7NO~g7G3Mnk-YPudTsoE1lhgYMLvHe0aZadG5018rCjE8wipjBIvZz9A__)
5. [https://www.digipen.edu/sites/default/files/public/docs/theses/denis-ishmukhametov-master-of-science-in-computer-science-thesis-efficient-tile-based-deferred-shading-pipeline.pdf](https://www.digipen.edu/sites/default/files/public/docs/theses/denis-ishmukhametov-master-of-science-in-computer-science-thesis-efficient-tile-based-deferred-shading-pipeline.pdf)

## Why we need Tiled Deferred Lighting?
기존 Deferred Lighting은 아래와 같은 코드로써 동작하였다. 그렇기 때문에 아무리 많은 물체를 그려도 물체마다 라이팅을 적용해야한다는 단점이 없어졌지만, 또다른 단점으로 **모든 빛에 대해서 Lighting 계산**을 해야된다는 문제가 발생하였다.

```cpp
for each object do
	G-Buffer = lighting properties of object;
for each light do
	frameBuffer += light_model(G-Buffer, light);
```

이를 `Bandwidth overhead when lights overlap` 문제라고 언급된다. 중첩이 문제가 되는 이유는 Lighting할 때 **GBuffer 데이터에 접근**해야하는데, 조명이 중첩되어 있을수록 접근하는 횟수가 늘어나게 된다. 이로 인해 **메모리 대역폭이 크게 증가**한다는 것이다.

![tile1](https://github.com/user-attachments/assets/c0ccb22c-68d1-4f7c-950a-59e65557fd64)

적은 조명이 겹쳐있는 경우에는 화면을 나누고 각 Grid마다 데이터를 세팅하는데 시간이 걸려 속도가 기존의 Lighting방법보다 느리지만, 조명이 늘어날 수록 확연한 차이를 보여준다는 것을 볼 수 있다.

## About Tiled Deferred Lighting
직관적으로 본다면, Screen Space를 grid영역을 구분하여 각 영역마다 Light가 영향을 주는지를 확인하고 해당 영역에 LightIndex를 추가한다. 나중에 해당 영역의 Lighting의 총량을 계산을 할 때 LightIndex를 참고해서 영향을 주는 Lighting만을 계산하는 것이다.

![tile2](https://github.com/user-attachments/assets/ae4b6650-d963-4e9c-ab54-6955898dd7ee)

(Image : [https://www.aortiz.me/2018/12/21/CG.html#clustered-shading](https://www.aortiz.me/2018/12/21/CG.html#clustered-shading))

$Tiled$란 이름이 사용되는 만큼 화면에 대해 영역(Group)을 나눈다. Tile이란 이름 때문에 2D 공간을 가진 tile은 아니다.

조명의 위치는 3D 공간에 있기 때문에 해당 Tile의 $Z_{min}$와 $Z_{max}$를 구해서 육면체 형태의 Tile을 구성한다.

## Implement
Tiled Deferred Lighting을 구현하기 위해 필요한 것은 아래와 같다.

1. 각 Tile의 $Z_{min}$와 $Z_{max}$
2. Light Indices : 어떤 조명이 누적되었는지 파악
3. Number of total lights : 해당 타일에 누적된 조명의 개수를 파악

해당 계산은 Compute Shader를 통해서 계산되는데, Compute Shader는 Dispatch를 통해 Group을 만들게 된다. GroupID -> GroupThreadID가 존재하고 각 Thread마다 GroupIndex를 가지고 있다. 그렇기 때문에 우리는 이번 GroupID에 대해 Lighting 계산하기 전에 만약, 처음 계산하는 것이라면 값의 초기화가 필요하다.

일단, 타일(Group)의 육면체를 구성하기 위해서는 가로, 세로, 깊이 값이 필요하다. 먼저, 깊이 값을 어떻게 구하는지를 살펴보자.
 
```cpp
float viewSpaceZ = surfaceSamples[sample].positionView.z;
bool validPixel = 
     viewSpaceZ >= mCameraNearFar.x &&
     viewSpaceZ <  mCameraNearFar.y;
[flatten] if (validPixel) {
    sampleMinZ = min(minZSample, viewSpaceZ);
    sampleMaxZ = max(maxZSample, viewSpaceZ);

// Initialize shared memory light list and Z bounds
if (groupIndex == 0) {
    sTileNumLights = 0;
    sNumPerSamplePixels = 0;
    sMinZ = 0x7F7FFFFF;      // Max float
    sMaxZ = 0;
}

GroupMemoryBarrierWithGroupSync();
```

현재 GroupID에서 담당하고 있는 모든 픽셀에 대해서 sampleDepth의 초기화가 끝났다면, 이제 Group의 $Z_{min}$과 $Z_{max}$를 정할 차례이다. 각 Group의 데이터에 대해서 원자성이 보장이 되어야하므로, `Interlocked` 키워드 함수를 통해서 값을 결정한다.

```cpp
    if (sampleMaxZ >= sampleMinZ) {
        InterlockedMin(sMinZ, asuint(sampleMinZ));
        InterlockedMax(sMaxZ, asuint(sampleMaxZ));
    }

    GroupMemoryBarrierWithGroupSync();
```

이렇게 우리는, 모든 Group에 대해서 각 Group의 최소 깊이와 최대 깊이에 대해서 파악하였다. 다음은 육면체를 만들기 위해서 필요한 것은 해당 Group이 가지는 가로와 세로이다.
#### Frustum Calculation
타일의 크기에 대해서 결정을 해야한다. 각 타일은 기본적으로 화면의 가로폭과 세로폭을 기반으로 한다. 우리는 이 화면을 타일의 개수(상수 값)를 통해서 일정하게 나누어야 한다. 

$$TileScale_{xy} = {window_{xy} \over 2 \times TILE\_GROUP\_SIZE} \tag{1}$$

화면을 타일 그룹 크기만큼 나눈다면, 화면에 대한 TileScale을 알 수 있는 $2$를 곱하는 이유는 타일의 중심점을 기준으로 계산하기 위함이다. 좌표계는 $(0, 0)$부터 시작이 아닌 $(-1, -1)$범위부터 시작하기 때문이다.

이렇게 계산함으로써, 화면 전체를 기준으로 각 타일의 상대적인 크기를 구할 수 있게된다.

화면을 기준으로한 타일의 상대적인 크기를 구했다면, 다음은 타일을 화면에 어떻게 위치시킬지가 관건이다. 

해당 문제를 해결하기 위해 투영행렬에 대해서 알아야 한다. 투영행렬은 기본적으로 $(2)$와 같이 구성된다. $P$ 행렬은 카메라가 바라보는 프러스텀의 형태를 정의한다.

$$ P = \begin{bmatrix}{1 \over aspect\cdot tan({FOV \over 2}) }&0&0&0\\0&{1 \over tan({FOV \over 2}) }&0&0\\0&0&{z_f + z_n \over zn - zf}&{2 \cdot z_f \cdot z_n \over zn - zf}\\0&0&-1&0\\\end{bmatrix} \tag{2} $$​​
그렇다면, 투영행렬은 $(2)$와 같은데 프러스텀은 어떻게 만드는건가? 바로 투영행렬의 값들의 계산을 통해 이루어진다.

$$Plane_{left} = m_4 + m_1$$
$$Plane_{right} = m_4 - m_1$$
$$Plane_{top} = m_4 - m_2$$
$$Plane_{bottom} = m_4 + m_2$$
$$Plane_{near} = m_4 + m_3$$
$$Plane_{far} = m_4 - m_3 \tag{3}$$

프로스텀의 각 평면에 대해서 $P$ 행렬을 이용하여 $(3)$과 같이 계산함으로써 도출할 수 있다. 우리는 여기서 $left,\ right, \ top, \ bottom$이 필요하다. 그러므로, 투영행렬을 전부다 계산할 필요는 없고, $m1,\ m2,\ m4$에 대해서만 계산을 하면 된다.(오른손 좌표계 기준)

```cpp
float4 c1 = float4(mCameraProj._11 * tileScale.x, 0.0f, tileBias.x, 0.0f);
float4 c2 = float4(0.0f, -mCameraProj._22 * tileScale.y, tileBias.y, 0.0f);
float4 c4 = float4(0.0f, 0.0f, 1.0f, 0.0f);
```

카메라의 projection 행렬을 곱하는 것이 보일텐데, 이는 기존 카메라가 갖고 있던 projection 행렬의 스케일값을 갖고 오는 것이다. 해당 값을 tile 스케일로 조정한다고 보면 될 것이다. 두가지 의문이 생길 것이라고 본다.

첫째는 $y$좌표에 대해서 `-`를 곱해주고 있는 것을 볼 수 있는데 이는 **Clip Space**에서는 일반적으로 **우측 상단**이 $(+X, +Y)$이지만, **Screen Space**에서는 **우측 하단**이 $(+X, +Y)$이므로 $y$좌표에 대해서 `-`를 곱해주는 것이다.

두번째는 $tileBias$의 존재이다. tileBias는 아래 코드와 같이 정의되는데, 해당 변수의 용도는 타일 스케일과 타일의 위치(ID)를 기반으로 계산된 offset이다. 

이 변수는 타일의 중심을 기준으로 타일이 위치하는 공간을 정의하게 된다. 타일의 중심을 원점으로 맞추기 위한 보정 값으로 타일의 좌표를 조정하여 타일의 중앙(tileScale)이 타일 좌표계에서 $(0, 0)$이 되도록 한다.

```cpp
float2 tileBias = tileScale - float2(groupId.xy);
```

##### Why we calculate tileBias? (by GPT)
tileBias가 필요한 이유는 각 타일이 독립적으로 자신의 프러스텀 평면을 계산해야하기 때문이다. 해당 값이 투영 행렬 값으로 들어가게 되면서, 계산하고 있는 타일이 화면 전체에서 **어디에 위치**하는지, 그리고 그 위치가 **원점(TileScale)으로부터 얼마나 떨어져 있는지**를 나타낸다.

만약 타일의 위치를 고려하지 않고 전체 화면 투영을 사용하게 되면 조명 계산이 정확하지 않거나 효율이 떨어질 수 있다.

이렇게 프러스텀을 계산할 투영행렬을 모두 구하였다. 참고로 $far$와 $near$ 프러스텀에 대해서 아래와 같이 정의되는 이유는 평면방적식이 $Ax+By+Cy+D=0$ 이렇게 계산되는데, $near$는 $z$방향이 양수이고 $far$는 음수이기 때문에 그렇다.   

```cpp
// Derive frustum planes
float4 frustumPlanes[6];
// Sides
frustumPlanes[0] = c4 - c1;
frustumPlanes[1] = c4 + c1;
frustumPlanes[2] = c4 - c2;
frustumPlanes[3] = c4 + c2;
// Near/far
frustumPlanes[4] = float4(0.0f, 0.0f,  1.0f, -minTileZ);
frustumPlanes[5] = float4(0.0f, 0.0f, -1.0f,  maxTileZ);

// Normalize frustum planes (near/far already normalized)
[unroll]
for (uint i = 0; i < 4; ++i) {
    frustumPlanes[i] *= rcp(length(frustumPlanes[i].xyz));
}
```

![TileBias](https://github.com/user-attachments/assets/4b877281-c61c-49b1-9d6e-0451c8b46bad)

드디어, 타일이 가지는 프러스텀에 대한 정의가 모두 끝났다. 다음은 타일이 조명의 영향력을 받는지 안 받는지 판단하기 위한 과정을 거쳐야 한다.

#### Cull lights for this tile
조명을 누적시키는데 있어서 중요한 것은 한 조명이 타일을 한번씩만 돌아야 한다는 것이다. 그리고 우리는 모든 조명에 대해서 병렬적으로 처리하길 원한다. 코드를 한번 살펴보자.

```cpp
    for (uint lightIndex = groupIndex; lightIndex < totalLights; lightIndex += TILED_GROUP_SIZE * TILED_GROUP_SIZE)
    {
        PackedLightData light = LightsBuffer[lightIndex];       
        if (!light.active || light.castsShadows) continue;
        
        bool inFrustum = true;
        if(light.type != DIRECTIONAL_LIGHT)
        {
            [unroll]
            for (uint i = 0; i < 6; ++i)
            {
                float d = dot(frustumPlanes[i], float4(light.position.xyz, 1.0f));
                inFrustum = inFrustum && (d >= -light.range);
            }
        }

        [branch]
        if (inFrustum)
        {
            uint listIndex;
            InterlockedAdd(TileNumLights, 1, listIndex);
            TileLightIndices[listIndex] = lightIndex;
        }
    }
    
	GroupMemoryBarrierWithGroupSync();
```

하나하나씩 살펴보도록 하자. 여기서 for문을 자세히 보면, `lightIndex = groupIndex`로 설정된 것을 볼 수 있다. `groupIndex`는 타일(groupID) 내 thread(GroupThread)의 index이다. 즉, 타일 내 모든 thread는 `LightsBuffer[]`에 대해서 다른 조명 인덱스부터 시작하게 된다. 이를 통해서 여러 thread가 동일한 타일에서 서로 다른 조명을 동시에 처리할 수 있게 되는 것이다. 다만, 이로 인해 생기는 단점은 thread의 수만큼 타일 내 조명 계산을 못한다는 것이다.

`lightIndex += TILED_GROUP_SIZE * TILED_GROUP_SIZE`을 해주는 것도 위의 이유 때문이다. 타일 내 모든 thread가 이미 병렬적으로 조명 영향력에 대해서 계산하므로, `lightIndex`에다가 해당 값을 더해주는 것이다. 해당 값을 더해주면 자동적으로 `totallights`보다 커지거나 같아지므로 반복문을 종료하게 된다.

```cpp
for (uint lightIndex = groupIndex; lightIndex < totalLights; lightIndex += TILED_GROUP_SIZE * TILED_GROUP_SIZE)
```

어떤 조명의 영향력을 계산할지에 대한 문제는 해결하였다. 이제 조명이 해당 타일에 영향을 미치는지 확인할 차례이다. 

dot연산의 통해서 프러스텀과 조명의 위치관계를 밝힐 수 있다. 

$$d = a \cdot b = ||a||||b||cos\theta = ||a-b|| \tag{4}$$

만일 $\vec{unit}$ 라면, $d > 0$이면, 프러스텀보다 조명이 앞에 있다는 것을 의미하고 $d = 0$이면, 조명의 위치가 프러스텀보다 위에 있다는 것을 의미한다. $d < 0$이면 프러스텀 뒤에 조명이 있다는 의미이다.

우리는 해당 개념을 사용해 조명의 영향력이 닿는지 알아내야 한다. 조명과 프러스텀의 거리가 조명의 영향력보다 크거나 같다면, 일부라도 겹친다는 것을 의미한다.

```cpp
bool inFrustum = true;
if(light.type != DIRECTIONAL_LIGHT)
{
    [unroll]
    for (uint i = 0; i < 6; ++i)
    {
        float d = dot(frustumPlanes[i], float4(light.position.xyz, 1.0f));
        inFrustum = inFrustum && (d >= -light.range);
    }
}
```

만일 inFrustum이 최종적으로 `true`라면, TileNumLights를 1 증가시키고 해당 값을 listIndex에 넘겨주어 tile에 어떤 lightIndex가 있는지 파악할 수 있도록 한다.

```cpp
[branch]
if (inFrustum)
{
    uint listIndex;
    InterlockedAdd(TileNumLights, 1, listIndex);
    TileLightIndices[listIndex] = lightIndex;
}
```

#### Calculate Lighting
타일에 대한 프러스텀도 만들고 해당 프러스텀에 대한 조명 검사도 모두 끝났다. 이제 해당 타일을 처리하고 있는 thread에게 Lighting에 대한 계산을 처리하면 된다.

Lighting 부분은 대부분의 계산이 Pixel Shader와 동일하게 수행된다. 다만 몇 가지 다른 부분은 조명을 하나하나씩 처리하는게 아니고 `TileNumLights`만큼 누적하여 처리한다는 점이다.

LightData의 구조체를 그대로 사용하지 않는 이유는 LightsBuffer의 크기가 커지면 메모리적으로 비효율적이므로, Lighting 계산에 필수적인 부분만을 담아두는 구조체를 따로 만든 것이다.

```cpp
for (int i = 0; i < TileNumLights; ++i)
{
    PackedLightData packedLightData = LightsBuffer[TileLightIndices[i]];
    LightData light = ConvertFromPackedLightData(packedLightData);
    switch (light.type)
    {
    ...
    }
}
```

다른 thread에서 누적시킨 값이 있으면 거기에 더해서 누적시켜야 되므로 아래와 같은 코드가 필요하다.

```cpp
float4 shadingColor = OutputTx.Load(int3(int2(dispatchThreadId.xy), 0)) + float4(Lo, 1.0f);
OutputTx[dispatchThreadId.xy] = shadingColor;
```

### 다른 방법(CS->PS)
Compute Shader에서는 dtID를 통해서 어떤 위치에 어떤 조명이 배치되는지만 저장 한 후에 Pixel Shader에서 Lighting을 하는 방법이다. 다만 해당 방법은 CS에서 값을 저장할 때 너무 큰 Buffer가 Bind되었다가 UnBind되므로, 속도가 너무 느려진다는 문제점이 발생하였다.

매우 단순하게 가정하면, 화면 크기의 길이(width * height)를 가진 배열에다가 조명정보를 저장해야한다. `float`와 `float[MAX_LIGHTS]`로 구성된 구조체이므로 엄청나게 많은 메모리가 Bind되었다가 UnBind 되는 것이다. CPU-GPU 통신은 상대적으로 매우 느리므로 비용이 상당하다.