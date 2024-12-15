---
layout: post
title: "[My Renderer] Apply Batch System in Vulkan"
description: "Simple Batching"
date: 2024-12-15
tags: Graphics Batching Indirect
comments: true
use_math: true
---

## 도입하는 이유

그래픽스 API를 사용하다보면은 `Vertex Buffer`와 `Index Buffer`를 다루게 된다. `vkCreateBuffer`를 통해서 실행하게 되는데, CPU에서 해당 명령어를 호출하는 순간 Vulkan 드라이버가 CPU에서 동작하며, GPU가 생성 명령을 처리하게 된다.

다만, Mesh마다 Buffer를 만들게 되면 명령 호출이 잦아져 성능 저하를 만들어 내게 된다. 그렇기 때문에 Batching을 도입하여, 해당 부분의 성능 저하를 없애기로 결정하였다.

## How Batch System works?

```cpp
struct MiniBatch
{
    VkBuffer m_vertexBuffer = nullptr;
    VkDeviceMemory m_vertexBufferMemory;
    VkBuffer m_indexBuffer = nullptr;
    VkDeviceMemory m_indexBufferMemory;

    uint32_t m_currentVertexOffset = 0;
    uint32_t m_currentIndexOffset = 0;
    uint32_t m_currentBatchSize = 0;

    std::vector<VkDrawIndexedIndirectCommand> m_drawIndexedCommands;
};
```

Batching의 구조는 매우 단순하다. `offset`은 `indirectCommands`에다가 특정 mesh가 가지는 vertex 정보의 좌표를 가리키는 역할을 수행한다. `batchSize`는 Batching에서 한 batch마다 최대 특정 사이즈까지만 저장하기 위해 추가한 변수이다. 해당 변수가 mesh의 정보가 누적될 때마다 그에 대한 메모리 용량을 기록해서 현재 mini batch가 다 차면 다음 batch를 생성할 수 있도록 도와준다.

```cpp
for(auto object : objectList) {
	SetVertexBuffer();
	SetIndexBuffer();
	Draw();
}
```

```cpp
for(auto batch : batches) {
	SetVertexBuffer();
	SetIndexBuffer();
	IndirectDraw();
}
```

이러한 식으로 반복문이 변하게 된다. 이렇게 되면 1000개의 물체에 대해서 1000번 Draw해야 했던 것을 Batch 시스템과 Indirect를 도입함으로써, 하나의 Batch를 통해서 한번 Draw 함수를 사용하는 것만으로 렌더링이 가능해진다.

![image](https://github.com/user-attachments/assets/57a3a594-0d66-406e-9a5e-3d6613695fab)

위의 그림과 같이, Mini Batch의 정보를 담은 하나의 큰 `VkBuffer`가 있고 `IndirectCommands`의 정보를 갖고 있는 `VkBuffer`를 통해서 Indirect Draw Call을 수행하는 것이다.

`IndirectCommands`가 Draw에 대한 정보를 갖고 있다는 것을 주의 깊게 보자. 우리는 이것을 이용해서 Compute Shader에서 IndirectCommands의 숫자를 조정해 Culling을 수행할 수도 있다. Culling을 수행할 때 indirectCommand의 instanceCount를 조정하게 되는데, `instanceCount = 1`인 경우에는 단순히 0으로 변경해줘서 렌더링을 수행하면 된다.

`instanceCount = n`인 경우에는 쉐이더 내에서 사용할 몇 개의 배열들을 추가해 주어야 한다.

```cpp
layout(set = 0, binding = 2) buffer VisibleInstances {
    uint visibleInstances[];
};

layout(set = 0, binding = 3) buffer VisibleCount {
    uint visibleCount;
};

.
.
.

// 몇 번째 Object인지에 대한 idx (Compute Shader의 dispatch 값에 따라 달라짐 주의)
uint idx = gl_GlobalInvocationID.x;
if (isVisble()) {
        // visibleCount를 atomicAdd로 증가시키고 그 위치에 이 인스턴스 인덱스를 저장
        uint cIndex = atomicAdd(visibleCount, 1);
        visibleInstances[cIndex] = idx;
    }
```

```cpp
uint originalIndex = visibleInstances[gl_InstanceIndex]; 
    mat4 model = modelMatrices[originalIndex].model;

    gl_Position = pushConstants.viewProjection * model * vec4(inPosition, 1.0);
```

위의 코드처럼 살아있는 instance의 model 행렬의 위치를 알아야 하기 때문에 `SSBO`를 추가해준다. 향후 vertex shader에서 해당 배열을 사용해서 살아남은 `index`를 가져온 후, 그 `index`를 이용해서 model 행렬 배열에 접근하는 것이다.

만일 Texture가 있다면, Texture도 `TextureArray[]` 형식으로 저장해서 위와 같이 접근해서 사용하면 된다.

## Code

```cpp
static void AddDataToMiniBatch(std::vector<MiniBatch>& miniBatches, VkUtils::ResourceManager& manager, const Mesh& mesh, bool flag = false) {
    // 현재 메쉬 데이터 크기 계산
    size_t vertexDataSize = mesh.vertexCount * sizeof(BasicVertex);
    size_t indexDataSize = mesh.indexCount * sizeof(uint32_t);
    size_t totalDataSize = vertexDataSize + indexDataSize;
    MiniBatch* currentBatch = nullptr;
    if (!miniBatches.empty()) {
        currentBatch = &miniBatches.back();
    }
    else {
        // mini-batch가 없으면 새로운 batch 생성
        MiniBatch newBatch;
        miniBatches.push_back(newBatch);
        currentBatch = &miniBatches.back();
    }

    // Draw Command 생성 및 추가
    VkDrawIndexedIndirectCommand drawCommand{};
    drawCommand.indexCount = mesh.indexCount;
    drawCommand.instanceCount = 1;
    drawCommand.firstIndex = accumulatedIndexSize / sizeof(uint32_t);
    drawCommand.vertexOffset = accumulatedVertexSize / sizeof(BasicVertex); // It's not byte offset, Just Index Offset
    drawCommand.firstInstance = meshIndex++;

    currentBatch->m_drawIndexedCommands.push_back(drawCommand);

    // 누적된 데이터에 현재 메쉬 추가
    accumulatedVertices.insert(accumulatedVertices.end(), mesh.vertices.begin(), mesh.vertices.end());
    accumulatedIndices.insert(accumulatedIndices.end(), mesh.indices.begin(), mesh.indices.end());

    accumulatedVertexSize += vertexDataSize;
    accumulatedIndexSize += indexDataSize;

    // 누적된 크기가 3MB를 초과하면 새로운 mini-batch 생성 또는 현재까지 아무것도 생성하지 않았을 경우, 마지막에 생성
    if (accumulatedVertexSize + accumulatedIndexSize > MAX_BATCH_SIZE || flag)
    {
        // 새로운 mini-batch 생성
        MiniBatch miniBatch;
        manager.CreateVertexBuffer(accumulatedVertexSize, &miniBatch.m_vertexBufferMemory, &miniBatch.m_vertexBuffer, accumulatedVertices.data());
        manager.CreateIndexBuffer(accumulatedIndexSize, &miniBatch.m_indexBufferMemory, &miniBatch.m_indexBuffer, accumulatedIndices.data());

        miniBatch.m_currentVertexOffset = accumulatedVertexSize;
        miniBatch.m_currentIndexOffset = accumulatedIndexSize;
        miniBatch.m_currentBatchSize = accumulatedVertexSize + accumulatedIndexSize;

        miniBatches.push_back(miniBatch);

        std::cout << "New mini-batch created with size: " << miniBatch.m_currentBatchSize << " bytes." << std::endl;

        // 누적된 데이터 초기화
        accumulatedVertices.clear();
        accumulatedIndices.clear();
        accumulatedVertexSize = 0;
        accumulatedIndexSize = 0;
    }
    else {
        std::cout << "Accumulating mesh data. Current accumulated size: "
            << accumulatedVertexSize + accumulatedIndexSize << " bytes." << std::endl;
    }
}
```

```cpp
static void FlushMiniBatch(std::vector<MiniBatch>& miniBatches, VkUtils::ResourceManager& manager)  {
    if (accumulatedVertexSize == 0 && accumulatedIndexSize == 0) return;

    MiniBatch* currentBatch = nullptr;
    if (!miniBatches.empty()) {
        currentBatch = &miniBatches.back();
    }
    else {
        // mini-batch가 없으면 새로운 batch 생성
        MiniBatch newBatch;
        miniBatches.push_back(newBatch);
        currentBatch = &miniBatches.back();
    }
    manager.CreateVertexBuffer(accumulatedVertexSize, &currentBatch->m_vertexBufferMemory, &currentBatch->m_vertexBuffer, accumulatedVertices.data());
    manager.CreateIndexBuffer(accumulatedIndexSize, &currentBatch->m_indexBufferMemory, &currentBatch->m_indexBuffer, accumulatedIndices.data());

    currentBatch->m_currentVertexOffset = accumulatedVertexSize;
    currentBatch->m_currentIndexOffset = accumulatedIndexSize;
    currentBatch->m_currentBatchSize = accumulatedVertexSize + accumulatedIndexSize;

    std::cout << "Flushed mini-batch with size: " << currentBatch->m_currentBatchSize << " bytes." << std::endl;

    // 누적된 데이터 초기화
    accumulatedVertices.clear();
    accumulatedIndices.clear();
    accumulatedVertexSize = 0;
    accumulatedIndexSize = 0;
}
```