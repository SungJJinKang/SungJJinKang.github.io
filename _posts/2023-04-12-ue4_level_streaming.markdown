---
layout: post
title:  "언리얼 엔진4 레벨 스트리밍 코드 분석 ( LoadStreamLevel(), ULevelStreaming, StreamingManager... ) ( 작성 중 )"
date:   2023-04-12
tags: [UE]
---


중요도가 떨어지는 코드는        
```

...
...
...

```
으로 처리해두었으니 확인하고 싶은 경우에는 UE4 소스코드를 확인해주세요.           

------------------------         

```cpp
void UGameplayStatics::LoadStreamLevel
(
	const UObject* WorldContextObject, 
	FName LevelName, 
	bool bMakeVisibleAfterLoad /* ⭐ 스트리밍 레벨이 로드된 후 바로 Visible하게 만들지 ⭐ */, 
	bool bShouldBlockOnLoad /* ⭐ 스트리밍 레벨 패키지 로드시 게임 스레드를 블록할지 ⭐ */, 
	FLatentActionInfo LatentInfo
)
{
	if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
	{
		FLatentActionManager& LatentManager = World->GetLatentActionManager();
		if (LatentManager.FindExistingAction<FStreamLevelAction>(LatentInfo.CallbackTarget, LatentInfo.UUID) == nullptr)
		{
			FStreamLevelAction* NewAction = new FStreamLevelAction
				(
					true, 
					LevelName, 
					bMakeVisibleAfterLoad, 
					bShouldBlockOnLoad /* ⭐ */, 
					LatentInfo, World
				); // ⭐
			LatentManager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, NewAction);
		}
	}
}

FStreamLevelAction::FStreamLevelAction(bool bIsLoading, const FName& InLevelName, bool bIsMakeVisibleAfterLoad, bool bInShouldBlock, const FLatentActionInfo& InLatentInfo, UWorld* World)
	: bLoading(bIsLoading)
	, bMakeVisibleAfterLoad(bIsMakeVisibleAfterLoad)
	, bShouldBlock(bInShouldBlock)
	, LevelName(InLevelName)
	, LatentInfo(InLatentInfo)
{
	ULevelStreaming* LocalLevel = FindAndCacheLevelStreamingObject( LevelName, World );
	Level = LocalLevel;
	ActivateLevel( LocalLevel ); // ⭐
}

void FStreamLevelAction::ActivateLevel( ULevelStreaming* LevelStreamingObject )
{	
	if (LevelStreamingObject)
	{
		// Loading.
		if (bLoading)
		{
			UE_LOG(LogStreaming, Log, TEXT("Streaming in level %s (%s)..."),*LevelStreamingObject->GetName(),*LevelStreamingObject->GetWorldAssetPackageName());
			LevelStreamingObject->SetShouldBeLoaded(true); // ⭐
			LevelStreamingObject->SetShouldBeVisible(LevelStreamingObject->GetShouldBeVisibleFlag()	|| bMakeVisibleAfterLoad);
			LevelStreamingObject->bShouldBlockOnLoad = bShouldBlock;
		}
		// Unloading.
		else 
		{
			...
            ...
            ...
		}

		// If we have a valid world
		if (UWorld* LevelWorld = LevelStreamingObject->GetWorld())
		{
			...
            ...
            ...
		}
	}

    ...
    ...
    ...
}
```

```cpp
// ⭐⭐⭐⭐⭐⭐⭐
void UWorld::UpdateLevelStreaming()
// ⭐⭐⭐⭐⭐⭐⭐
{
	...
    ...
    ...

	StreamingLevelsToConsider.BeginConsideration();

	for (int32 Index = StreamingLevelsToConsider.GetStreamingLevels().Num() - 1; Index >= 0; --Index)
	{
		if (ULevelStreaming* StreamingLevel = StreamingLevelsToConsider.GetStreamingLevels()[Index])
		{
			bool bUpdateAgain = true;
			bool bShouldContinueToConsider = true;
			while (bUpdateAgain && bShouldContinueToConsider)
			{
				bool bRedetermineTarget = false;

                // ⭐⭐⭐⭐⭐⭐⭐
				FStreamingLevelPrivateAccessor::UpdateStreamingState(StreamingLevel, bUpdateAgain, bRedetermineTarget); 
                // ⭐⭐⭐⭐⭐⭐⭐

				if (bRedetermineTarget)
				{
					bShouldContinueToConsider = FStreamingLevelPrivateAccessor::DetermineTargetState(StreamingLevel);
				}
			}

			if (!bShouldContinueToConsider)
			{
				StreamingLevelsToConsider.RemoveAt(Index);
			}
		}
		else
		{
			StreamingLevelsToConsider.RemoveAt(Index);
		}
	}

	StreamingLevelsToConsider.EndConsideration();

    ...
}

void ULevelStreaming::UpdateStreamingState(bool& bOutUpdateAgain, bool& bOutRedetermineTarget)
{
	...
    ...
    ...

	auto UpdateStreamingState_RequestLevel = [&]()
	{
		...
        ...
        ...

        // ⭐ 
		// 위에서 호출한 UGameplayStatics::LoadStreamLevel 함수에 매개변수로 전달하였던 bShouldBlockOnLoad이 false인 경우, ASync로 처리 
		// ⭐
		bool bBlockOnLoad = (bShouldBlockOnLoad || ShouldBeAlwaysLoaded()); 

		const bool bAllowLevelLoadRequests = (bBlockOnLoad || World->AllowLevelLoadRequests());
		bBlockOnLoad |= (!GUseBackgroundLevelStreaming || !World->IsGameWorld());

		const ECurrentState PreviousState = CurrentState;

        // ⭐⭐⭐⭐⭐⭐⭐
		// ⭐ 레벨을 요청
		RequestLevel(World, bAllowLevelLoadRequests, (bBlockOnLoad ? ULevelStreaming::AlwaysBlock : ULevelStreaming::BlockAlwaysLoadedLevelsOnly)); 
        // ⭐⭐⭐⭐⭐⭐⭐

		...
	};

	...
    ... // ⭐ 현재 스트리밍 레벨 로딩 상태에 따른 여러 동작들 ⭐
    ... // ⭐ ex) 현재 레벨이 언로드 상태이다 ( 로딩 중도 아니다 ), 그런데 목표로 하는 상태는 로드된 상태이다-> 레벨 로드 요청  ⭐
    ... 
}
```

```cpp
bool ULevelStreaming::RequestLevel(UWorld* PersistentWorld, bool bAllowLevelLoadRequests, EReqLevelBlock BlockPolicy)
{
    ...
	... // ⭐ 이미 로딩 중이거나, 로드 실패인 경우 return ⭐
    ...

    const bool bIsGameWorld = PersistentWorld->IsGameWorld();
    // ⭐ 여기서 로드하려는 레벨이 속해 있는 Package명을 가져옴 ⭐ 
	const FName DesiredPackageName = bIsGameWorld ? GetLODPackageName() : GetWorldAssetPackageFName(); 
	const FName LoadedLevelPackageName = GetLoadedLevelPackageName();

	...
	... // ⭐ 이미 로딩 완료되었거나, 언로드 중인 경우 return ⭐
    ...

	...
	... // ⭐ 여러 예외 처리 ⭐
    ...

    ...
	... // ⭐ 여기서 로드하려는 스트리밍 레벨이 유니크한지 확인하는 코드. ⭐
    ...

	TRACE_LOADTIME_REQUEST_GROUP_SCOPE(TEXT("LevelStreaming - %s"), *GetPathName());

	// Try to find the [to be] loaded package.
    // ⭐ 스트리밍 레벨이 속한 패키지를 메모리에서 찾아봄. ( 이미 메모리에 올라와있는지 확인하는 코드 ) ⭐
	UPackage* LevelPackage = (UPackage*)StaticFindObjectFast(UPackage::StaticClass(), nullptr, DesiredPackageName, 0, 0, RF_NoFlags, EInternalObjectFlags::PendingKill); 

    ...
	... // ⭐ 현재 에디터에서 실행되는 중이라면 월드가 에디터에 이미 로드되어 있는지를 확인하고, 그렿다면 바로 가져옴. ( 로드 성공! ) ⭐
    ...

	// Package is already or still loaded.
    // ⭐ 패키지가 이미 로드가 되어 있는 경우 ( 이미 패키지가 메모리에 올라와있는 경우 ) ⭐
	if (LevelPackage)
	{
		// Find world object and use its PersistentLevel pointer.
        // ⭐ 이미 로드되어 있는 패키지에서 레벨을 바로 가져옴. ⭐
		UWorld* World = UWorld::FindWorldInPackage(LevelPackage);

		// Check for a redirector. Follow it, if found.
		if (!World)
		{
			World = UWorld::FollowWorldRedirectorInPackage(LevelPackage);
			if (World)
			{
				LevelPackage = World->GetOutermost();
			}
		}

		if (World != nullptr)
		{
			...
            ... // ⭐ 각종 예외 처리 코드 ⭐
            ...

            // Level already exists but may have the wrong type due to being inactive before, so copy data over
            World->WorldType = PersistentWorld->WorldType;

			// ⭐ 
			// 로드 된 스트리밍 레벨의 부모 레벨을 현재의 Persistent Level로 셋팅 
			// ⭐
            World->PersistentLevel->OwningWorld = PersistentWorld; 

			// ⭐ 
			// 레벨 로드 완료! 
			// ⭐
            SetLoadedLevel(World->PersistentLevel); 

            // Broadcast level loaded event to blueprints
			// ⭐ 
			// 레벨 로드 완료 콜백 호출 
			// ⭐
            OnLevelLoaded.Broadcast(); 
			
			
			return true;
		}
	}

	// Async load package if world object couldn't be found and we are allowed to request a load.
    // ⭐ 위에서 호출한 UGameplayStatics::LoadStreamLevel 함수에 매개변수로 전달하였던 bShouldBlockOnLoad이 false인 경우, ⭐ 
    // ⭐ ASync 레벨이 속한 패키지를 로드로 처리 ⭐
	if (bAllowLevelLoadRequests)
	{
		const FName DesiredPackageNameToLoad = bIsGameWorld ? GetLODPackageNameToLoad() : PackageNameToLoad;
		const FString PackageNameToLoadFrom = DesiredPackageNameToLoad != NAME_None ? DesiredPackageNameToLoad.ToString() : DesiredPackageName.ToString();

		if (FPackageName::DoesPackageExist(PackageNameToLoadFrom))
		{
            // ⭐ 패키지가 존재하는 경우 ⭐

			CurrentState = ECurrentState::Loading;
			FWorldNotifyStreamingLevelLoading::Started(PersistentWorld);
			
			ULevel::StreamedLevelsOwningWorld.Add(DesiredPackageName, PersistentWorld);
			UWorld::WorldTypePreLoadMap.FindOrAdd(DesiredPackageName) = PersistentWorld->WorldType;

			// Kick off async load request.
			STAT_ADD_CUSTOMMESSAGE_NAME( STAT_NamedMarker, *(FString( TEXT( "RequestLevel - " ) + DesiredPackageName.ToString() )) );
			TRACE_BOOKMARK(TEXT("RequestLevel - %s"), *DesiredPackageName.ToString());

            // ⭐⭐⭐⭐⭐⭐⭐
            // ASync로 패키지를 로드! 
			// [참고 자료](https://sungjjinkang.github.io/ue4_async_load)
			LoadPackageAsync
            (
                DesiredPackageName.ToString(), 
                nullptr, 
                *PackageNameToLoadFrom, 

                // ⭐ ASync 로딩 완료시 호출될 콜백 ⭐ 
                FLoadPackageAsyncDelegate::CreateUObject(this, &ULevelStreaming::AsyncLevelLoadComplete), 

                PackageFlags,
                PIEInstanceID,
                GetPriority()
            ;
            // ⭐⭐⭐⭐⭐⭐⭐

			// streamingServer: server loads everything?
			// Editor immediately blocks on load and we also block if background level streaming is disabled.
			if (BlockPolicy == AlwaysBlock || (ShouldBeAlwaysLoaded() && BlockPolicy != NeverBlock))
			{
                // ⭐ 
				// 위에서 호출한 UGameplayStatics::LoadStreamLevel 함수에 
				// 매개변수로 전달하였던 bShouldBlockOnLoad에 true를 전달한 경우 
				// ⭐ 
				if (IsAsyncLoading())
				{
					UE_LOG(LogStreaming, Display, TEXT("ULevelStreaming::RequestLevel(%s) is flushing async loading"), *DesiredPackageName.ToString());
				}

				// Finish all async loading.
                // ⭐ ASync 로딩을 Flush 해버림 -> Sync 로딩으로 처리 ⭐
				FlushAsyncLoading();
			}
		}
		else
		{
			UE_LOG(LogStreaming, Error,TEXT("Couldn't find file for package %s."), *PackageNameToLoadFrom);
			CurrentState = ECurrentState::FailedToLoad;
			return false;
		}
	}

	return true;
}
```
        
         
여기서 문제의 코드가 등장한다.                  
**스트리밍 레벨이 로드가 완료되었음에도 간혹 텍스쳐(메쉬 텍스쳐, 라이트맵...)가 뒤늦게 로드되는 현상이 발생**한다.       
( 필자도 실제로 스트리밍 레벨의 로드가 완료되면 화면을 가리던 Loading UI를 없앴는데, 이후 뒤늦게 텍스쳐들이 로딩되는 문제가 있었다. )               
이는 **스트리밍 레벨에서 필요한 텍스쳐와 같은 각종 리소스를 Streaming으로 로드해오기 때문에 발생**한다.        
리소스 로드가 늦어지면 간혹 카메라에 팝업 현상이 보이곤 한다.           


```cpp
void ULevelStreaming::AsyncLevelLoadComplete(const FName& InPackageName, UPackage* InLoadedPackage, EAsyncLoadingResult::Type Result)
{
	CurrentState = ECurrentState::LoadedNotVisible;
	if (UWorld* World = GetWorld())
	{
		if (World->GetStreamingLevels().Contains(this))
		{
			// ⭐ 
			// 스트리밍 레벨 로드가 끝났음을 알림 
			// ⭐
			FWorldNotifyStreamingLevelLoading::Finished(World); 
		}
	}

	if (InLoadedPackage)
	{
		UPackage* LevelPackage = InLoadedPackage;
		
		// Try to find a UWorld object in the level package.
		UWorld* World = UWorld::FindWorldInPackage(LevelPackage);

		if (World)
		{
			ULevel* Level = World->PersistentLevel;
			if (Level)
			{
				UWorld* LevelOwningWorld = Level->OwningWorld;
				if (LevelOwningWorld)
				{
					ULevel* PendingLevelVisOrInvis = (LevelOwningWorld->GetCurrentLevelPendingVisibility() ? LevelOwningWorld->GetCurrentLevelPendingVisibility() : LevelOwningWorld->GetCurrentLevelPendingInvisibility());
					if (PendingLevelVisOrInvis && PendingLevelVisOrInvis == LoadedLevel)
					{
						...
                        ... // ⭐ 
                        ... // 스트리밍 레벨이 가시성 처리 ( 레벨의 오브젝트들을 화면에 보이게 만드는 작업 )를 수행 중인 경우
						... // ⭐ 
						...
					}
					else
					{
						check(PendingUnloadLevel == nullptr);
					
						SetLoadedLevel(Level); // ⭐ 레벨이 로드되었음을 셋팅 ⭐
						// Broadcast level loaded event to blueprints
						OnLevelLoaded.Broadcast(); // ⭐ 레벨 로드 성공 콜백 호출 ⭐
					}
				}

				Level->HandleLegacyMapBuildData();

				// Notify the streamer to start building incrementally the level streaming data.

                // ⭐⭐⭐⭐⭐⭐⭐
                // 문제의 코드!!!!
                // StreamingManager에 이 스트리밍 레벨에 속한 여러 리소스들을 스트리밍 해줄 것을 요청
				IStreamingManager::Get().AddLevel(Level); 
                // ⭐⭐⭐⭐⭐⭐⭐

				...
                ...
                ...
                
			}
			else
			{
				UE_LOG(LogLevelStreaming, Warning, TEXT("Couldn't find ULevel object in package '%s'"), *InPackageName.ToString() );
			}
		}
		else
		{
			...
            ... // ⭐ 로드한 패키지에 원하는 스트리밍 레벨이 없는 경우에 대한 예외 처리 ⭐
            ...
		}
	}
	else if (Result == EAsyncLoadingResult::Canceled)
	{
        // ⭐ 패키지 로드를 취소된 경우 ⭐

		// Cancel level streaming
		CurrentState = ECurrentState::Unloaded;
		SetShouldBeLoaded(false);
	}
	else
	{
        // ⭐ 패키지 로드를 실패한 경우 ⭐

		UE_LOG(LogLevelStreaming, Warning, TEXT("Failed to load package '%s'"), *InPackageName.ToString() );
		
		CurrentState = ECurrentState::FailedToLoad;
 		SetShouldBeLoaded(false);
	}

	...
    ...
    ...

}
```


```cpp
// ⭐ 여러 리소스들의 스트리밍을 관리하는 매니저 인스턴스들을 모아둔 클래스 FStreamingManagerCollection ⭐
FStreamingManagerCollection& IStreamingManager::Get()
{
	if (StreamingManagerCollection == nullptr)
	{
		StreamingManagerCollection = new FStreamingManagerCollection();
	}
	return *StreamingManagerCollection;
}

void FStreamingManagerCollection::AddLevel( ULevel* Level )
{
	// Route to streaming managers.
	for( int32 ManagerIndex=0; ManagerIndex < StreamingManagers.Num(); ManagerIndex++ )
	{
		IStreamingManager* StreamingManager = StreamingManagers[ManagerIndex];
		StreamingManager->AddLevel( Level );
	}
}
```

대표적으로 몇몇 StreamingManager를 살펴보겠다.         

```cpp
// ⭐ 렌더링에 필요한 텍스쳐, 메쉬 데이터의 스트리밍을 담당하는 FRenderAssetStreamingManager ⭐
struct FRenderAssetStreamingManager final : public IRenderAssetStreamingManager

void FRenderAssetStreamingManager::AddLevel( ULevel* Level )
{
	...
    ...
    ...

	// If the level was not already there, create a new one, find an available slot or add a new one.
	RenderAssetInstanceAsyncWork->EnsureCompletion();
	FLevelRenderAssetManager* LevelRenderAssetManager = new FLevelRenderAssetManager(Level, RenderAssetInstanceAsyncWork->GetTask());

	uint32 LevelIndex = LevelRenderAssetManagers.FindLastByPredicate([](FLevelRenderAssetManager* Ptr) { return (Ptr == nullptr); });
	if (LevelIndex != INDEX_NONE)
	{
		LevelRenderAssetManagers[LevelIndex] = LevelRenderAssetManager;
	}
	else
	{
		LevelRenderAssetManagers.Add(LevelRenderAssetManager);
	}
}


FLevelRenderAssetManager::FLevelRenderAssetManager(ULevel* InLevel, RenderAssetInstanceTask::FDoWorkTask& AsyncTask)
	: Level(InLevel)
	, bIsInitialized(false)
	, bHasBeenReferencedToStreamedTextures(false)
	, StaticInstances(AsyncTask)
	, BuildStep(EStaticBuildStep::BuildTextureLookUpMap)
{
	...
    ...
    ...
}
```


```cpp
void IStreamingManager::Tick( float DeltaTime, bool bProcessEverything/*=false*/ )
{
	UpdateResourceStreaming( DeltaTime, bProcessEverything );

    ...
    ...
    ...
}

/** Stages [0,N-2] is non-threaded data collection, Stage N-1 is wait-for-AsyncWork-and-finalize. */
// ⭐ 현재 스트리밍 단계 ⭐
int32 FRenderAssetStreamingManager::ProcessingStage;

/** Total number of processing stages (N). */
// ⭐ 스트리밍 과정이 총 몇 단계인지. ⭐
int32 FRenderAssetStreamingManager::NumRenderAssetProcessingStages;

void FRenderAssetStreamingManager::UpdateResourceStreaming( float DeltaTime, bool bProcessEverything/*=false*/ )
{
	...
    ...
    ...

	RenderAssetInstanceAsyncWork->EnsureCompletion();

	if (NumRenderAssetProcessingStages <= 0 || bProcessEverything)
	{
		ProcessingStage = 0;
		NumRenderAssetProcessingStages = Settings.FramesForFullUpdate;

		// Update Thread Data
		SetLastUpdateTime();
		UpdateStreamingRenderAssets(0, 1, false);

		UpdatePendingStates(true);
		PrepareAsyncTask(bProcessEverything || Settings.bStressTest);

        /** Async work for calculating priorities and target number of mips for all textures/meshes. */
        // ⭐⭐⭐⭐⭐⭐⭐
        // ⭐ 스트리밍할 텍스쳐, 메쉬들에 대한 mip 개수, 스트리밍 우선순위( 어떤 리소스를 먼저 가져올지 )등을 ASync로 연산 해둠. ⭐
		AsyncWork->StartSynchronousTask(); 
        // ⭐⭐⭐⭐⭐⭐⭐

		TickFastResponseAssets();

        // ⭐⭐⭐⭐⭐⭐⭐
        // ⭐ 리소스들을 스트리밍해옴. ⭐
		StreamRenderAssets(bProcessEverything);
        // ⭐⭐⭐⭐⭐⭐⭐
		....
        ....
	}
	else if (ProcessingStage == 0)
	{
		NumRenderAssetProcessingStages = Settings.FramesForFullUpdate;

		// Here we rely on dynamic components to be updated on the last stage, in order to split the workload. 
		UpdatePendingStates(false);
		PrepareAsyncTask(bProcessEverything || Settings.bStressTest);
		AsyncWork->StartBackgroundTask(CVarUseBackgroundThreadPool.GetValueOnGameThread() ? GBackgroundPriorityThreadPool : GThreadPool);
		TickFastResponseAssets();
		++ProcessingStage;
	}
	else if (ProcessingStage <= NumRenderAssetProcessingStages)
	{
		if (ProcessingStage == 1)
		{
			SetLastUpdateTime();
		}

		TickFastResponseAssets();

		FEvent* SyncEvent = nullptr;

        // ⭐ CVarStreamingOverlapAssetAndLevelTicks : "Ticks render asset streaming info on a high priority task thread while ticking levels on GT" ⭐
        // PS4 플랫폼을 제외하고는 기본적으로 false
		const bool bOverlappedExecution = bUseThreadingForPerf && CVarStreamingOverlapAssetAndLevelTicks.GetValueOnGameThread();
		if (bOverlappedExecution)
		{
			...
            ...
            ...
		}
		else
		{
            // ⭐⭐⭐⭐⭐⭐⭐
			UpdateStreamingRenderAssets(ProcessingStage - 1, NumRenderAssetProcessingStages, DeltaTime > 0.f);
            // ⭐⭐⭐⭐⭐⭐⭐
		}

		IncrementalUpdate(1.f / (float)FMath::Max(NumRenderAssetProcessingStages - 1, 1), true); // -1 since we don't want to do anything at stage 0.
		++ProcessingStage;

		if (bOverlappedExecution)
		{
            ...
            ...
            ...
		}
	}
	else if (AsyncWork->IsDone())
	{
		// Since this step is lightweight, tick each texture inflight here, to accelerate the state changes.
		for (int32 TextureIndex : InflightRenderAssets)
		{
			StreamingRenderAssets[TextureIndex].UpdateStreamingStatus(DeltaTime > 0);
		}

		TickFastResponseAssets();

		StreamRenderAssets(bProcessEverything);
		// Release the old view now as the destructors can be expensive. Now only the dynamic manager holds a ref.
		AsyncWork->GetTask().ReleaseAsyncViews();
		IncrementalUpdate(1.f / (float)FMath::Max(NumRenderAssetProcessingStages - 1, 1), true); // Just in case continue any pending update.
		DynamicComponentManager.PrepareAsyncView();

		ProcessingStage = 0;
	}

	if (!bProcessEverything)
	{
		ProcessPendingMipCopyRequests();
	}

	TickDeferredMipLevelChangeCallbacks();

	if (bUseThreadingForPerf)
	{
		RenderAssetInstanceAsyncWork->StartBackgroundTask(GThreadPool);
	}
	else
	{
		RenderAssetInstanceAsyncWork->StartSynchronousTask();
	}
}
```

```cpp
// ⭐⭐⭐⭐⭐⭐⭐
// 텍스쳐, 메쉬들을 스트리밍 해옴.
// 위에서 ASync로 처리해둔 스트리밍 우선 순위에 따라 스트리밍을 수행.
void FRenderAssetStreamingManager::StreamRenderAssets
( 
	// ⭐
	// 모든 텍스쳐를 한번에 처리할지, 여기서는 false
	// ⭐
	bool bProcessEverything 
)
// ⭐⭐⭐⭐⭐⭐⭐
{
	const FRenderAssetStreamingMipCalcTask& AsyncTask = AsyncWork->GetTask();

	if (!bPauseRenderAssetStreaming || bProcessEverything)
	{
		for (int32 AssetIndex : AsyncTask.GetCancelationRequests())
		{
			if (StreamingRenderAssets.IsValidIndex(AssetIndex) && !VisibleFastResponseRenderAssetIndices.Contains(AssetIndex))
			{
                // ⭐ 
				// 스트리밍 와중에도 스트리밍이 ( 당장 ) 필요없는 경우 경우, 요청을 취소함. 
				// ⭐
				StreamingRenderAssets[AssetIndex].CancelStreamingRequest();
			}
		}

		if (!bProcessEverything && ShouldAmortizeMipCopies() /* 엔진 기본 셋팅에서는 false */)
		{
			...
            ...
            ...
		}
		else
		{
			for (int32 AssetIndex : AsyncTask.GetLoadRequests())
			{
				if (StreamingRenderAssets.IsValidIndex(AssetIndex) && !VisibleFastResponseRenderAssetIndices.Contains(AssetIndex))
				{
                    // ⭐⭐⭐⭐⭐⭐⭐
                    // 리소스들을 스트리밍함
					StreamingRenderAssets[AssetIndex].StreamWantedMips(*this);
                    // ⭐⭐⭐⭐⭐⭐⭐
				}
			}
		}
	}
	
	
	for (int32 AssetIndex : AsyncTask.GetPendingUpdateDirties())
	{
		if (StreamingRenderAssets.IsValidIndex(AssetIndex) && !VisibleFastResponseRenderAssetIndices.Contains(AssetIndex))
		{
			FStreamingRenderAsset& StreamingRenderAsset = StreamingRenderAssets[AssetIndex];
			const bool bNewState = StreamingRenderAsset.HasUpdatePending(bPauseRenderAssetStreaming, AsyncTask.HasAnyView());

			// Always update the texture/mesh and the streaming texture/mesh together to make sure they are in sync.
			StreamingRenderAsset.bHasUpdatePending = bNewState;
			if (StreamingRenderAsset.RenderAsset)
			{
				StreamingRenderAsset.RenderAsset->bHasStreamingUpdatePending = bNewState;
			}
		}
	}

	...
    ...
    ...

	VisibleFastResponseRenderAssetIndices.Empty();
}
```


```cpp
void FStreamingRenderAsset::StreamWantedMips(FRenderAssetStreamingManager& Manager, bool bUseCachedData)
{
	if (RenderAsset && !RenderAsset->HasPendingInitOrStreaming())
	{
		const FStreamableRenderResourceState ResourceState = RenderAsset->GetStreamableResourceState();

		const uint32 bLocalForceFullyLoadHeuristic = bUseCachedData ? bCachedForceFullyLoadHeuristic : bForceFullyLoadHeuristic;
		const int32 LocalVisibleWantedMips = bUseCachedData ? CachedVisibleWantedMips : VisibleWantedMips;
		// Update ResidentMips now as it is guarantied to not change here (since no pending requests).
		ResidentMips = ResourceState.NumResidentLODs;

		// Prevent streaming-in optional mips and non optional mips as they are from different files.
		int32 LocalWantedMips = bUseCachedData ? CachedWantedMips : WantedMips;
		if (ResidentMips < ResourceState.NumNonOptionalLODs && LocalWantedMips > ResourceState.NumNonOptionalLODs)
		{ 
			LocalWantedMips = ResourceState.NumNonOptionalLODs;
		}

		if (LocalWantedMips != ResidentMips)
		{
			if (LocalWantedMips < ResidentMips)
			{
                // ⭐⭐⭐⭐⭐⭐⭐
				RenderAsset->StreamOut(LocalWantedMips);
                // ⭐⭐⭐⭐⭐⭐⭐
			}
			else // WantedMips > ResidentMips
			{
				const bool bShouldPrioritizeAsyncIORequest = (bLocalForceFullyLoadHeuristic || bIsTerrainTexture || bLoadWithHigherPriority) && LocalWantedMips <= LocalVisibleWantedMips;
                
                // ⭐⭐⭐⭐⭐⭐⭐
				RenderAsset->StreamIn(LocalWantedMips, bShouldPrioritizeAsyncIORequest);
                // ⭐⭐⭐⭐⭐⭐⭐
			}
			UpdateStreamingStatus(false);
			TrackRenderAssetEvent(this, RenderAsset, bLocalForceFullyLoadHeuristic != 0, &Manager);
		}
	}
}

```

```cpp
/**
 * Not thread-safe: Updates a portion (as indicated by 'StageIndex') of all streaming textures,
 * allowing their streaming state to progress.
 *
 * @param Context			Context for the current stage (frame)
 * @param StageIndex		Current stage index
 * @param NumUpdateStages	Number of texture update stages
 */
void FRenderAssetStreamingManager::UpdateStreamingRenderAssets(int32 StageIndex, int32 NumUpdateStages, bool bWaitForMipFading, bool bAsync)
{
	...
    ...
    ...

#if !STATS
	if (GAllowParallelUpdateStreamingRenderAssets)
	{
		struct FPacket
		{
			FPacket(int32 InStartIndex,
				int32 InEndIndex,
				TArray<FStreamingRenderAsset>& InStreamingRenderAssets)
				: StartIndex(InStartIndex),
				EndIndex(InEndIndex),
				StreamingRenderAssets(InStreamingRenderAssets)
			{
			}
			int32 StartIndex;
			int32 EndIndex;
			TArray<FStreamingRenderAsset>& StreamingRenderAssets;
			TArray<int32> LocalInflightRenderAssets;
			TArray<UStreamableRenderAsset*> LocalDeferredTickCBAssets;
			char FalseSharingSpacerBuffer[PLATFORM_CACHE_LINE_SIZE];		// separate Packets to avoid false sharing
		};
		// In order to pass the InflightRenderAssets and DeferredTickCBAssets to each parallel for loop we need to break this into a number of workgroups.
		// Cannot be too large or the overhead of consolidating all the arrays will take too long.  Cannot be too small or the parallel for will not have
		// enough work.  Can be adjusted by CVarStreamingParallelRenderAssetsNumWorkgroups.
		int32 Num = EndIndex - StartIndex;
		int32 NumThreadTasks = FMath::Min<int32>(FTaskGraphInterface::Get().GetNumWorkerThreads() * GParallelRenderAssetsNumWorkgroups, Num - 1);
		TArray<FPacket> Packets;
		Packets.Reset(NumThreadTasks); // Go ahead and reserve space up front
		int32 Start = StartIndex;
		int32 NumRemaining = Num;
		int32 NumItemsPerGroup = Num / NumThreadTasks + 1;
		for (int32 i = 0; i < NumThreadTasks; ++i)
		{
			int32 NumAssetsToProcess = FMath::Min<int32>(NumRemaining, NumItemsPerGroup);
			Packets.Add(FPacket(Start, Start + NumAssetsToProcess, StreamingRenderAssets));
			Start += NumAssetsToProcess;
			NumRemaining -= NumAssetsToProcess;
			if (NumRemaining <= 0)
			{
				break;
			}
		}

		ParallelFor(Packets.Num(), [this, &Packets, &bWaitForMipFading, &bAsync](int32 PacketIndex)
		{
			for (int32 Index = Packets[PacketIndex].StartIndex; Index < Packets[PacketIndex].EndIndex; ++Index)
			{
				FStreamingRenderAsset& StreamingRenderAsset = Packets[PacketIndex].StreamingRenderAssets[Index];

				// Is this texture/mesh marked for removal? Will get cleanup once the async task is done.
				if (!StreamingRenderAsset.RenderAsset)
				{
					continue;
				}

				const int32* NumStreamedMips;
				const int32 NumLODGroups = GetNumStreamedMipsArray(StreamingRenderAsset.RenderAssetType, NumStreamedMips);

				StreamingRenderAsset.UpdateDynamicData(NumStreamedMips, NumLODGroups, Settings, bWaitForMipFading, &Packets[PacketIndex].LocalDeferredTickCBAssets); // We always use the Deferred CBs when doing the ParallelFor since those CBs are not thread safe.

				// Make a list of each texture/mesh that can potentially require additional UpdateStreamingStatus
				if (StreamingRenderAsset.RequestedMips != StreamingRenderAsset.ResidentMips)
				{
					Packets[PacketIndex].LocalInflightRenderAssets.Add(Index);
				}
			}
		});

		for (FPacket Packet : Packets) {
			InflightRenderAssets.Append(Packet.LocalInflightRenderAssets);
			DeferredTickCBAssets.Append(Packet.LocalDeferredTickCBAssets);
			Packet.LocalInflightRenderAssets.Empty();
			Packet.LocalDeferredTickCBAssets.Empty();
		}
		Packets.Empty();
	}
	else
#endif
	{
		for (int32 Index = StartIndex; Index < EndIndex; ++Index)
		{
			FStreamingRenderAsset& StreamingRenderAsset = StreamingRenderAssets[Index];
			FPlatformMisc::Prefetch(&StreamingRenderAsset + 1);

			// Is this texture/mesh marked for removal? Will get cleanup once the async task is done.
			if (!StreamingRenderAsset.RenderAsset) continue;

			STAT(int32 PreviousResidentMips = StreamingRenderAsset.ResidentMips;)

			const int32* NumStreamedMips;
			const int32 NumLODGroups = GetNumStreamedMipsArray(StreamingRenderAsset.RenderAssetType, NumStreamedMips);

			StreamingRenderAsset.UpdateDynamicData(NumStreamedMips, NumLODGroups, Settings, bWaitForMipFading, bAsync ? &DeferredTickCBAssets : nullptr);

			// Make a list of each texture/mesh that can potentially require additional UpdateStreamingStatus
			if (StreamingRenderAsset.RequestedMips != StreamingRenderAsset.ResidentMips)
			{
				InflightRenderAssets.Add(Index);
			}
	}
	CurrentUpdateStreamingRenderAssetIndex = EndIndex;
}
```

현재 스트리밍으로 디스크로부터 읽어오고 있는 텍스쳐의 수를 알고 싶다면 간단한 방법이 있다.          

```cpp
int32 FTextureStreamingNotificationImpl::GetNumStreamingTextures()
{
	FStreamingManagerCollection& StreamingManagerCollection = IStreamingManager::Get();

	if (StreamingManagerCollection.IsTextureStreamingEnabled())
	{
		IRenderAssetStreamingManager& TextureStreamingManager = StreamingManagerCollection.GetTextureStreamingManager();
		return TextureStreamingManager.GetNumWantingResources();
	}

	return 0;
}

/** Returns the number of resources that currently wants to be streamed in. */
virtual int32 IStreamingManager::GetNumWantingResources() const
{
	return NumWantingResources;
}

/**
 * Pure virtual base class of a streaming manager.
 */
struct IStreamingManager
{

	...
	...
	...

	/** Number of resources that currently wants to be streamed in. */
	int32		NumWantingResources;

	...
	...
	...
}
```