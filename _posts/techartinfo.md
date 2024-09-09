---
layout: post
title: test
description: "About AABB And Rotation Matrix"
date: 2024-08-01
tags: AABB Matrix Graphics
comments: true
use_math: true
permalink: /techartinfo/
---

## 참고
[https://blog.naver.com/sorkelf/40169749877](https://blog.naver.com/sorkelf/40169749877)
[https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nn-d3d11-id3d11resource](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nn-d3d11-id3d11resource)

## ID3D11 Resource
[MSDN](https://learn.microsoft.com/en-us/windows/win32/api/d3d11/nn-d3d11-id3d11resource)에서 알 수 있듯이, ID3D11Resource는 메모리 블럭이 아닌 <mark>interface</mark>라고 소개된다. 

해당 데이터의 interface라고 생각하면 된다. ID3D11Resource를 통해 GPU(DEFAULT, DYNAMIC)나 CPU(STAGING) 메모리에 간섭가능한 interface를 생성하는 것이다.
### How to write something to Memory
기본적으로 DEFAULT인 경우에는, ID3D11Resource가 RenderTarget이나 DepthStencil과 같은 객체들에 묶여있는 경우로써 따로 메모리에 쓰는 작업을 하지 않아도 된다. 왜냐하면, `Draw`, `DrawIndexed`와 같은 함수를 거치면서 ID3D11Resource 인터페이스가 가리키는 메모리에 알아서 작성되기 때문이다.

앞의 두 객체에 묶여있지 않더라도 Resource자체를 `CreateBuffer`와 같은 함수로 생성할 때 `SubResourceData`를 통해서 생성하는 동시에 메모리에 작성 할 수 있다.
```cpp
device->CreateBuffer(&bufferDesc, &initData, reinterpret_cast<ID3D11Buffer**>(&m_resource))
```

만약 DYNAMIC인 경우, 두가지 방법이 있다.
1. `Create`함수를 호출할 때 작성하는 방법
2. `Map`, `Unmap`을 통해서 작성하는 방법

#### Map and Unmap
```cpp
// if buffer Usage is Dynamic, Update the Buffer by srcData.
template <typename T_DATA>
void Update(ID3D11DeviceContext* context, const T_DATA& srcData,
            uint64 dataSize) {
    if (m_desc.resourceUsage == DXResourceUsage::Dynamic) {
        memcpy(Map(context), &srcData, dataSize); // context->Map(m_resource, 0, D3D11_MAP_WRITE_DISCARD, 0, &ms);
        UnMap(context); // context->Unmap(m_resource, 0);
    }
}
```
1. `context->Map()`을 통해서 CPU에서 데이터를 GPU에 전송하는 데 사용되는 메모리 영역을 인터페이스에 할당한다.
2. `memcpy()`를 통해 할당한 메모리 영역에 데이터를 복사한다.
3. `context->Unmap()`을 호출하여 데이터를 GPU에 전송한다.

위와 같은 과정을 거쳐서 GPU의 값을 실시간으로 업데이트 가능하다.

### Release ID3D11Resource
인터페이스의 역할을 하는 ID3D11Resource의 특징으로 여러 개의 RenderTarget이나 ShaderResourceView를 생성할 때, 단 하나의 ID3D11Resource를 사용하는 것이 가능하다.
```cpp
device->CreateTexture2D(&texDesc, nullptr, reinterpret_cast<ID3D11Texture2D**>(&m_resource));

device->CreateRenderTargetView(reinterpret_cast<ID3D11Texture2D*>(m_resource), &rtvDesc, &rtv);
```
ID3D11Resource를 통해 RenderTarget을 만들때는 위와 같은 과정을 거친다. ID3D11Resource에 메모리 영역을 할당해준 뒤, 해당 영역에 RenderTarget을 만드는 것이다. 

여기서 중요한 특징은 ID3D11Resource는 단순히 메모리 영역에 대한 인터페이스일 뿐이라서, `Release()`를 해줘도 된다는 점이다. 그러나, `Release` 해준 뒤 RenderTarget에 대한 ShaderResourceView를 생성하게 되면, 다른 메모리 영역에 대해서 설정이 되기 때문에 원치 않은 결과가 나오게 될 것이다.

그렇기 때문에, 하나의 Resource를 복수의 RTV나 SRV를 생성할 수 있지만 해제하고서, 다시 해당 메모리 영역을 다시 접근해 SRV를 생성하는 것은 불가능하므로 이점에 유의해서 사용해야 한다.

하지만, 특정 메모리 영역을 사용해 이미 만든 `RTV`나 `SRV`가 존재한다면, `GetResource()`함수로 해당 메모리 영역의 인터페이스에 접근이 가능하다.
```cpp
ID3D11Resource* rtvResource = nullptr;
rtv->GetResource(&rtvResource);
```