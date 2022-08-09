---
layout: post
title:  "UE4에서 Mesh의 LOD를 계산하고 렌더링에 반영하는 과정"
date:   2022-07-24
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---         
          
          
![0](https://user-images.githubusercontent.com/33873804/180654464-36c39017-f103-42b2-9084-92b63e06cccc.png)          
![1](https://user-images.githubusercontent.com/33873804/180654457-9e3bd540-f196-4303-a4a9-1aa4213ea9ff.png)          
![2](https://user-images.githubusercontent.com/33873804/180654460-0e408f88-e0ab-48f3-9923-b47f56da0cb4.png)          
![3](https://user-images.githubusercontent.com/33873804/180654461-78825a07-a437-430b-97a4-7579729f7cb6.png)          
![4](https://user-images.githubusercontent.com/33873804/180654462-68b824f2-9e5c-495a-b92f-09408d15eab9.png)          
                        
                        
                        
-----------------------------------
                        
**SkinnedMeshComponent의 경우**      
                        
**캐릭터(SkinnedMeshComponent)의 LOD Index를 결정하는 지점**           
                        
USkinnedMeshComponent::TickComponent(게임 스레드) -> USkinnedMeshComponent::UpdateLODStatus -> USkinnedMeshComponent::UpdateLODStatus_Internal(게임 스레드) -> USkinnedMeshComponent::UpdateLODStatus_Internal 함수에서 FSkeletalMeshObject::MinDesiredLODLevel 변수를 읽어서 최적의 LOD 레벨을 결정함. 이 FSkeletalMeshObject::MinDesiredLODLevel 변수는 몇 프레임 전 렌더스레드에서 Write 됨. ( 아래 참고 )                        
                        
                        
**FSkeletalMeshObject::MinDesiredLODLevel 변수 계산**                        
                        
FSkeletalMeshSceneProxy::GetDynamicMeshElements(렌더스레드) -> FSkeletalMeshSceneProxy::GetMeshElementsConditionallySelectable -> FSkeletalMeshObject::UpdateMinDesiredLODLevel(렌더스레드, 여기서 현재 프레임에 적절한 LOD 레벨을 연산해서 WorkingMinDesiredLODLevel에 저장함) -> 다음 프레임에 WorkingMinDesiredLODLevel의 값이 FSkeletalMeshObject::MinDesiredLODLevel로 복사됨 -> FSkeletalMeshObject::MinDesiredLODLevel는 렌더 스레드에서 쓰여지고, 게임 스레드에서 읽혀짐                        
                        
                         
                        
**캐릭터(SkinnedMeshComponent)의 LOD를 렌더스레드에 넘기는 지점**                        
                        
캐릭터의 위치 변경 등의 이유로 Render State Dirty -> USkinnedMeshComponent::CreateRenderState_Concurrent(게임 스레드) -> (USkinnedMeshComponent::GetPredictedLODLevel( 위의 USkinnedMeshComponent::UpdateLODStatus_Internal에서 연산한 렌더링에 사용 될 LOD 레벨을 반환함 ) ->) FSkeletalMeshObjectGPUSkin::Update -> FDynamicSkelMeshObjectDataGPUSkin::InitDynamicSkelMeshObjectDataGPUSkin(게임스레드, FDynamicSkelMeshObjectDataGPUSkin::LODIndex에 렌더링에 사용 될 LOD Level을 저장, 이 LODIndex 기준으로 GPU에서 Skinning 연산함) -> FSkeletalMeshObjectGPUSkin::UpdateDynamicData_RenderThread(렌더스레드)                        
                        
                         
                        
**렌더스레드에서 게임 스레드로부터 전달 받은 LOD Index를 가지고 MeshBatch를 생성하는 과정**                        
                        
InitViews-> FSceneRenderer::GatherDynamicMeshElements(렌더스레드) -> FSkeletalMeshSceneProxy::GetDynamicMeshElements -> FSkeletalMeshSceneProxy::GetMeshElementsConditionallySelectable(FDynamicSkelMeshObjectDataGPUSkin::LODIndex를 가지고, MeshBatch를 만드는데 필요한 LOD Section Data를 정함) -> FSkeletalMeshSceneProxy::GetDynamicElementsSection(렌더스레드)                        
                        
                 
-------------------------                 
**StaticMesh ( Static Mesh Component의 경우 대부분 이 경우에 해당함 )의 LOD 적용 경로**                 
                 
기본적으로 각 LOD별 렌더링에 필요한 데이터는 PrimitiveSceneInfo에서 LOD 레벨별로 모두 가지고 있다. ( FStaticMeshRenderData::LODResources )                 
                 
InitViews ->  FSceneRenderer::ComputeViewVisibility -> ComputeAndMarkRelevanceForViewParallel -> FRelevancePacket::MarkRelevant ( ComputeLODForMeshes로 적절한 LOD Level 선정 후 RelevantStaticPrimitives 순회하면서 각 Primitive의 StaticMeshRelevances 순회하여서 선정된 LOD Level에 맞는 FStaticMeshBatchRelevance 발견하면 ) -> FDrawCommandRelevancePacket::AddCommandsForMesh로 최종적으로 캐싱해둔 MeshDrawCommnads를 추가함.                 
                 
                  
                 
**Dynamic Mesh ( SkinnedMeshComponent의 경우 대부분 경우 여기에 해당 )의 LOD 적용 경로**                 
                 
기본적으로 각 LOD별 렌더링에 필요한 데이터는 PrimitiveSceneInfo에서 LOD 레벨별로 모두 가지고 있다. ( FSkeletalMeshRenderData::LODRenderData )                 
                 
SkinnedMeshComponent::TickComponenet에서 적절한 LOD Level 결정 후 렌더스레드에 넘겨줌.                 
                 
StaticMesh와 다르게 SkinnedMeshComponent의 경우 Skinning이 필요하다. SkinnedMeshComponent로부터 넘겨 받은 LOD Level에 맞는 Mesh 데이터를 Skinning을 수행 ( FSkeletalMeshObjectGPUSkin::Update -> ( 게임스레드로부터 넘겨 받은 적절한 LOD레벨에 맞는 데이터를 넘김 ) FDynamicSkelMeshObjectDataGPUSkin::InitDynamicSkelMeshObjectDataGPUSkin의 경로로 GPU Skinning을 위한 셋팅들 수행 )                 
                 
DynamicMesh는 MeshDrawCommands를 그때 그때 만드니 FSkeletalMeshSceneProxy::GetDynamicMeshElements에서 현재 LOD 레벨에 맞는 MeshBatch를 생성해줌.                 