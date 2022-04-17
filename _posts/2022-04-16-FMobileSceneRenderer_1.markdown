---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 2"
date:   2022-04-16
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---

중요도가 떨어지는 코드는        
```

...
...
...

```
으로 처리해두었으니 확인하고 싶은 경우에는 UE4 소스코드를 확인해주세요.           

------------------------         

**FMobileSceneRenderer**는 언리얼 엔진4의 모바일용 그래픽스 파이프라인이다.                
이 글은 **FMobileSceneRenderer::Render** 함수를 위주로 FMobileSceneRenderer의 소스 코드를 분석할 것이다.      
          
UE4 4.27 버전을 기준으로 작성하였다.      
4.27 버전이 언리얼 엔진 4세대의 마지막 Release 버전이다.             
          
----------------------------              
             
이번 챕터에는 FMobileSceneRenderer::Render 함수의 첫번째 중요 함수인 **FScene::UpdateAllPrimitiveSceneInfos**에 대해 분석해볼 것이다.        
              
함께 보면 좋은 참고 자료 : [UE5 MeshDrawCommand (1/2)](https://scahp.tistory.com/74?category=848072), [UE5 MeshDrawCommand (2/2)](https://scahp.tistory.com/75?category=848)             
                   
------------------------------         

```cpp
void FScene::UpdateAllPrimitiveSceneInfos(FRHICommandListImmediate& RHICmdList, bool bAsyncCreateLPIs)
{
	SCOPED_NAMED_EVENT(FScene_UpdateAllPrimitiveSceneInfos, FColor::Orange);
	SCOPE_CYCLE_COUNTER(STAT_UpdateScenePrimitiveRenderThreadTime);

	// ⭐ 
	// 이 함수는 렌더 스레드에서 도는 함수이다. 
	check(IsInRenderingThread()); 
	// ⭐


	// ⭐⭐⭐⭐⭐⭐⭐
	// FPrimitiveSceneInfo는 하나의 UPrimitiveComponent에 대한 내부 상대 정보를 담은 클래스이다.
	// 이는 엔진 모듈 내에서 FPrimitiveSceneProxy와 1대 1로 매칭된다.	
	// FPrimitiveSceneInfo와 FPrimitiveSceneProxy는 게임 스레드에서 생성되어 렌더스레드에서 제거된다.
	// 이 두 데이터가 생성되는 것은 FScene::AddPrimitive 함수 ( 게임 스레드에서 수행됨 )에서 볼 수 있다.
	// 더 자세한 분석은 아래 링크를 타고 들어가면 볼 수 있다.
	// https://scahp.tistory.com/74?category=848072
	// https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/
	// ⭐⭐⭐⭐⭐⭐⭐


	// ⭐
	// 삭제할 Privmitive Scene Info 리스트를 로컬 변수에 저장한다. 
	// 이는 바로 밑에 보이는 Sorting 작업을 위함이다. Sorting은 SceneInfo와 매칭되는 FPrimitiveSceneProxy의 Hash값을 기준으로 Sorting한다.
	// -> 동일한 FPrimitiveSceneProxy TypeHash를 가진 SceneInfo들끼리 뭉치게된다.
	//
	// 게임 스레드에서 Primitive를 제거하면 이후 렌더스레드에서 해당 Primitive가 RemovedPrimitiveSceneInfos 추가된다.
	// 게임 스레드에서 Primitive를 제거했다고 바로 Primitive를 바로 삭제해버리면 뒤따라오는 렌더스레드는 Primitive가 삭제되기 전 프레임을 렌더링해야하는데 문제가 생길 수 있다.
	// 그렇기 때문에 게임 스레드에서 Primitive를 삭제하더라도 곧바로 삭제가 되는 것이 아니라, 뒤따라오는 렌더스레드에서 실제 삭제를 수행한다.
	//
	// UActorComponent::ExecuteUnregisterEvents ( 게임 스레드 )  -> UPrimitiveComponent::DestroyRenderState_Concurrent ( 게임 스레드 ) -> FScene::RemovePrimitive ( 게임 스레드 ) -> 
	// RemovePrimitiveSceneInfo_RenderThread ( 렌더 스레드 )를 통해 제거된 Primitive가 RemovedPrimitiveSceneInfos에 추가된다. 
	TArray<FPrimitiveSceneInfo*> RemovedLocalPrimitiveSceneInfos(RemovedPrimitiveSceneInfos.Array());
	// ⭐

	// ⭐ 
	// FPrimitiveArraySortKey : return A.Proxy->GetTypeHash() < B.Proxy->GetTypeHash(); 
	RemovedLocalPrimitiveSceneInfos.Sort(FPrimitiveArraySortKey()); 
	// ⭐

	// ⭐ 
	// 새로 추가된 Primitive Scene Info 
	// Primtive를 삭제할때도 마찬가지이다.
	// 게임 스레드에서 Primitive를 추가했더라도 곧바로 렌더스레드에 반영되는 것이 아니라, 게임 스레드가 Primitive를 추가한 프레임에 맞추어서 렌더스레드에서 추가된다.
	TArray<FPrimitiveSceneInfo*> AddedLocalPrimitiveSceneInfos(AddedPrimitiveSceneInfos.Array());
	AddedLocalPrimitiveSceneInfos.Sort(FPrimitiveArraySortKey());
	// ⭐
	
	// ⭐ 
	// 삭제된 SceneInfo를 저장한다.
	// 밑에서 Transform 데이터를 업데이트하거나 할 때 이미 삭제된 SceneInfo와 연관된 ( 불필요한 ) 데이터를 업데이트하는 것을 방지하기 위함이다.
	TSet<FPrimitiveSceneInfo*> DeletedSceneInfos;
	DeletedSceneInfos.Reserve(RemovedLocalPrimitiveSceneInfos.Num());
	// ⭐

	if (!!GAsyncCreateLightPrimitiveInteractions && !AsyncCreateLightPrimitiveInteractionsTask)
	{
		// ⭐
		// Primtive에 영향을 주는 Light를 찾아서 Interaction(FLightPrimitiveInteraction)을 만드는 동작을 ASync로 처리한다.
		// 자세히는 모르겠다.
		// FLightPrimitiveInteraction를 확인해보면 자세한 기능을 알 수 있을 것 같다.
		AsyncCreateLightPrimitiveInteractionsTask = new FAsyncTask<FAsyncCreateLightPrimitiveInteractionsTask>();
		// ⭐
	}

	if (AsyncCreateLightPrimitiveInteractionsTask)
	{
		if (!AsyncCreateLightPrimitiveInteractionsTask->IsDone())
		{
			AsyncCreateLightPrimitiveInteractionsTask->EnsureCompletion();
		}
		AsyncCreateLightPrimitiveInteractionsTask->GetTask().Init(this);
	}

	{
		SCOPED_NAMED_EVENT(FScene_RemovePrimitiveSceneInfos, FColor::Red);
		SCOPE_CYCLE_COUNTER(STAT_RemoveScenePrimitiveTime);
		for (FPrimitiveSceneInfo* PrimitiveSceneInfo : RemovedLocalPrimitiveSceneInfos)
		{
			// clear it up, parent is getting removed
			SceneLODHierarchy.UpdateNodeSceneInfo(PrimitiveSceneInfo->PrimitiveComponentId, nullptr);
		}

		while (RemovedLocalPrimitiveSceneInfos.Num())
		{
			int StartIndex = RemovedLocalPrimitiveSceneInfos.Num() - 1;
			SIZE_T InsertProxyHash = RemovedLocalPrimitiveSceneInfos[StartIndex]->Proxy->GetTypeHash();

			while (StartIndex > 0 && RemovedLocalPrimitiveSceneInfos[StartIndex - 1]->Proxy->GetTypeHash() == InsertProxyHash)
			{
				// ⭐
				// Type Hash ID로 정렬된 RemovedLocalPrimitiveSceneInfos 리스트에서 같은 ID들 중 가장 낮은 Index를 찾는다.
				// ex) 1, 1, 3, 2, 2, 4, 4, 4
				// 여기서 현재 Type ID 4 중 가장 앞의 4의 Index인 5을 찾는 과정이다.
				// ⭐
				StartIndex--;
			}


			// ⭐
			// TypeOffsetTable에서 StartIndex가 속한 Index를 BroadIndex에 저장
			// ⭐
			int BroadIndex = -1;
			//broad phase search for a matching type
			for (BroadIndex = TypeOffsetTable.Num() - 1; BroadIndex >= 0; BroadIndex--)
			{
				// ⭐
				// Proxy가 워낙 많다보니 만약 어떤 SceneInfo Type Hash를 가진 Proxy를 찾으려할때 PrimitiveSceneProxies를 순회하는데 시간이 많이 걸린다. 
				// 이를 위한 대안으로 TypeOffsetTable가 사용된다.
				// TypeOffsetTable는 정렬된 Proxy 리스트에서 다음 종류의 Proxy의 시작 Index를 저장한 리스트이다.
				// 그럼 자신이 원하는  SceneInfo Type Hash의 Proxy를 찾고자 할 때 TypeOffsetTable를 참고해가면서 같은 SceneInfo Type Hash를 가진 Element들은 널뛰면서 순회를 빠르게 할 수 있는 것이다.
				// 
				// ex)
				// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,2,1,1,1,7,4,8]
				// TypeOffsetTable[3 ( Type ID '6'의 시작 Index는 '3'이다. ), 8 ( Type ID '2'의 시작 Index는 '8'이다. ),12,15,16,17,18]
				//
				// ⭐
				
				if (TypeOffsetTable[BroadIndex].PrimitiveSceneProxyType == InsertProxyHash)
				{
					const int InsertionOffset = TypeOffsetTable[BroadIndex].Offset;
					const int PrevOffset = BroadIndex > 0 ? TypeOffsetTable[BroadIndex - 1].Offset : 0;
					for (int CheckIndex = StartIndex; CheckIndex < RemovedLocalPrimitiveSceneInfos.Num(); CheckIndex++)
					{
						// ⭐
						// 아래의 코드는 RemovedLocalPrimitiveSceneInfos에서 각 Type ID의 Element들의 SceneInfo::PrimitiveIndex가 TypeOffsetTable에 올바르게 들어있는지 확인(assert)하는 코드이다.
						// ex) 위의 예제에서 Type ID 6을 가진 Proxy들의 PrimitiveSceneProxies에서의 Index는 TypeOffsetTable의 Element들 중 3보다 크거나 같고, 8보다는 작아야한다. 
						int32 PrimitiveIndex = RemovedLocalPrimitiveSceneInfos[CheckIndex]->PackedIndex;
						checkfSlow(PrimitiveIndex >= PrevOffset && PrimitiveIndex < InsertionOffset, TEXT("PrimitiveIndex %d not in Bucket Range [%d, %d]"), PrimitiveIndex, PrevOffset, InsertionOffset);
						// ⭐
					}
					break;
				}
			}

			{
				SCOPED_NAMED_EVENT(FScene_SwapPrimitiveSceneInfos, FColor::Turquoise);

				for (int CheckIndex = StartIndex; CheckIndex < RemovedLocalPrimitiveSceneInfos.Num(); CheckIndex++)
				{
					// ⭐
					int SourceIndex = RemovedLocalPrimitiveSceneInfos[CheckIndex]->PackedIndex;
					// ⭐

					for (int TypeIndex = BroadIndex; TypeIndex < TypeOffsetTable.Num(); TypeIndex++)
					{
						FTypeOffsetTableEntry& NextEntry = TypeOffsetTable[TypeIndex];

						// ⭐
						// Swap하려는 SceneInfo의 Type Hash를 가진 연속된 Element들 중 마지막 Element의 Index를 DestIndex에 저장한다.
						int DestIndex = --NextEntry.Offset; //decrement and prepare swap 
						// ⭐

						// ⭐
						// TypeOffsetTable를 통해 널뛰면서 현재 삭제하려는 Index를 동일한  SceneInfo Type Hash 중 가장 마지막에 있는 Index와 Swap하면서
						// 삭제하려는 SceneInfo과 같은 TypeHash를 가진 SceneInfo들에 대한 렌더링 Data들을 리스트의 마지막으로 모두 밀어넣는다.
						// 
						// 이는 렌더링 관련 데이터의 리스트에서 같은 SceneInfo Type Hash를 가진 데이터들은 리스트 내에 연속되게 존재함을 유지하기 위함이다.
						// 나중에 드로우콜 Batch를 위해 쓰인다 ( ?, 확인 필요 )
						//
						// 밑에서 Pop하기 위함.
						// 
						// example swap chain of removing X 
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 ),2,2,2( DestIndex, 삭제하려는 데이터와 같은  ),1,1,1,7,4,8]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 ),1,1( DestIndex ),1,7,4,8]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,1,1,1,X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 ),7( DestIndex ),4,8]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,1,1,1,7,X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 ),4( DestIndex ),8]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,1,1,1,7,4,X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 ),8( DestIndex )]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,1,1,1,7,4,8, X( StartIndex, 삭제하려는 Rendering 데이터의 현재 위치 )]
						// ⭐

						if (DestIndex != SourceIndex)
						{
							checkfSlow(DestIndex > SourceIndex, TEXT("Corrupted Prefix Sum [%d, %d]"), DestIndex, SourceIndex);
							Primitives[DestIndex]->PackedIndex = SourceIndex;
							Primitives[SourceIndex]->PackedIndex = DestIndex;

							// ⭐
							// SourceIndex와 DestIndex의 각종 Rendering 관련 데이터들을 Swap한다.
							// ⭐

							TArraySwapElements(Primitives, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveTransforms, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveSceneProxies, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveBounds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveFlagsCompact, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVisibilityIds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveOcclusionFlags, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveComponentIds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVirtualTextureFlags, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVirtualTextureLod, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveOcclusionBounds, DestIndex, SourceIndex);
							TBitArraySwapElements(PrimitivesNeedingStaticMeshUpdate, DestIndex, SourceIndex);

							// ⭐⭐⭐⭐⭐⭐⭐
							// Swap을 할 때마다 위치가 바뀐 데이터들은 GPUScene에서 업데이트 해주어야한다.
							// GPUScene에 대해서는 더 조사가 필요..
							// UpdateGPUScene ( GPUScene.cpp ) 함수를 참고하세요.
							// ( 나중에 분석 필요 )
							// https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/
							AddPrimitiveToUpdateGPU(*this, SourceIndex);
							AddPrimitiveToUpdateGPU(*this, DestIndex);
							// ⭐⭐⭐⭐⭐⭐⭐

							SourceIndex = DestIndex;
						}
					}
				}
			}

			const int PreviousOffset = BroadIndex > 0 ? TypeOffsetTable[BroadIndex - 1].Offset : 0;
			const int CurrentOffset = TypeOffsetTable[BroadIndex].Offset;

			checkfSlow(PreviousOffset <= CurrentOffset, TEXT("Corrupted Bucket [%d, %d]"), PreviousOffset, CurrentOffset);
			if (CurrentOffset - PreviousOffset == 0)
			{
				// ⭐
				// 어떤 특정 종류의 SceneInfo가 완전히 다 없어진 경우, 해당 SceneInfo Type Hash에 대한 TypeOffsetTable Element를 제거해준다.
				// ⭐

				// remove empty OffsetTable entries e.g.
				// TypeOffsetTable[3,8,12,15,15,17,18]
				// TypeOffsetTable[3,8,12,15,17,18]
				TypeOffsetTable.RemoveAt(BroadIndex);
			}

			checkfSlow((TypeOffsetTable.Num() == 0 && Primitives.Num() == (RemovedLocalPrimitiveSceneInfos.Num() - StartIndex)) || TypeOffsetTable[TypeOffsetTable.Num() - 1].Offset == Primitives.Num() - (RemovedLocalPrimitiveSceneInfos.Num() - StartIndex), TEXT("Corrupted Tail Offset [%d, %d]"), TypeOffsetTable[TypeOffsetTable.Num() - 1].Offset, Primitives.Num() - (RemovedLocalPrimitiveSceneInfos.Num() - StartIndex));

			for (int CheckIndex = StartIndex; CheckIndex < RemovedLocalPrimitiveSceneInfos.Num(); CheckIndex++)
			{
				checkf(RemovedLocalPrimitiveSceneInfos[CheckIndex]->PackedIndex >= Primitives.Num() - RemovedLocalPrimitiveSceneInfos.Num(), TEXT("Removed item should be at the end"));
			}

			//Remove all items from the location of StartIndex to the end of the arrays.
			int RemoveCount = RemovedLocalPrimitiveSceneInfos.Num() - StartIndex;
			int SourceIndex = Primitives.Num() - RemoveCount;

			// ⭐
			// RemovedLocalPrimitiveSceneInfos[StartIndex]의 Type Hash와 동일한 Type Hash를 가지는 
			// Element들 ( 위에서 맨 위에서 Sorting을 했으므로 함께 몰려있다 )와 관련된 렌더링 관련 데이터 Element들은 아래 보이는 각 리스트에서 가장 마지막에 몰려있다.
			//
			// RemovedLocalPrimitiveSceneInfos[StartIndex]의 Type Hash와 동일한 Type Hash의 SceneInfo들의 렌더링 데이터들을 모두 리스트에서 삭제해준다. 
			// ⭐
			Primitives.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveTransforms.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveSceneProxies.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveBounds.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveFlagsCompact.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveVisibilityIds.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveOcclusionFlags.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveComponentIds.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveVirtualTextureFlags.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveVirtualTextureLod.RemoveAt(SourceIndex, RemoveCount);
			PrimitiveOcclusionBounds.RemoveAt(SourceIndex, RemoveCount);
			PrimitivesNeedingStaticMeshUpdate.RemoveAt(SourceIndex, RemoveCount);

			// ⭐
			// Primitives와 Primitives~ 리스트의 Element 개수가 같은지 확인해주는 디버깅용 함수.
			// ⭐
			CheckPrimitiveArrays();

			for (int RemoveIndex = StartIndex; RemoveIndex < RemovedLocalPrimitiveSceneInfos.Num(); RemoveIndex++)
			{
				// ⭐
				// 삭제하려는 SceneInfo와 관련된 렌더링 데이터는 바로 위에서 삭제했으니 이제는 SceneInfo를 삭제해준다.
				// 마찬가지로 같은 TypeHash를 가진 SceneInfo들을 한꺼번에 삭제해줌
				// ⭐

				FPrimitiveSceneInfo* PrimitiveSceneInfo = RemovedLocalPrimitiveSceneInfos[RemoveIndex];
				FScopeCycleCounter Context(PrimitiveSceneInfo->Proxy->GetStatId());
				int32 PrimitiveIndex = PrimitiveSceneInfo->PackedIndex;
				PrimitiveSceneInfo->PackedIndex = INDEX_NONE;

				if (ShouldPrimitiveOutputVelocity(PrimitiveSceneInfo->Proxy, GetShaderPlatform()))
				{
					// Remove primitive's motion blur information.
					VelocityData.RemoveFromScene(PrimitiveSceneInfo->PrimitiveComponentId);
				}

				// Unlink the primitive from its shadow parent.
				PrimitiveSceneInfo->UnlinkAttachmentGroup();

				// Unlink the LOD parent info if valid
				PrimitiveSceneInfo->UnlinkLODParentComponent();

				// Flush virtual textures touched by primitive
				PrimitiveSceneInfo->FlushRuntimeVirtualTexture();

				// ⭐
				// Remove the primitive from the scene.
				//
				// PrimitiveSceneInfo를 Scene에서 삭제한다.
				// ⭐
				PrimitiveSceneInfo->RemoveFromScene(true);

				AddPrimitiveToUpdateGPU(*this, PrimitiveIndex);

				DistanceFieldSceneData.RemovePrimitive(PrimitiveSceneInfo);

				// ⭐
				// 삭제된 Primitive에 대한 FPrimitiveSceneInfo는 임시로 저장해둔다.
				// 여기서 임시로 저장을 해두었다가 맨 마지막에 가서 관련 데이터들을 삭제해줄 것이다.
				DeletedSceneInfos.Add(PrimitiveSceneInfo);
				// ⭐
			}
			RemovedLocalPrimitiveSceneInfos.RemoveAt(StartIndex, RemovedLocalPrimitiveSceneInfos.Num() - StartIndex);
		}
	}

	// ⭐
	// 삭제하려는 SceneInfo에 대한 처리를 끝냈다!
	// ⭐

	{
		// ⭐
		// 추가된 FPrimitiveSceneInfo를 처리하는 과정은 비교적 간단하다.
		// ⭐

		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(AddPrimitiveSceneInfos);
		SCOPED_NAMED_EVENT(FScene_AddPrimitiveSceneInfos, FColor::Green);
		SCOPE_CYCLE_COUNTER(STAT_AddScenePrimitiveRenderThreadTime);
		if (AddedLocalPrimitiveSceneInfos.Num())
		{
			Primitives.Reserve(Primitives.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveTransforms.Reserve(PrimitiveTransforms.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveSceneProxies.Reserve(PrimitiveSceneProxies.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveBounds.Reserve(PrimitiveBounds.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveFlagsCompact.Reserve(PrimitiveFlagsCompact.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveVisibilityIds.Reserve(PrimitiveVisibilityIds.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveOcclusionFlags.Reserve(PrimitiveOcclusionFlags.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveComponentIds.Reserve(PrimitiveComponentIds.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveVirtualTextureFlags.Reserve(PrimitiveVirtualTextureFlags.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveVirtualTextureLod.Reserve(PrimitiveVirtualTextureLod.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitiveOcclusionBounds.Reserve(PrimitiveOcclusionBounds.Num() + AddedLocalPrimitiveSceneInfos.Num());
			PrimitivesNeedingStaticMeshUpdate.Reserve(PrimitivesNeedingStaticMeshUpdate.Num() + AddedLocalPrimitiveSceneInfos.Num());
		}

		while (AddedLocalPrimitiveSceneInfos.Num())
		{
			// ⭐
			// 제거된 PrimitiveSceneInfos를 처리할 때와 비슷하게 같은 TypeHash를 가진 PrimitiveSceneInfos를 한꺼번에 처리한다.
			// ⭐
			int StartIndex = AddedLocalPrimitiveSceneInfos.Num() - 1;
			SIZE_T InsertProxyHash = AddedLocalPrimitiveSceneInfos[StartIndex]->Proxy->GetTypeHash();

			while (StartIndex > 0 && AddedLocalPrimitiveSceneInfos[StartIndex - 1]->Proxy->GetTypeHash() == InsertProxyHash)
			{
				StartIndex--;
			}

			for (int AddIndex = StartIndex; AddIndex < AddedLocalPrimitiveSceneInfos.Num(); AddIndex++)
			{
				// ⭐
				// 여기서 당장 TypeHash를 뭉쳐서 Insert해주지는 않고,
				// 여기서는 일단 그냥 리스트에 마지막에 넣음.
				// 같은 TypeHash끼리 Primitive를 뭉쳐주는 동작은 밑에 있다.
				// ⭐
				FPrimitiveSceneInfo* PrimitiveSceneInfo = AddedLocalPrimitiveSceneInfos[AddIndex];
				Primitives.Add(PrimitiveSceneInfo);
				const FMatrix LocalToWorld = PrimitiveSceneInfo->Proxy->GetLocalToWorld();
				PrimitiveTransforms.Add(LocalToWorld);
				PrimitiveSceneProxies.Add(PrimitiveSceneInfo->Proxy);
				PrimitiveBounds.AddUninitialized();
				PrimitiveFlagsCompact.AddUninitialized();
				PrimitiveVisibilityIds.AddUninitialized();
				PrimitiveOcclusionFlags.AddUninitialized();
				PrimitiveComponentIds.AddUninitialized();
				PrimitiveVirtualTextureFlags.AddUninitialized();
				PrimitiveVirtualTextureLod.AddUninitialized();
				PrimitiveOcclusionBounds.AddUninitialized();
				PrimitivesNeedingStaticMeshUpdate.Add(false);

				const int SourceIndex = PrimitiveSceneProxies.Num() - 1;
				PrimitiveSceneInfo->PackedIndex = SourceIndex;

				// ⭐
 				// FGPUScene::PrimitivesToUpdate에 Primitive를 추가해준다.
				// ⭐
				AddPrimitiveToUpdateGPU(*this, SourceIndex);
			}

			bool EntryFound = false;
			int BroadIndex = -1;
			//broad phase search for a matching type
			for (BroadIndex = TypeOffsetTable.Num() - 1; BroadIndex >= 0; BroadIndex--)
			{
				// example how the prefix sum of the tails could look like
				// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,2,1,1,1,7,4,8]
				// TypeOffsetTable[3,8,12,15,16,17,18]

				// ⭐
				// TypeOffsetTable에 추가하려는 SceneInfo의 TypeHash가 이미 존재하는지 확인
				// ⭐
				if (TypeOffsetTable[BroadIndex].PrimitiveSceneProxyType == InsertProxyHash)
				{
					EntryFound = true;
					break;
				}
			}

			//new type encountered
			if (EntryFound == false)
			{
				// ⭐
				// 아직 TypeHash에 대한 Element가 TypeOffsetTable에 없는 경우 추가해준다.
				// ⭐
				BroadIndex = TypeOffsetTable.Num();
				if (BroadIndex)
				{
					FTypeOffsetTableEntry Entry = TypeOffsetTable[BroadIndex - 1];
					//adding to the end of the list and offset of the tail (will will be incremented once during the while loop)
					TypeOffsetTable.Push(FTypeOffsetTableEntry(InsertProxyHash, Entry.Offset));
				}
				else
				{
					//starting with an empty list and offset zero (will will be incremented once during the while loop)
					TypeOffsetTable.Push(FTypeOffsetTableEntry(InsertProxyHash, 0));
				}
			}

			{
				SCOPED_NAMED_EVENT(FScene_SwapPrimitiveSceneInfos, FColor::Turquoise);

				for (int AddIndex = StartIndex; AddIndex < AddedLocalPrimitiveSceneInfos.Num(); AddIndex++)
				{
					int SourceIndex = AddedLocalPrimitiveSceneInfos[AddIndex]->PackedIndex;

					for (int TypeIndex = BroadIndex; TypeIndex < TypeOffsetTable.Num(); TypeIndex++)
					{
						FTypeOffsetTableEntry& NextEntry = TypeOffsetTable[TypeIndex];
						int DestIndex = NextEntry.Offset++; //prepare swap and increment

						// ⭐
						// ex) 위에서 새로 추가해둔 Primitive에 대해 ( Type '6' ) 같은 Type끼리 뭉쳐주는 과정.
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,2,2,2,2,1,1,1,7,4,8,6]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,6,2,2,2,1,1,1,7,4,8,2]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,6,2,2,2,2,1,1,7,4,8,1]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,6,2,2,2,2,1,1,1,4,8,7]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,6,2,2,2,2,1,1,1,7,8,4]
						// PrimitiveSceneProxies[0,0,0,6,6,6,6,6,6,2,2,2,2,1,1,1,7,4,8]
						//
						// 위에서는 새로 추가된 Primitive를 리스트에 마지막에 넣어두었고
						// 이제는 마지막에 넣어둔 Primtive에 대해 리스트상에서 같은 TypeHash끼리 Primitive를 뭉쳐주는 동작
						//
						// ⭐

						if (DestIndex != SourceIndex)
						{
							checkfSlow(SourceIndex > DestIndex, TEXT("Corrupted Prefix Sum [%d, %d]"), SourceIndex, DestIndex);
							Primitives[DestIndex]->PackedIndex = SourceIndex;
							Primitives[SourceIndex]->PackedIndex = DestIndex;

							TArraySwapElements(Primitives, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveTransforms, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveSceneProxies, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveBounds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveFlagsCompact, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVisibilityIds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveOcclusionFlags, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveComponentIds, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVirtualTextureFlags, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveVirtualTextureLod, DestIndex, SourceIndex);
							TArraySwapElements(PrimitiveOcclusionBounds, DestIndex, SourceIndex);
							TBitArraySwapElements(PrimitivesNeedingStaticMeshUpdate, DestIndex, SourceIndex);

							AddPrimitiveToUpdateGPU(*this, DestIndex);
						}
					}
				}
			}

			CheckPrimitiveArrays();

			for (int AddIndex = StartIndex; AddIndex < AddedLocalPrimitiveSceneInfos.Num(); AddIndex++)
			{
				FPrimitiveSceneInfo* PrimitiveSceneInfo = AddedLocalPrimitiveSceneInfos[AddIndex];
				FScopeCycleCounter Context(PrimitiveSceneInfo->Proxy->GetStatId());
				int32 PrimitiveIndex = PrimitiveSceneInfo->PackedIndex;

				// Add the primitive to its shadow parent's linked list of children.
				// Note: must happen before AddToScene because AddToScene depends on LightingAttachmentRoot
				PrimitiveSceneInfo->LinkAttachmentGroup();
			}


			{
				SCOPED_NAMED_EVENT(FScene_AddPrimitiveSceneInfoToScene, FColor::Turquoise);
				if (GIsEditor)
				{
					//
					// ...
					// ...
					// ...
					//
				}
				else
				{
					const bool bAddToDrawLists = !(CVarDoLazyStaticMeshUpdate.GetValueOnRenderThread());
					// ⭐
					// bAddToDrawLists 값은 기본 셋팅으로는 true이다.
					// ⭐
					if (bAddToDrawLists)
					{
						// ⭐⭐⭐⭐⭐⭐⭐
						// 추가된 FPrimitiveSceneInfo를 한꺼번에 Scene에 추가해줌.
						//
						// FPrimitiveSceneInfo::AddToScene 매우 중요한 함수이다.
						// FMeshBatch, FMeshDrawCommand가 생성되고 FScene에 캐싱되는 함수이다.
						// FMeshBatch, FMeshDrawCommand는 실제 렌더링 API에 넘겨질 모든 데이터를 가지고 있다.
						// 이 두 데이터만 가지고 있으면 렌더링 API를 호출해서 렌더링을 수행할 수 있다. 다른건 필요없다.       
						//
						// ---------------------------
						//
						//
						//
						// /* 
						// A batch of mesh elements, all with the same material and vertex buffer 
						// */
						// struct FMeshBatch
						// {
						//
						// ...
						// ...
						// ...
						//
						// };
						//
						// ---------------------------
						//
						// /*
						// FMeshDrawCommand fully describes a mesh pass draw call, captured just above the RHI.  
						// FMeshDrawCommand should contain only data needed to draw.  
						// For InitViews payloads, use / FVisibleMeshDrawCommand.
						// FMeshDrawCommands are cached at Primitive AddToScene time for vertex factories that support it 
						// (no per-frame or per-view shader binding changes).
						// Dynamic Instancing operates at the FMeshDrawCommand level for robustness.
						// Adding per-command shader bindings will reduce the efficiency of Dynamic Instancing, 
						// but rendering will always be correct.
						// Any resources referenced by a command must be kept alive for the lifetime of the command.  
						// FMeshDrawCommand is not responsible for lifetime management of resources.
						// For uniform buffers referenced by cached FMeshDrawCommand's, 
						// RHIUpdateUniformBuffer makes it possible to access per-frame data in the shader without changing bindings.
						// */
						// class FMeshDrawCommand
						// {
						//
						// ...
						// ...
						// ...
						// 
						// };
						//
						// ---------------------------
						//
						// 아주 아주 중요하다.
						//
						// 아래의 링크를 타고 들어가면 자세한 분석을 볼 수 있다.
						// 반드시 읽기를 강추드립니다. 
						// https://scahp.tistory.com/74?category=848072
						// https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/
						FPrimitiveSceneInfo::AddToScene(RHICmdList, this, TArrayView<FPrimitiveSceneInfo*>(&AddedLocalPrimitiveSceneInfos[StartIndex], AddedLocalPrimitiveSceneInfos.Num() - StartIndex), true, true, bAsyncCreateLPIs);
						// ⭐⭐⭐⭐⭐⭐⭐
					}
					else
					{
						FPrimitiveSceneInfo::AddToScene(RHICmdList, this, TArrayView<FPrimitiveSceneInfo*>(&AddedLocalPrimitiveSceneInfos[StartIndex], AddedLocalPrimitiveSceneInfos.Num() - StartIndex), true, false, bAsyncCreateLPIs);

						for (int AddIndex = StartIndex; AddIndex < AddedLocalPrimitiveSceneInfos.Num(); AddIndex++)
						{
							FPrimitiveSceneInfo* PrimitiveSceneInfo = AddedLocalPrimitiveSceneInfos[AddIndex];
							PrimitiveSceneInfo->BeginDeferredUpdateStaticMeshes();
						}
					}
				}
			}

			for (int AddIndex = StartIndex; AddIndex < AddedLocalPrimitiveSceneInfos.Num(); AddIndex++)
			{
				FPrimitiveSceneInfo* PrimitiveSceneInfo = AddedLocalPrimitiveSceneInfos[AddIndex];
				int32 PrimitiveIndex = PrimitiveSceneInfo->PackedIndex;

				if (ShouldPrimitiveOutputVelocity(PrimitiveSceneInfo->Proxy, GetShaderPlatform()))
				{
					// We must register the initial LocalToWorld with the velocity state. 
					// In the case of a moving component with MarkRenderStateDirty() called every frame, UpdateTransform will never happen.
					VelocityData.UpdateTransform(PrimitiveSceneInfo, PrimitiveTransforms[PrimitiveIndex], PrimitiveTransforms[PrimitiveIndex]);
				}

				// 추가된 Primitive를 Scene에 추가.
				AddPrimitiveToUpdateGPU(*this, PrimitiveIndex);

				// Invalidate PathTraced image because we added something to the scene
				bPathTracingNeedsInvalidation = true;

				DistanceFieldSceneData.AddPrimitive(PrimitiveSceneInfo);

				// Flush virtual textures touched by primitive
				PrimitiveSceneInfo->FlushRuntimeVirtualTexture();

				// Set LOD parent information if valid
				PrimitiveSceneInfo->LinkLODParentComponent();

				// Update scene LOD tree
				SceneLODHierarchy.UpdateNodeSceneInfo(PrimitiveSceneInfo->PrimitiveComponentId, PrimitiveSceneInfo);
			}
			AddedLocalPrimitiveSceneInfos.RemoveAt(StartIndex, AddedLocalPrimitiveSceneInfos.Num() - StartIndex);
		}
	}

	// ⭐
	// 새로 추가된 Primitive를 Scene에 반영해주는 작업도 끝났다!
	// ⭐

	{
		// ⭐
		// 이제 Transform 값이 변경된 Primitive ( SceneComponent의 Trasnfrom 데이터가 변경된 Primitive Component )들을 업데이트해주겠다.
		// ⭐


		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(UpdatePrimitiveTransform);
		SCOPED_NAMED_EVENT(FScene_AddPrimitiveSceneInfos, FColor::Yellow);
		SCOPE_CYCLE_COUNTER(STAT_UpdatePrimitiveTransformRenderThreadTime);

		TArray<FPrimitiveSceneInfo*> UpdatedSceneInfosWithStaticDrawListUpdate;
		TArray<FPrimitiveSceneInfo*> UpdatedSceneInfosWithoutStaticDrawListUpdate;
		UpdatedSceneInfosWithStaticDrawListUpdate.Reserve(UpdatedTransforms.Num());
		UpdatedSceneInfosWithoutStaticDrawListUpdate.Reserve(UpdatedTransforms.Num());

		// ⭐⭐⭐⭐⭐⭐⭐
		for (const auto& Transform : UpdatedTransforms)
		{
			// ⭐
			// Transform : 새로 업데이트된 Transform 데이터
			//
			// TMap<FPrimitiveSceneProxy*, FUpdateTransformCommand> UpdatedTransforms;
			// Key : Transform이 업데이트된 Primitive Component의 FPrimitiveSceneProxy
			// Value : 새로 업데이트된 Transform 데이터
			// ⭐
			FPrimitiveSceneProxy* PrimitiveSceneProxy = Transform.Key;
			if (DeletedSceneInfos.Contains(PrimitiveSceneProxy->GetPrimitiveSceneInfo()))
			{
				continue;
			}
			check(PrimitiveSceneProxy->GetPrimitiveSceneInfo()->PackedIndex != INDEX_NONE);

			const FBoxSphereBounds& WorldBounds = Transform.Value.WorldBounds;
			const FBoxSphereBounds& LocalBounds = Transform.Value.LocalBounds;
			const FMatrix& LocalToWorld = Transform.Value.LocalToWorld;
			const FVector& AttachmentRootPosition = Transform.Value.AttachmentRootPosition;
			FScopeCycleCounter Context(PrimitiveSceneProxy->GetStatId());

			// ⭐
			// PrimitiveSceneProxy와 매핑되어 있는 FPrimitiveSceneInfo를 가져옴
			// ⭐
			FPrimitiveSceneInfo* PrimitiveSceneInfo = PrimitiveSceneProxy->GetPrimitiveSceneInfo();
			
			//
			// 이 PrimitiveSceneProxy가 Uniform Buffer를 사용하지 않는다면 StaticDrawList는 업데이트 안해주어도 된다. -> 빠름
			const bool bUpdateStaticDrawLists = !PrimitiveSceneProxy->StaticElementsAlwaysUseProxyPrimitiveUniformBuffer();
			
			if (bUpdateStaticDrawLists)
			{
				UpdatedSceneInfosWithStaticDrawListUpdate.Push(PrimitiveSceneInfo);
			}
			else
			{
				UpdatedSceneInfosWithoutStaticDrawListUpdate.Push(PrimitiveSceneInfo);
			}

			PrimitiveSceneInfo->FlushRuntimeVirtualTexture();

			// ⭐⭐⭐⭐⭐⭐⭐
			// Remove the primitive from the scene at its old location
			// (note that the octree update relies on the bounds not being modified yet).
			//
			// Transform 위치가 변경되기 전에 Scene에 추가했던 Primitive를 Scene에서 제거해줌.
			PrimitiveSceneInfo->RemoveFromScene(bUpdateStaticDrawLists);
			// ⭐⭐⭐⭐⭐⭐⭐

			if (ShouldPrimitiveOutputVelocity(PrimitiveSceneInfo->Proxy, GetShaderPlatform()))
			{
				VelocityData.UpdateTransform(PrimitiveSceneInfo, LocalToWorld, PrimitiveSceneProxy->GetLocalToWorld());
			}

			// ⭐
			// Update the primitive transform.
			//
			// PrimitiveSceneProxy에 Transform 데이터를 업데이트 해줌
			// ⭐
			PrimitiveSceneProxy->SetTransform(LocalToWorld, WorldBounds, LocalBounds, AttachmentRootPosition);
			PrimitiveTransforms[PrimitiveSceneInfo->PackedIndex] = LocalToWorld;

			if (!RHISupportsVolumeTextures(GetFeatureLevel())
				&& (PrimitiveSceneProxy->IsMovable() || PrimitiveSceneProxy->NeedsUnbuiltPreviewLighting() || PrimitiveSceneProxy->GetLightmapType() == ELightmapType::ForceVolumetric))
			{
				PrimitiveSceneInfo->MarkIndirectLightingCacheBufferDirty();
			}

			// ⭐⭐⭐⭐⭐⭐⭐
			// 위에서 Scene에서 Primitive를 다시 Scene에 추가해줌
			AddPrimitiveToUpdateGPU(*this, PrimitiveSceneInfo->PackedIndex);
			// ⭐⭐⭐⭐⭐⭐⭐

			DistanceFieldSceneData.UpdatePrimitive(PrimitiveSceneInfo);

			// If the primitive has static mesh elements, it should have returned true from ShouldRecreateProxyOnUpdateTransform!
			check(!(bUpdateStaticDrawLists && PrimitiveSceneInfo->StaticMeshes.Num()));
		}
		// ⭐⭐⭐⭐⭐⭐⭐

		// Re-add the primitive to the scene with the new transform.
		if (UpdatedSceneInfosWithStaticDrawListUpdate.Num() > 0)
		{
			FPrimitiveSceneInfo::AddToScene(RHICmdList, this, UpdatedSceneInfosWithStaticDrawListUpdate, true, true, bAsyncCreateLPIs);
		}

		if (UpdatedSceneInfosWithoutStaticDrawListUpdate.Num() > 0)
		{
			FPrimitiveSceneInfo::AddToScene(RHICmdList, this, UpdatedSceneInfosWithoutStaticDrawListUpdate, false, true, bAsyncCreateLPIs);
			for (FPrimitiveSceneInfo* PrimitiveSceneInfo : UpdatedSceneInfosWithoutStaticDrawListUpdate)
			{
				PrimitiveSceneInfo->FlushRuntimeVirtualTexture();
			}
		}

		if (AsyncCreateLightPrimitiveInteractionsTask && AsyncCreateLightPrimitiveInteractionsTask->GetTask().HasPendingPrimitives())
		{
			check(GAsyncCreateLightPrimitiveInteractions);
			AsyncCreateLightPrimitiveInteractionsTask->StartBackgroundTask();
		}

		for (const auto& Transform : OverridenPreviousTransforms)
		{
			FPrimitiveSceneInfo* PrimitiveSceneInfo = Transform.Key;
			VelocityData.OverridePreviousTransform(PrimitiveSceneInfo->PrimitiveComponentId, Transform.Value);
		}
	}

	for (const auto& Attachments : UpdatedAttachmentRoots)
	{
		FPrimitiveSceneInfo* PrimitiveSceneInfo = Attachments.Key;
		if (DeletedSceneInfos.Contains(PrimitiveSceneInfo))
		{
			continue;
		}

		PrimitiveSceneInfo->UnlinkAttachmentGroup();
		PrimitiveSceneInfo->LightingAttachmentRoot = Attachments.Value;
		PrimitiveSceneInfo->LinkAttachmentGroup();
	}

	for (const auto& CustomParams : UpdatedCustomPrimitiveParams)
	{
		FPrimitiveSceneProxy* PrimitiveSceneProxy = CustomParams.Key;
		if (DeletedSceneInfos.Contains(PrimitiveSceneProxy->GetPrimitiveSceneInfo()))
		{
			continue;
		}

		FScopeCycleCounter Context(PrimitiveSceneProxy->GetStatId());
		PrimitiveSceneProxy->CustomPrimitiveData = CustomParams.Value;

		// Make sure the uniform buffer is updated before rendering
		PrimitiveSceneProxy->GetPrimitiveSceneInfo()->SetNeedsUniformBufferUpdate(true);
	}

	for (FPrimitiveSceneInfo* PrimitiveSceneInfo : DistanceFieldSceneDataUpdates)
	{
		if (DeletedSceneInfos.Contains(PrimitiveSceneInfo))
		{
			continue;
		}

		DistanceFieldSceneData.UpdatePrimitive(PrimitiveSceneInfo);
	}

	{
		SCOPED_NAMED_EVENT(FScene_DeletePrimitiveSceneInfo, FColor::Red);
		for (FPrimitiveSceneInfo* PrimitiveSceneInfo : DeletedSceneInfos)
		{
			// It is possible that the HitProxies list isn't empty if PrimitiveSceneInfo was Added/Removed in same frame
			// Delete the PrimitiveSceneInfo on the game thread after the rendering thread has processed its removal.
			// This must be done on the game thread because the hit proxy references (and possibly other members) need to be freed on the game thread.
			struct DeferDeleteHitProxies : FDeferredCleanupInterface
			{
				DeferDeleteHitProxies(TArray<TRefCountPtr<HHitProxy>>&& InHitProxies) : HitProxies(MoveTemp(InHitProxies)) {}
				TArray<TRefCountPtr<HHitProxy>> HitProxies;
			};

			BeginCleanup(new DeferDeleteHitProxies(MoveTemp(PrimitiveSceneInfo->HitProxies)));


			// ⭐
			// free the primitive scene proxy.
			//
			// PrimitiveSceneInfo를 정리(제거) 해준다.
			//
			// PrimitiveSceneInfo와 매칭되는 FPrimitiveSceneProxy도 함께 제거해준다.
			// 게임 스레드에서 생성해준 PrimitiveSceneInfo와와 FPrimitiveSceneProxy를 렌더스레드에서 제거해주는 것을 알 수 있다.
			// 생성은 게임 스레드가, 제거는 렌더스레드가 맡으므로서 게임 스레드에서 PrimitiveComponnet를 제거해주더라도
			// 게임 스레드를 뒤따라오는 렌더스레드가 제거 되기 전 프레임에서 해당 Primitive를 안전하게 렌더링할 수 있다.
			// ⭐
			delete PrimitiveSceneInfo->Proxy;
			delete PrimitiveSceneInfo;


			// Invalidate PathTraced image because we removed something from the scene
			bPathTracingNeedsInvalidation = true;
		}
	}

	UpdatedAttachmentRoots.Reset();
	UpdatedTransforms.Reset();
	UpdatedCustomPrimitiveParams.Reset();
	OverridenPreviousTransforms.Reset();
	DistanceFieldSceneDataUpdates.Reset();
	RemovedPrimitiveSceneInfos.Reset();
	AddedPrimitiveSceneInfos.Reset();
}
```



------------------------------         

references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
