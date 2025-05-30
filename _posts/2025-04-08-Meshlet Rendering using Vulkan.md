---
layout: post
title: "Meshlet Rendering using Vulkan"
description: "Understanding Mesh Shader"
date: 2025-04-08
tags: Graphics Meshlet MeshShader
comments: true
use_math: true
---

## 1. What is Mesh Shader?
![Image](https://private-user-images.githubusercontent.com/165359228/448721042-7a65aa73-423b-4f6a-bca4-94ac5c2dfec6.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDg1MDI0ODEsIm5iZiI6MTc0ODUwMjE4MSwicGF0aCI6Ii8xNjUzNTkyMjgvNDQ4NzIxMDQyLTdhNjVhYTczLTQyM2ItNGY2YS1iY2E0LTk0YWM1YzJkZmVjNi5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjUwNTI5JTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI1MDUyOVQwNzAzMDFaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT1kNjJlYzZkZGNkMTRhY2NmMGQ5YjkxZThiOWRmMTA5NmI4ZDFkMzk2MTQwNjlmZDI1NTg3NzBmOGFmZjc4Njk2JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.ASFMz7tkn8Bfvf1MHQMkvl-ZrKRgoqd8FOeS1cmUCkw)
<div align="center">
    <span style="color: #cccccc; font-size: 0.85em;">
        그림1. 전통 파이프라인과 Meshlet 파이프라인의 차이점(https://developer.nvidia.com/blog/introduction-turing-mesh-shaders/)
    </span>
</div>

기존의 그래픽 파이프라인은 정점 셰이더(Vertex Shader), 테셀레이션 셰이더(Tessellation Shader), 지오메트리 셰이더(Geometry Shader) 등 여러 단계로 구성되어 있다. 이러한 전통적인 파이프라인을 대체하는 새로운 개념으로 Mesh Shader를 도입하였습니다. Mesh Shader는 **Task Shader**와 **Mesh Shader**의 두 단계로 구성된다. 이 두 단계를 통해 기존의 파이프라인이 담당하였던 Vertex Shader, Tessllation Shader, Geometry Shader 부분을 대체하는 것이다.

- Task Shader: 렌더링할 작업을 정의하고, 각 작업에 대한 데이터를 준비한다. 해당 Shader에서 Culling과 같은 사전작업을 수행한다.
- Mesh Shader: Task Shader에서 전달된 데이터를 기반으로 실제 기하 정보를 생성한다.

이러한 구조는 GPU의 병렬 처리 능력을 극대화하며, 복잡한 기하 구조를 효율적으로 처리할 수 있게 한다.

## 2. What is Meshlet?
Meshlet은 큰 메쉬를 작은 단위로 분할한 것이다다. 각 Meshlet은 일정 수의 정점과 삼각형으로 구성되며, GPU에서 효율적으로 처리할 수 있도록 만든 블록이라고 보면 된다.

기존의 큰 메쉬를 한 번에 GPU로 보내 처리하면, GPU의 병렬 처리 자원을 충분히 활용하지 못하거나 효율이 떨어지는 문제가 발생할 수 있다. 반면 Meshlet은 이러한 병목 현상을 줄이고 렌더링 효율성을 극대화하기 위해 고안되었다.

<div style="display: flex; justify-content: center;">
    <img src="https://private-user-images.githubusercontent.com/165359228/449110223-36b1162c-8937-47ad-a7a2-60a8f620ba26.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDg1ODQzNjgsIm5iZiI6MTc0ODU4NDA2OCwicGF0aCI6Ii8xNjUzNTkyMjgvNDQ5MTEwMjIzLTM2YjExNjJjLTg5MzctNDdhZC1hN2EyLTYwYThmNjIwYmEyNi5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjUwNTMwJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI1MDUzMFQwNTQ3NDhaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT0zYzU3MTVmMjA1NTMyNzEwNTJmNTJhN2JhZWVlZjQwNTRkOTFiZDNmM2I2MTUyZTBkYzdkMDZhNmRlZGJmMTg5JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.lalEsJM4Kx_4G0SHnU101G4rDASpOHLEJff85nIqzm8" alt="이미지1 설명" width="400"/>
    <img src="https://private-user-images.githubusercontent.com/165359228/449110542-cda606dc-69dc-42d0-85a7-507db80c9ea6.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDg1ODQ0MTcsIm5iZiI6MTc0ODU4NDExNywicGF0aCI6Ii8xNjUzNTkyMjgvNDQ5MTEwNTQyLWNkYTYwNmRjLTY5ZGMtNDJkMC04NWE3LTUwN2RiODBjOWVhNi5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjUwNTMwJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI1MDUzMFQwNTQ4MzdaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT1kNTk4ZDNmMWM2YTEyZDgzOGYyYTUxYjFkZjVlZmRjZmJjYjYwYzYzM2Q0NWNhOTQxNDM2MDgwZjE2ZDNkNzViJlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.EBPOQp7VnAGT7ljBhk-IXfEm249D_pc47FPPuMHS3Ho" alt="이미지2 설명" width="400"/>
</div>
<div align="center">
    <span style="color: #cccccc; font-size: 0.85em;">
        그림2. Meshlet의 원리(https://gpuopen.com/learn/mesh_shaders/mesh_shaders-meshlet_compression/)
    </span>
</div>

예를 들어, 한 개의 Meshlet당 64개의 정점과 126개의 삼각형으로 구성될 수 있다. 이렇게 구성하면, 각 Meshlet은 독립적으로 처리될 수 있어 병렬 처리가 용이해지고 126개의 삼각형 씩 병렬로 돌리는 것이므로 성능도 향상된다. 또한, Culling 과정도 Meshlet을 이용하기 때문에 렌더링 효율이 향상된다.

### 2.1. Meshlet 생성 과정
오픈소스 라이브러리인 [MeshOptimizer](https://github.com/zeux/meshoptimizer)를 활용하면 매우 쉽고 효율적으로 Meshlet을 생성할 수 있다.

```cpp
meshlets = meshopt_buildMeshlets(
    vertices,          // 메쉬의 정점 배열
    indices,           // 메쉬의 인덱스 배열
    max_vertices=64,   // Meshlet 당 최대 정점 수
    max_triangles=126  // Meshlet 당 최대 삼각형 수
)
```

위와 같이, 코드를 작성할 수 있는데, 한 Mesh를 64개의 정점 및 126개의 삼각형을 초과하지 않는 작은 블록으로 나눈다는 의미이다.

```cpp
const size_t maxMeshlets = meshopt_buildMeshletsBound(mesh.indices.size(), kMaxVertices, kMaxTriangles);
mesh.m_meshlets.resize(maxMeshlets);
mesh.m_meshletVertices.resize(maxMeshlets * kMaxVertices);
mesh.m_meshletTriangles.resize(maxMeshlets * kMaxTriangles * 3);

for (BasicVertex& v : mesh.vertices) {
  mesh.m_positions.push_back(v.pos);
}

size_t meshletCount = meshopt_buildMeshlets(
    mesh.m_meshlets.data(), mesh.m_meshletVertices.data(), mesh.m_meshletTriangles.data(),
    reinterpret_cast<const uint32_t*>(mesh.indices.data()), mesh.indices.size(), &mesh.ray_vertices.data()->pos.x,
    mesh.ray_vertices.size(), sizeof(RayTracingVertex), kMaxVertices, kMaxTriangles, kConeWeight);

auto& last = mesh.m_meshlets[meshletCount - 1];
mesh.m_meshletVertices.resize(last.vertex_offset + last.vertex_count);
mesh.m_meshletTriangles.resize(last.triangle_offset + ((last.triangle_count * 3 + 3) & ~3));
mesh.m_meshlets.resize(meshletCount);

/*
 * Repack triangles from 3 consecutive bytes to 4byte uint32_t to make it easier to unpack on GPU.
 */
std::vector<uint32_t> meshletTrianglesU32;
for (auto& m : mesh.m_meshlets) {
  // Save triangle offset for current meshlet
  uint32_t triangleOffset = static_cast<uint32_t>(meshletTrianglesU32.size());

  // Repack to uint32_t
  for (uint32_t i = 0; i < m.triangle_count; ++i) {
    uint32_t i0 = 3 * i + 0 + m.triangle_offset;
    uint32_t i1 = 3 * i + 1 + m.triangle_offset;
    uint32_t i2 = 3 * i + 2 + m.triangle_offset;

    uint8_t vIdx0 = mesh.m_meshletTriangles[i0];
    uint8_t vIdx1 = mesh.m_meshletTriangles[i1];
    uint8_t vIdx2 = mesh.m_meshletTriangles[i2];
    uint32_t packed = ((static_cast<uint32_t>(vIdx0) & 0xFF) << 0) | ((static_cast<uint32_t>(vIdx1) & 0xFF) << 8) |
                      ((static_cast<uint32_t>(vIdx2) & 0xFF) << 16);
    meshletTrianglesU32.push_back(packed);
  }

  // Update triangle offset for current meshlet
  m.triangle_offset = triangleOffset;
}
```

코드를 더 세부적으로 본다면 위와 같다. 이렇게 라이브러리를 통해 만든 meshlet 데이터를 이용해서 GPU Buffer를 생성하는 것이다. 생성하는 버퍼의 종류는 다음과 같다.

1. **posBuffer**: Mesh를 이루는 각 정점의 위치 정보를 저장한다.
2. **meshletBuffer**: 각 Meshlet마다 어떤 정점 및 삼각형들이 포함되어 있는지에 대한 정보를 담는다. 예를 들어, vertex 범위와 index 정보가 있다.
3. **meshletVerticesBuffer**: Meshlet 내에서 사용하는 vertex를 저장한다.
4. **meshletTrianglesBuffer**: 삼각형을 구성하는 정점 인덱스를 저장한다. Meshlet 별 삼각형 구성 정보 저장으로 GPU의 삼각형 렌더링을 위한 데이터 구조이다.
5. **meshletBoundBuffer**: 경계(Bounding Sphere 또는 Bounding Box) 정보를 저장한다.

### 2.2. Task Shader
Task Shader는 Mesh Shader 단계 전에 실행되며, Mesh Shader가 처리할 Meshlet(작은 메쉬 단위)의 작업 단위를 결정하여 GPU가 효율적으로 Meshlet 데이터를 처리할 수 있게 도와준다.

**Payload 구조체**는 Task Shader에서 Mesh Shader로 데이터를 넘겨주는 역할을 한다. 여기서 **meshletIndices**는 Mesh Shader가 처리해야 할 Meshlet의 인덱스를 저장하는 배열이다. **taskPayloadSharedEXT**를 통해 워크그룹 내 공유 메모리에 이 Payload 데이터를 저장한다.

각 쓰레드가 자신이 처리할 Meshlet 인덱스(dtid)를 Payload 배열의 자신의 위치(gtid)에 저장하고, 이를 통해 각 Mesh Shader 쓰레드가 정확히 어떤 Meshlet을 처리해야 하는지 정보를 얻는 것이다.

```GLSL
EmitMeshTasksEXT(TASK_WORKGROUP_SIZE, 1, 1);
```

이 함수는 Task Shader에서 Mesh Shader를 호출하는 명령어로, GPU에게 몇 개의 Mesh Shader 워크그룹을 실행할지 알려준다.

```GLSL
#define TASK_WORKGROUP_SIZE 32
#define MESH_WORKGROUP_SIZE 128

layout(local_size_x = TASK_WORKGROUP_SIZE, local_size_y = 1, local_size_z = 1) in;


struct Payload {
	uint meshletIndices[TASK_WORKGROUP_SIZE];
};

taskPayloadSharedEXT Payload s_Payload;

void main()
{
	uint gtid = gl_LocalInvocationID.x;
	uint gid = gl_WorkGroupID.x;
	uint dtid = gl_WorkGroupID.x * gl_WorkGroupSize.x + gtid;

	s_Payload.meshletIndices[gtid] = dtid;

	EmitMeshTasksEXT(TASK_WORKGROUP_SIZE, 1, 1);
}
```

### 2.3. Mesh Shader
Mesh Shader는 GPU가 효율적으로 데이터에 접근할 수 있도록, 미리 정의된 구조체와 버퍼를 사용합니다. 예를 들어 Meshlet 정보를 나타내는 구조체(Meshlet)는 다음과 같은 내용을 포함한다.

```glsl
struct Meshlet {
	uint vertexOffset;
	uint triangleOffset;
	uint vertexCount;
	uint triangleCount;
};
```

이 Meshlet 구조체는 GPU에서 읽기 전용 버퍼(SSBO)에 저장되어 있으며, Mesh Shader가 렌더링 과정에서 접근하여 활용하게 된다.

Mesh Shader는 앞서 Task Shader 단계에서 전달된 Payload 데이터를 공유 메모리 형태로 받는다. Payload는 Mesh Shader가 처리해야 할 Meshlet의 인덱스 목록을 담고 있으며, 이 정보는 GPU 쓰레드가 각자 어떤 Meshlet을 처리해야 하는지 명확하게 결정한다.

```GLSL
struct Payload {
	uint meshletIndices[TASK_WORKGROUP_SIZE];
};
taskPayloadSharedEXT Payload s_Payload;
```

GPU는 각 쓰레드마다 Meshlet 내부의 삼각형 정보를 가져와서, 이를 Mesh Shader 출력 데이터로 설정한다다. Meshlet에서 삼각형 인덱스는 효율성을 위해 압축된 형태로 저장되는데, 비트 연산을 통해 삼각형을 구성하는 정점 인덱스를 빠르게 해독한다.

```glsl
if(gtid < m.triangleCount) {
    uint packed = meshletTriangleIndices[m.triangleOffset + gtid];
    uint vIdx0  = (packed >>  0) & 0xFF;
    uint vIdx1  = (packed >>  8) & 0xFF;
    uint vIdx2  = (packed >> 16) & 0xFF;
    gl_PrimitiveTriangleIndicesEXT[gtid] = uvec3(vIdx0, vIdx1, vIdx2);
}
```

Mesh Shader는 메쉬 데이터를 GPU에서 직접 렌더링하기 위해 pos를 변환하고, 다양한 부가 정보를 전송한다. 특히 모델, 뷰, 프로젝션 변환 행렬을 곱하여 최종 화면 좌표를 생성한다.

```glsl
if(gtid < m.vertexCount) {
	uint vertexIndex = gtid + m.vertexOffset;
	vertexIndex = meshletVertexIndices[vertexIndex];

	mat4 model = nonuniformEXT(ssbo_Model.transform[u_ShaderSetting.batchIdx].currentModel);
	gl_MeshVerticesEXT[gtid].gl_Position = u_Camera.projection * u_Camera.view * model * vec4(meshletVertices[vertexIndex].position.xyz, 1.0);

	
	vec3 color = vec3(float(gid & 1u), float(gid & 3u) / 4.0, float(gid & 7u) / 8.0);
	meshOuptuts[gtid].pos = model * vec4(meshletVertices[vertexIndex].position.xyz, 1.0);
	meshOuptuts[gtid].color = color;
	meshOuptuts[gtid].normal = meshletVertices[vertexIndex].normal;
	meshOuptuts[gtid].tex = meshletVertices[vertexIndex].tex;
}
```

## 3. 최종 렌더링 이미지
![image](https://private-user-images.githubusercontent.com/165359228/449116339-62da5810-3ba4-4c23-87ca-d8b5bfc8aa80.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTUiLCJleHAiOjE3NDg1ODU3MjIsIm5iZiI6MTc0ODU4NTQyMiwicGF0aCI6Ii8xNjUzNTkyMjgvNDQ5MTE2MzM5LTYyZGE1ODEwLTNiYTQtNGMyMy04N2NhLWQ4YjViZmM4YWE4MC5wbmc_WC1BbXotQWxnb3JpdGhtPUFXUzQtSE1BQy1TSEEyNTYmWC1BbXotQ3JlZGVudGlhbD1BS0lBVkNPRFlMU0E1M1BRSzRaQSUyRjIwMjUwNTMwJTJGdXMtZWFzdC0xJTJGczMlMkZhd3M0X3JlcXVlc3QmWC1BbXotRGF0ZT0yMDI1MDUzMFQwNjEwMjJaJlgtQW16LUV4cGlyZXM9MzAwJlgtQW16LVNpZ25hdHVyZT1mZjJhNDE1OWMyMzgxNjRkOWE0YjJhNTYxYmNiZTcyMmIxMDA5ZDU0ODRhMTBjZTg4ZjUxMDNjNjAyNmNjNGU2JlgtQW16LVNpZ25lZEhlYWRlcnM9aG9zdCJ9.ANGI0rrGAGo7L6DvxEcBkVn8PTXJDfUYe-XYXeNPSIE)
<div align="center">
    <span style="color: #cccccc; font-size: 0.85em;">
        그림3. Meshlet Rendering(직접 구현)
    </span>
</div>