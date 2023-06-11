---
layout: post
title:  "언리얼 엔진4 HISM(Hierarchical Instanced Static Mesh), UHierarchicalInstancedStaticMeshComponent, Foliage 코드 분석 ( 작성 중 )"
date:   2026-08-09
tags: [UE]
---         
                
![0_UInstancedStaticMeshComponent_CreateSceneProxy](https://user-images.githubusercontent.com/33873804/183704194-ed1bcee8-bf0b-4939-a8a5-aaefaab3ad37.png)                  
![01_UInstancedStaticMeshComponent_FlushInstanceUpdateCommands](https://user-images.githubusercontent.com/33873804/183704205-3b1a1612-ff4b-4375-b3a4-0793e235fd39.png)                  
![1_UInstancedStaticMeshComponent_BuildRenderData](https://user-images.githubusercontent.com/33873804/183704206-8801008b-ce78-4b92-8a31-83b437f3252f.png)                  
![2_FStaticMeshInstanceData_AllocateInstance](https://user-images.githubusercontent.com/33873804/183704208-1ccbe210-14d7-41d1-9258-2dfa963114c5.png)                  
![3_FInstancedStaticMeshRenderData](https://user-images.githubusercontent.com/33873804/183704211-738ce573-09d1-44bd-b269-4d7666446acb.png)                  
![4_FStaticMeshInstanceBuffer_InitRHI](https://user-images.githubusercontent.com/33873804/183704212-bd6f6fec-d75a-4482-9b92-5ce90487d212.png)                  
![5_FInstancedStaticMeshRenderData_InitVertexFactories](https://user-images.githubusercontent.com/33873804/183704214-45637e5c-f87c-4577-a50d-d6df08bf297f.png)                  
![6_FInstancedStaticMeshVertexFactory](https://user-images.githubusercontent.com/33873804/183704216-b32d3503-03b5-45bd-aac7-2381cbc2ba6c.png)                  
![7_FInstancedStaticMeshVertexFactory_InitRHI](https://user-images.githubusercontent.com/33873804/183704218-012bf898-293c-4220-b9e8-529f2c003729.png)                  
![9_FStaticMeshInstanceBuffer_BindInstanceVertexBuffer](https://user-images.githubusercontent.com/33873804/183704221-f22e51ee-1296-4e81-b35e-afd8cfc3dffc.png)                  
![19_FInstancedStaticMeshSceneProxy_GetViewRelevance](https://user-images.githubusercontent.com/33873804/183704225-2247ddb8-27cc-44ec-ab66-da10e202e7a5.png)                  
![20_FInstancedStaticMeshSceneProxy_GetMeshElement](https://user-images.githubusercontent.com/33873804/183704226-b1773fb3-a53a-4761-a567-e2613b7f7968.png)                  
![21_FInstancedStaticMeshSceneProxy_SetupInstancedMeshBatch](https://user-images.githubusercontent.com/33873804/183704227-72397742-a7fb-494e-a924-2d6eb4ee7f4f.png)                  
![28_UHierarchicalInstancedStaticMeshComponent_CreateSceneProxy](https://user-images.githubusercontent.com/33873804/183704230-e074e8f6-7762-4d60-88ce-2b51359cd061.png)                  
![29_FHierarchicalStaticMeshSceneProxy](https://user-images.githubusercontent.com/33873804/183704233-b14a139a-c9ff-4583-b52b-518ae8161298.png)                  
![30_UHierarchicalInstancedStaticMeshComponent_BuildTree](https://user-images.githubusercontent.com/33873804/183704235-ed9f6891-ff14-4663-b21e-9617b357a4df.png)                  
![31_FHierarchicalStaticMeshSceneProxy_GetDynamicMeshElements_1](https://user-images.githubusercontent.com/33873804/183704236-f635cf44-fded-4cca-9dd7-dee370d3031d.png)                  
![32_FHierarchicalStaticMeshSceneProxy_GetDynamicMeshElements_2](https://user-images.githubusercontent.com/33873804/183704240-955f6568-52df-414d-800b-db1307807de1.png)                  
![33_FHierarchicalStaticMeshSceneProxy_GetDynamicMeshElements_3](https://user-images.githubusercontent.com/33873804/183704243-cac5f961-4b62-42e3-b755-18a03d0fb2cb.png)                  
