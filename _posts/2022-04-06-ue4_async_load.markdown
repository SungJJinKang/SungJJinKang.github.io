---
layout: post
title:  "언리얼 엔진4 Asynchronous 에셋 로드 코드 분석"
date:   2022-04-06
tags: [UE]
---

불필요한 코드 ( 여러 코멘트들, SHIPPING 빌드에서 빠지는 코드들, 디버깅용 코드들.... )들은 제거하였으니 풀 코드는 실제 소스 코드를 참고하세요.              
          
-----------------------------            
        
Asynchronous Loading Thread Enabled 옵션과 Event-Driven Loader Enabled 옵션을 킨 상태를 기준으로 한다.               
Asynchronous Loading Thread 옵션을 키면 ASync 동작을 게임 스레드와 별개의 스레드에서 수행할 수 있다. 게임 스레드dhk 동시에(!) 동작 할 수 있다는 것이다.                
다만 이로 인해 몇가지 제약이 생긴다.         
UE4 공식 문서에서는 아래와 같이 말한다.          

```
시리얼라이즈와 포스트 로딩 코드를 두 개의 별도 스레드에 동시 실행시키는 방식으로 작동하므로, 그에 따라 게임 코드의 UObject 클래스 생성자, PostInitProperties 함수, Serialize 함수는 반드시 스레드 안전성을 확보(thread-safe)한다.
```

-------------------------

이와 관련하여 좋은 자료 ( [https://zhuanlan.zhihu.com/p/357904199](https://zhuanlan.zhihu.com/p/357904199) )가 있어서 함께 첨부한다.           

-------------------------


FStreamableManager::RequestAsyncLoad 함수를 통해 에셋들에 대한 Async Load를 수행한다. return 되는 TSharedPtr<FStreamableHandle>를 통해 진행 중인 에셋 로딩과 관련된 데이터에 접근할 수 있다.        

```cpp
TSharedPtr<FStreamableHandle> FStreamableManager::RequestAsyncLoad(TArray<FSoftObjectPath> TargetsToStream, FStreamableDelegate DelegateToCall, TAsyncLoadPriority Priority, bool bManageActiveHandle, bool bStartStalled, FString DebugName)
{
    // ⭐ 
	// FStreamableHandle : 동기적, 비동기적 에셋 로드와 관련된 데이터를 가짐. 
	// 로드 후 호출 델리게이트, 로드 우선 순위, 로드 완료 여부, 등등 유저 코드에서 접근할 수 있는 에셋 로딩과 관련된 모든 데이터가 들어 있다.
	// ⭐               
	TSharedRef<FStreamableHandle> NewRequest = MakeShareable(new FStreamableHandle());
	NewRequest->CompleteDelegate = DelegateToCall;
	NewRequest->OwningManager = this;
	NewRequest->RequestedAssets = MoveTemp(TargetsToStream); // ⭐ 로드할 에셋들에 대한 경로를 FStreamableHandle에 저장. ⭐
	NewRequest->Priority = Priority;

	int32 NumValidRequests = NewRequest->RequestedAssets.Num();
	
	TSet<FSoftObjectPath> TargetSet;
	TargetSet.Reserve(NumValidRequests);

    // ⭐ 전달된 에셋 경로 중 유효한 것들만 걸러냄. ⭐
	for (const FSoftObjectPath& TargetName : NewRequest->RequestedAssets)
	{
		if (TargetName.IsNull())
		{
			--NumValidRequests;
			continue;
		}
		else if (FPackageName::IsShortPackageName(TargetName.GetAssetPathName()))
		{
			UE_LOG(LogStreamableManager, Error, TEXT("RequestAsyncLoad called with invalid package name %s"), *TargetName.ToString());
			NewRequest->CancelHandle();
			return nullptr;
		}
		TargetSet.Add(TargetName);
	}

	if (NumValidRequests == 0)
	{
		// Original array was empty or all null
		UE_LOG(LogStreamableManager, Error, TEXT("RequestAsyncLoad called with empty or only null assets!"));
		NewRequest->CancelHandle();
		return nullptr;
	} 
	else if (NewRequest->RequestedAssets.Num() != NumValidRequests)
	{
		FString RequestedSet;

		for (auto It = NewRequest->RequestedAssets.CreateIterator(); It; ++It)
		{
			FSoftObjectPath& Asset = *It;
			if (!RequestedSet.IsEmpty())
			{
				RequestedSet += TEXT(", ");
			}
			RequestedSet += Asset.ToString();

			// Remove null entries
			if (Asset.IsNull())
			{
				It.RemoveCurrent();
			}
		}

		// Some valid, some null
		UE_LOG(LogStreamableManager, Warning, TEXT("RequestAsyncLoad called with both valid and null assets, null assets removed from %s!"), *RequestedSet);
	}

	if (TargetSet.Num() != NewRequest->RequestedAssets.Num())
	{
		NewRequest->RequestedAssets = TargetSet.Array();
	}

	if (bManageActiveHandle)
	{
		// ⭐ 
		// 에셋이 로드 된 후에도 에셋 로드 Handle ( FStreamableHandle ) 에 대한 강참조를 가지고 있게 하여, 
		// FStreamableHandle가 RefCount이 되어 파괴되지 않도록 만듬. 
		// ⭐
		ManagedActiveHandles.Add(NewRequest);
	}

	if (bStartStalled)
	{
		// ⭐ 
		// bStartStalled true일시 ASyncLoad를 즉시 시작 안함. 
		// ⭐
		NewRequest->bStalled = true; 
	}
	else
	{
		StartHandleRequests(NewRequest);
	}

	return NewRequest;
}
```


```cpp
struct FStreamable
{
	// ⭐ 로드된 에셋 ( 로드 후 유효한 에셋이 셋팅된다. ) ⭐
	UObject* Target;
	
	// ⭐ 현재 로딩 중인지 ⭐
	bool	bAsyncLoadRequestOutstanding;

	// ⭐ 로드가 실패했는지 ⭐
	bool	bLoadFailed;

	// ⭐ 에셋이 로드되기를 기다리는 FStreamableHandle ⭐
	TArray< TSharedRef< FStreamableHandle> > LoadingHandles;

	/** List of handles that are keeping this streamable in memory. The same handle may be here multiple times */
	TArray< TWeakPtr< FStreamableHandle> > ActiveHandles;
}

void FStreamableManager::StartHandleRequests(TSharedRef<FStreamableHandle> Handle)
{
	TArray<FStreamable *> ExistingStreamables; // ⭐ ASyncLoad에 대한 요청들 ⭐
	ExistingStreamables.Reserve(Handle->RequestedAssets.Num());

	for (int32 i = 0; i < Handle->RequestedAssets.Num(); i++)
	{
		// ⭐ 
		// FStreamableHandle 내의 에셋 로드 요청들에 대해 ASyncLoad를 요청함. 
		// ⭐
		FStreamable* Existing = StreamInternal(Handle->RequestedAssets[i], Handle->Priority, Handle); 

		ExistingStreamables.Add(Existing);
		Existing->AddLoadingRequest(Handle); ASyncLoad 요청에 FStreamableHandle을 추가함.
	}

	for (int32 i = 0; i < Handle->RequestedAssets.Num(); i++)
	{
		FStreamable* Existing = ExistingStreamables[i];

		if (Existing && (Existing->Target || Existing->bLoadFailed))
		{
			// ⭐ 
			// 이미 에셋이 로드된 경우
			Existing->bAsyncLoadRequestOutstanding = false;
			// ⭐ 

			// ⭐ 
			// 매개변수로 전달된 Hand이 참조 중인 에셋 로드 요청들 중 
			// 이미 로드된 에셋에 대해서는 해당 에셋의 로드를 기다리고 있던 
			// 다른 Handle들까지 포함하여 에셋 로드 콜백을 날려줌. 
			// ⭐
			CheckCompletedRequests(Handle->RequestedAssets[i], Existing); 
		}
	}
}

FStreamable* FStreamableManager::StreamInternal(const FSoftObjectPath& InTargetName, TAsyncLoadPriority Priority, TSharedRef<FStreamableHandle> Handle)
{
	check(IsInGameThread());

	// ⭐ 
	// 리다이렉트된 경우를 대비하여 원래 경로를 찾아감. 
	// ⭐
	FSoftObjectPath TargetName = ResolveRedirects(InTargetName); 

	// ⭐ 
	// FStreamableManager에서 해당 에셋을 이미 로드 중인지를 확인 혹은 로드 되기 중인지를 확인. 
	// ⭐
	FStreamable* Existing = StreamableItems.FindRef(TargetName); 
	if (Existing)
	{
		// ⭐ 이미 에셋을 로드 중 ( 혹은 로드가 끝난 경우 )인 경우. ⭐

		if (Existing->bAsyncLoadRequestOutstanding)
		{
			// ⭐ 이미 현재 이 오브젝트가 Async 로딩 중인 경우 ⭐
			check(!Existing->Target); // should not be a load request unless the target is invalid
			ensure(IsAsyncLoading()); // Nothing should be pending if there is no async loading happening

			// Don't return as we potentially want to sync load it
		}
		if (Existing->Target)
		{
			// ⭐ 에셋이 이미 로드된 경우 ⭐
			return Existing;
		}
	}
	else
	{
		Existing = StreamableItems.Add(TargetName, new FStreamable());
	}

	if (!Existing->bAsyncLoadRequestOutstanding)
	{
		// ⭐ 에셋을 Async 로드 중이지 않은 경우 일단 메모리에서 에셋을 찾아봄 ⭐
		FindInMemory(TargetName, Existing);
	}
	
	if (!Existing->Target)
	{
		// ⭐ 에셋이 로드되어 있지 않은 경우. ⭐

		// Disable failed flag as it may have been added at a later point
		Existing->bLoadFailed = false;

		FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();

		// If async loading isn't safe or it's forced on, we have to do a sync load which will flush all async loading
		if (GIsInitialLoad || ThreadContext.IsInConstructor > 0 || bForceSynchronousLoads)
		{
			// ⭐ 
			// ASync 에셋 로딩이 위험한 경우 혹은 현재 함수가 클래스 생성자에서 호출된 경우, 혹은 Sync 로드가 강제된 경우. 
			// ⭐ 

			FRedirectedPath RedirectedPath;
			UE_LOG(LogStreamableManager, Verbose, TEXT("     Static loading %s"), *TargetName.ToString());
			// ⭐ 
			// Sync 로드를 수행 
			// ⭐
			Existing->Target = StaticLoadObject(UObject::StaticClass(), nullptr, *TargetName.ToString()); 

			// Need to manually detect redirectors because the above call only expects to load a UObject::StaticClass() type
			UObjectRedirector* Redir = Cast<UObjectRedirector>(Existing->Target);
			if (Redir)
			{
				TargetName = HandleLoadedRedirector(Redir, TargetName, Existing);
			}

			if (Existing->Target)
			{
				UE_LOG(LogStreamableManager, Verbose, TEXT("     Static loaded %s"), *Existing->Target->GetFullName());
			}
			else
			{
				Existing->bLoadFailed = true;
				UE_LOG(LogStreamableManager, Log, TEXT("Failed attempt to load %s"), *TargetName.ToString());
			}
			Existing->bAsyncLoadRequestOutstanding = false;
		}
		else
		{
			// We always queue a new request in case the existing one gets cancelled
			FString Package = TargetName.ToString();
			int32 FirstDot = Package.Find(TEXT("."), ESearchCase::CaseSensitive);
			if (FirstDot != INDEX_NONE)
			{
				Package.LeftInline(FirstDot,false);
			}

			Existing->bAsyncLoadRequestOutstanding = true;
			Existing->bLoadFailed = false;

			// ⭐ 
			// LoadPackageAsync로 패키지명, 로드 우선순위. 
			// 그리고 로드 완료시 호출될 멤버 함수 ( 위에서 본 CheckCompletedRequests 함수를 호출한다. )를 그 Handle 인스턴스와 함께 전달한다. 
			// ⭐
			int32 RequestId = LoadPackageAsync(Package, FLoadPackageAsyncDelegate::CreateSP(Handle, &FStreamableHandle::AsyncLoadCallbackWrapper, TargetName), Priority); 
		}
	}
	return Existing;
}
```

```cpp
int32 LoadPackageAsync(const FString& InName, const FGuid* InGuid /*= nullptr*/, const TCHAR* InPackageToLoadFrom /*= nullptr*/, FLoadPackageAsyncDelegate InCompletionDelegate /*= FLoadPackageAsyncDelegate()*/, EPackageFlags InPackageFlags /*= PKG_None*/, int32 InPIEInstanceID /*= INDEX_NONE*/, int32 InPackagePriority /*= 0*/, const FLinkerInstancingContext* InstancingContext /*=nullptr*/)
{
	LLM_SCOPE(ELLMTag::AsyncLoading);
	UE_CLOG(!GAsyncLoadingAllowed && !IsInAsyncLoadingThread(), LogStreaming, Fatal, TEXT("Requesting async load of \"%s\" when async loading is not allowed (after shutdown). Please fix higher level code."), *InName);
	
	// ⭐⭐⭐⭐⭐⭐⭐
	// AsyncPackageLoader에 LoadPackage를 요청
	return GetAsyncPackageLoader().LoadPackage(InName, InGuid, InPackageToLoadFrom, InCompletionDelegate, InPackageFlags, InPIEInstanceID, InPackagePriority, InstancingContext);
	// ⭐⭐⭐⭐⭐⭐⭐ 
}

int32 LoadPackageAsync(const FString& PackageName, FLoadPackageAsyncDelegate CompletionDelegate, int32 InPackagePriority /*= 0*/, EPackageFlags InPackageFlags /*= PKG_None*/, int32 InPIEInstanceID /*= INDEX_NONE*/)
{
	const FGuid* Guid = nullptr;
	const TCHAR* PackageToLoadFrom = nullptr;
	return LoadPackageAsync(PackageName, Guid, PackageToLoadFrom, CompletionDelegate, InPackageFlags, InPIEInstanceID, InPackagePriority );
}
```

----------------------

참고로 Sync Load시에도 내부적으로는 Async Load를 수행하고 그 Async Load를 Flush하는 방법으로 구현이 되어 있다.       
궁금하다면 아래의 함수들을 따라가 보기 바란다.            

```cpp          
StaticLoadObject(...) -> StaticLoadObjectInternal(...) -> LoadPackage(...) -> LoadPackageInternal(...) -> LoadPackageAsync(...)

UPackage* LoadPackageInternal(UPackage* InOuter, const TCHAR* InLongPackageNameOrFilename, uint32 LoadFlags, FLinkerLoad* ImportLinker, FArchive* InReaderOverride, const FLinkerInstancingContext* InstancingContext)
{
	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("LoadPackageInternal"), STAT_LoadPackageInternal, STATGROUP_ObjectVerbose);
	SCOPED_CUSTOM_LOADTIMER(LoadPackageInternal)
		ADD_CUSTOM_LOADTIMER_META(LoadPackageInternal, PackageName, InLongPackageNameOrFilename);

	checkf(IsInGameThread(), TEXT("Unable to load %s. Objects and Packages can only be loaded from the game thread."), InLongPackageNameOrFilename);

	UPackage* Result = nullptr;

// ⭐
// 기본적으로 true이다.
	if (
		(
			// ⭐
			// 대부분(윈도우, 안드로이드, IOS...)의 플랫폼에서 true
			FPlatformProperties::RequiresCookedData() && 
			// ⭐

			// ⭐
			// 엔진 기본 값은 true
			GEventDrivenLoaderEnabled && 
			// ⭐

			// ⭐
			// 항상 true
			EVENT_DRIVEN_ASYNC_LOAD_ACTIVE_AT_RUNTIME
			// ⭐
		)
// ⭐
#if WITH_IOSTORE_IN_EDITOR
		|| FIoDispatcher::IsInitialized()
#endif
		)
	{
		FString InName;
		FString InPackageName;

		if (FPackageName::IsPackageFilename(InLongPackageNameOrFilename))
		{
			FPackageName::TryConvertFilenameToLongPackageName(InLongPackageNameOrFilename, InPackageName);
		}
		else
		{
			InPackageName = InLongPackageNameOrFilename;
		}

		if (InOuter)
		{
			InName = InOuter->GetPathName();
		}
		else
		{
			InName = InPackageName;
		}

		FName PackageFName(*InPackageName);
#if WITH_IOSTORE_IN_EDITOR
		// Use the old loader if an uncooked package exists on disk
		const bool bDoesUncookedPackageExist = FPackageName::DoesPackageExist(InPackageName, nullptr, nullptr, true) && !DoesPackageExistInIoStore(FName(*InPackageName));
		if (!bDoesUncookedPackageExist)
#endif
		{
			if (FCoreDelegates::OnSyncLoadPackage.IsBound())
			{
				FCoreDelegates::OnSyncLoadPackage.Broadcast(InName);
			}

			// ⭐ 
			// 바로 위에서 확인하였다.
			// 아래에서 자세한 분석 예정
			int32 RequestID = LoadPackageAsync(InName, nullptr, *InPackageName);
			// ⭐ 

			if (RequestID != INDEX_NONE)
			{
				// ⭐
				// 현재 처리 대기 중인 모든 Async 로드 ( 기존에 처리되기를 기다리고 있던 것들을 모두 포함하여 )들이 완료되기를 기다리고 로드된 패키지들에 대한 후처리 동작 ( ex) PostLoad... )을 수행한다.
				// 즉 처리 대기 중인 모든 Async 로드들이 완료되기를 게임 스레드가 기다린다는 의미이다.
				//
				// FAsyncLoadingThread2::FlushLoading 함수 실행
				// 아래에서 자세한 분석 예정
				FlushAsyncLoading(RequestID);
				// ⭐
			}

			Result = (InOuter ? InOuter : FindObjectFast<UPackage>(nullptr, PackageFName));
			return Result;
		}
	}

	// ...
	// ...
	// ...

}
```

AsyncLoader는 버전 1과 버전 2, 두가지 버전이 있다.          
버전 2의 경우 UE4 4.24 버전부터 들어온 새로운 Async 로드 구현이다.       
기본 버전 1에 비해 로딩 속도에서의 개선이 있다.          
기본적으로 버전 2가 사용된다.                   
[참고 - IoStore](https://youtu.be/i31wSiqt-7w)            


```cpp
int32 FAsyncLoadingThread2::LoadPackage(const FString& InName, const FGuid* InGuid, const TCHAR* InPackageToLoadFrom, FLoadPackageAsyncDelegate InCompletionDelegate, EPackageFlags InPackageFlags, int32 InPIEInstanceID, int32 InPackagePriority, const FLinkerInstancingContext*)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(LoadPackage);

	LazyInitializeFromLoadPackage();

	int32 RequestID = INDEX_NONE;

	// happy path where all inputs are actual package names
	const FName Name = FName(*InName);
	FName DiskPackageName = InPackageToLoadFrom ? FName(InPackageToLoadFrom) : Name;
	bool bHasCustomPackageName = Name != DiskPackageName;

	// Verify PackageToLoadName, or fixup to handle any input string that can be converted to a long package name.
	FPackageId DiskPackageId = FPackageId::FromName(DiskPackageName);
	const FPackageStoreEntry* StoreEntry = GlobalPackageStore.FindStoreEntry(DiskPackageId);
	if (!StoreEntry)
	{
		FString PackageNameStr = DiskPackageName.ToString();
		if (!FPackageName::IsValidLongPackageName(PackageNameStr))
		{
			FString NewPackageNameStr;
			if (FPackageName::TryConvertFilenameToLongPackageName(PackageNameStr, NewPackageNameStr))
			{
				DiskPackageName = *NewPackageNameStr;
				DiskPackageId = FPackageId::FromName(DiskPackageName);
				StoreEntry = GlobalPackageStore.FindStoreEntry(DiskPackageId);
				bHasCustomPackageName &= Name != DiskPackageName;
			}
		}
	}

	// Verify CustomPackageName, or fixup to handle any input string that can be converted to a long package name.
	// CustomPackageName must not be an existing disk package name,
	// that could cause missing or incorrect import objects for other packages.
	FName CustomPackageName;
	FPackageId CustomPackageId;
	if (bHasCustomPackageName)
	{
		FPackageId PackageId = FPackageId::FromName(Name);
		if (!GlobalPackageStore.FindStoreEntry(PackageId))
		{
			FString PackageNameStr = Name.ToString();
			if (FPackageName::IsValidLongPackageName(PackageNameStr))
			{
				CustomPackageName = Name;
				CustomPackageId = PackageId;
			}
			else
			{
				FString NewPackageNameStr;
				if (FPackageName::TryConvertFilenameToLongPackageName(PackageNameStr, NewPackageNameStr))
				{
					PackageId = FPackageId::FromName(FName(*NewPackageNameStr));
					if (!GlobalPackageStore.FindStoreEntry(PackageId))
					{
						CustomPackageName = *NewPackageNameStr;
						CustomPackageId = PackageId;
					}
				}
			}
		}
	}
	// When explicitly requesting a redirected package then set CustomName to
	// the redirected name, otherwise the UPackage name will be set to the base game name.
	else if (GlobalPackageStore.IsRedirect(DiskPackageId))
	{
		bHasCustomPackageName = true;
		CustomPackageName = DiskPackageName;
		CustomPackageId = DiskPackageId;
	}

	check(CustomPackageId.IsValid() == !CustomPackageName.IsNone());

	bool bCustomNameIsValid = (!bHasCustomPackageName && CustomPackageName.IsNone()) || (bHasCustomPackageName && !CustomPackageName.IsNone());
	bool bDiskPackageIdIsValid = !!StoreEntry;
	if (!bDiskPackageIdIsValid)
	{
		// While there is an active load request for (InName=/Temp/PackageABC_abc, InPackageToLoadFrom=/Game/PackageABC), then allow these requests too:
		// (InName=/Temp/PackageA_abc, InPackageToLoadFrom=/Temp/PackageABC_abc) and (InName=/Temp/PackageABC_xyz, InPackageToLoadFrom=/Temp/PackageABC_abc)
		FAsyncPackage2* Package = GetAsyncPackage(DiskPackageId);
		if (Package)
		{
			if (CustomPackageName.IsNone())
			{
				CustomPackageName = Package->Desc.CustomPackageName;
				CustomPackageId = Package->Desc.CustomPackageId;
				bHasCustomPackageName = bCustomNameIsValid = true;
			}
			DiskPackageName = Package->Desc.DiskPackageName;
			DiskPackageId = Package->Desc.DiskPackageId;
			StoreEntry = Package->Desc.StoreEntry;
			bDiskPackageIdIsValid = true;
		}
	}

	if (bDiskPackageIdIsValid && bCustomNameIsValid)
	{
		if (FCoreDelegates::OnAsyncLoadPackage.IsBound())
		{
			FCoreDelegates::OnAsyncLoadPackage.Broadcast(InName);
		}

		// Generate new request ID and add it immediately to the global request list (it needs to be there before we exit
		// this function, otherwise it would be added when the packages are being processed on the async thread).
		RequestID = IAsyncPackageLoader::GetNextRequestId();
		TRACE_LOADTIME_BEGIN_REQUEST(RequestID);
		AddPendingRequest(RequestID);

		// Allocate delegate on Game Thread, it is not safe to copy delegates by value on other threads
		TUniquePtr<FLoadPackageAsyncDelegate> CompletionDelegatePtr;
		if (InCompletionDelegate.IsBound())
		{
			CompletionDelegatePtr.Reset(new FLoadPackageAsyncDelegate(InCompletionDelegate));
		}

#if !UE_BUILD_SHIPPING
		if (FileOpenLogWrapper)
		{
			FileOpenLogWrapper->AddPackageToOpenLog(*DiskPackageName.ToString());
		}
#endif

		// Add new package request

		// ⭐⭐⭐⭐⭐⭐⭐
		// FAsyncPackageDesc2를 만들고 Async 로드 대기 큐에 넣는다.
		FAsyncPackageDesc2 PackageDesc(RequestID, InPackagePriority, DiskPackageId, StoreEntry, DiskPackageName, CustomPackageId, CustomPackageName, MoveTemp(CompletionDelegatePtr));

		// Fixup for redirected packages since the slim StoreEntry itself has been stripped from both package names and package ids
		FPackageId RedirectedDiskPackageId = GlobalPackageStore.GetRedirectedPackageId(DiskPackageId);
		if (RedirectedDiskPackageId.IsValid())
		{
			PackageDesc.DiskPackageId = RedirectedDiskPackageId;
			PackageDesc.SourcePackageName = PackageDesc.DiskPackageName;
			PackageDesc.DiskPackageName = FName();
		}

		QueuePackage(PackageDesc);
		// ⭐⭐⭐⭐⭐⭐⭐

		UE_ASYNC_PACKAGE_LOG(Verbose, PackageDesc, TEXT("LoadPackage: QueuePackage"), TEXT("Package added to pending queue."));
	}
	else
	{
		static TSet<FName> SkippedPackages;
		bool bIsAlreadySkipped = false;
		if (!StoreEntry)
		{
			SkippedPackages.Add(DiskPackageName, &bIsAlreadySkipped);
			if (!bIsAlreadySkipped)
			{
				UE_LOG(LogStreaming, Warning,
					TEXT("LoadPackage: SkipPackage: %s (0x%llX) - The package to load does not exist on disk or in the loader"),
					*DiskPackageName.ToString(), FPackageId::FromName(DiskPackageName).ValueForDebugging());
			}
		}
		else
		{
			SkippedPackages.Add(Name, &bIsAlreadySkipped);
			if (!bIsAlreadySkipped)
			{
				UE_LOG(LogStreaming, Warning,
					TEXT("LoadPackage: SkipPackage: %s (0x%llX) - The package name is invalid"),
					*Name.ToString(), FPackageId::FromName(Name).ValueForDebugging());
			}
		}

		if (InCompletionDelegate.IsBound())
		{
			// Queue completion callback and execute at next process loaded packages call to maintain behavior compatibility with old loader
			FQueuedFailedPackageCallback& QueuedFailedPackageCallback = QueuedFailedPackageCallbacks.AddDefaulted_GetRef();
			QueuedFailedPackageCallback.PackageName = Name;
			QueuedFailedPackageCallback.Callback.Reset(new FLoadPackageAsyncDelegate(InCompletionDelegate));
		}
	}

	return RequestID;
}

void FAsyncLoadingThread2::QueuePackage(FAsyncPackageDesc2& Package)
{
	//TRACE_CPUPROFILER_EVENT_SCOPE(QueuePackage);
	UE_ASYNC_PACKAGE_DEBUG(Package);
	checkf(Package.StoreEntry, TEXT("No package store entry for package %s"), *Package.DiskPackageName.ToString());
	{
		FScopeLock QueueLock(&QueueCritical);
		++QueuedPackagesCounter;

		// ⭐
		QueuedPackages.Add(new FAsyncPackageDesc2(Package, MoveTemp(Package.PackageLoadedDelegate)));
		// ⭐
	}
	AltZenaphore.NotifyOne();
}

```

```cpp
// ⭐⭐⭐⭐⭐⭐⭐
// AsyncLoadingThread가 반복적으로 실행하는 함수이다. 
uint32 FAsyncLoadingThread2::Run()
// ⭐⭐⭐⭐⭐⭐⭐
{
	LLM_SCOPE(ELLMTag::AsyncLoading);

	AsyncLoadingThreadID = FPlatformTLS::GetCurrentThreadId();

	FAsyncLoadingThreadState2::Create(GraphAllocator, IoDispatcher);

	TRACE_LOADTIME_START_ASYNC_LOADING();

	FPlatformProcess::SetThreadAffinityMask(FPlatformAffinity::GetAsyncLoadingThreadMask());
	FMemory::SetupTLSCachesOnCurrentThread();

	FAsyncLoadingThreadState2& ThreadState = *FAsyncLoadingThreadState2::Get();
	
	FinalizeInitialLoad();

	FZenaphoreWaiter Waiter(AltZenaphore, TEXT("WaitForEvents"));
	bool bIsSuspended = false;
	while (!bStopRequested)
	{
		if (bIsSuspended)
		{
			if (!bSuspendRequested.Load(EMemoryOrder::SequentiallyConsistent) && !IsGarbageCollectionWaiting())
			{
				ThreadResumedEvent->Trigger();
				bIsSuspended = false;
				ResumeWorkers();
			}
			else
			{
				FPlatformProcess::Sleep(0.001f);
			}
		}
		else
		{
			bool bDidSomething = false;
			{
				FGCScopeGuard GCGuard;
				TRACE_CPUPROFILER_EVENT_SCOPE(AsyncLoadingTime);
				do
				{
					bDidSomething = false;

					if (QueuedPackagesCounter)
					{
						// ⭐⭐⭐⭐⭐⭐⭐
						if (CreateAsyncPackagesFromQueue(ThreadState))
						// ⭐⭐⭐⭐⭐⭐⭐
						{
							bDidSomething = true;
						}
					}

					bool bShouldSuspend = false;
					bool bPopped = false;
					do 
					{
						bPopped = false;
						for (FAsyncLoadEventQueue2* Queue : AltEventQueues)
						{
							if (Queue->PopAndExecute(ThreadState))
							{
								bPopped = true;
								bDidSomething = true;
							}

							if (bSuspendRequested.Load(EMemoryOrder::Relaxed) || IsGarbageCollectionWaiting())
							{
								bShouldSuspend = true;
								bPopped = false;
								break;
							}
						}
					} while (bPopped);

					if (bShouldSuspend || bSuspendRequested.Load(EMemoryOrder::Relaxed) || IsGarbageCollectionWaiting())
					{
						SuspendWorkers();
						ThreadSuspendedEvent->Trigger();
						bIsSuspended = true;
						bDidSomething = true;
						break;
					}

					{
						bool bDidExternalRead = false;
						do
						{
							bDidExternalRead = false;
							FAsyncPackage2* Package = nullptr;
							if (ExternalReadQueue.Peek(Package))
							{
								TRACE_CPUPROFILER_EVENT_SCOPE(ProcessExternalReads);

								FAsyncPackage2::EExternalReadAction Action = FAsyncPackage2::ExternalReadAction_Poll;

								EAsyncPackageState::Type Result = Package->ProcessExternalReads(Action);
								if (Result == EAsyncPackageState::Complete)
								{
									ExternalReadQueue.Pop();
									bDidExternalRead = true;
									bDidSomething = true;
								}
							}
						} while (bDidExternalRead);
					}

				} while (bDidSomething);
			}

			if (!bDidSomething)
			{
				if (ThreadState.HasDeferredFrees())
				{
					TRACE_CPUPROFILER_EVENT_SCOPE(AsyncLoadingTime);
					ThreadState.ProcessDeferredFrees();
					bDidSomething = true;
				}

				if (!DeferredDeletePackages.IsEmpty())
				{
					FGCScopeGuard GCGuard;
					TRACE_CPUPROFILER_EVENT_SCOPE(AsyncLoadingTime);
					FAsyncPackage2* Package = nullptr;
					int32 Count = 0;
					while (++Count <= 100 && DeferredDeletePackages.Dequeue(Package))
					{
						DeleteAsyncPackage(Package);
					}
					bDidSomething = true;
				}
			}

			if (!bDidSomething)
			{
				FAsyncPackage2* Package = nullptr;
				if (WaitingForIoBundleCounter.GetValue() > 0)
				{
					TRACE_CPUPROFILER_EVENT_SCOPE(AsyncLoadingTime);
					TRACE_CPUPROFILER_EVENT_SCOPE(WaitingForIo);
					Waiter.Wait();
				}
				else if (ExternalReadQueue.Peek(Package))
				{
					TRACE_CPUPROFILER_EVENT_SCOPE(AsyncLoadingTime);
					TRACE_CPUPROFILER_EVENT_SCOPE(ProcessExternalReads);

					EAsyncPackageState::Type Result = Package->ProcessExternalReads(FAsyncPackage2::ExternalReadAction_Wait);
					check(Result == EAsyncPackageState::Complete);
					ExternalReadQueue.Pop();
				}
				else
				{
					Waiter.Wait();
				}
			}
		}
	}
	return 0;
}

bool FAsyncLoadingThread2::CreateAsyncPackagesFromQueue(FAsyncLoadingThreadState2& ThreadState)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(CreateAsyncPackagesFromQueue);
	
	bool bPackagesCreated = false;
	const int32 TimeSliceGranularity = ThreadState.UseTimeLimit() ? 4 : MAX_int32;
	TArray<FAsyncPackageDesc2*> QueueCopy;

	do
	{
		{
			QueueCopy.Reset();
			FScopeLock QueueLock(&QueueCritical);

			const int32 NumPackagesToCopy = FMath::Min(TimeSliceGranularity, QueuedPackages.Num());
			if (NumPackagesToCopy > 0)
			{
				// ⭐
				QueueCopy.Append(QueuedPackages.GetData(), NumPackagesToCopy);
				QueuedPackages.RemoveAt(0, NumPackagesToCopy, false);
				// ⭐
			}
			else
			{
				break;
			}
		}

		for (FAsyncPackageDesc2* PackageDesc : QueueCopy)
		{
			bool bInserted;

			// ⭐⭐⭐⭐⭐⭐⭐
			// FAsyncPackage2를 만든다. 이미 존재하는 경우 기존의 것을 가져온다.
			// FAsyncPackage2는 패키지를 Async 로드하는데 필요한 중간 데이터들을 담은 구조체이다.
			FAsyncPackage2* Package = FindOrInsertPackage(PackageDesc, bInserted);
			// ⭐⭐⭐⭐⭐⭐⭐

			checkf(Package, TEXT("Failed to find or insert imported package %s"), *PackageDesc->DiskPackageName.ToString());

			// ⭐
			// FAsyncPackage2가 새로 만들어진 경우
			if (bInserted)
			// ⭐
			{
				UE_ASYNC_PACKAGE_LOG(Verbose, *PackageDesc, TEXT("CreateAsyncPackages: AddPackage"),
					TEXT("Start loading package."));
			}
			else
			{
				UE_ASYNC_PACKAGE_LOG_VERBOSE(Verbose, *PackageDesc, TEXT("CreateAsyncPackages: UpdatePackage"),
					TEXT("Package is alreay being loaded."));
			}

			--QueuedPackagesCounter;
			if (Package)
			{
				{
					TRACE_CPUPROFILER_EVENT_SCOPE(ImportPackages);
					Package->ImportPackagesRecursive();
				}

				if (bInserted)
				{
					// ⭐⭐⭐⭐⭐⭐⭐
					Package->StartLoading();
					// ⭐⭐⭐⭐⭐⭐⭐
				}

				// ⭐⭐⭐⭐⭐⭐⭐
				StartBundleIoRequests();
				// ⭐⭐⭐⭐⭐⭐⭐
			}
			delete PackageDesc;
		}

		bPackagesCreated |= QueueCopy.Num() > 0;
	} while (!ThreadState.IsTimeLimitExceeded(TEXT("CreateAsyncPackagesFromQueue")));

	return bPackagesCreated;
}

FAsyncPackage2* FAsyncLoadingThread2::FindOrInsertPackage(FAsyncPackageDesc2* Desc, bool& bInserted)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(FindOrInsertPackage);
	FAsyncPackage2* Package = nullptr;
	bInserted = false;
	{
		FScopeLock LockAsyncPackages(&AsyncPackagesCritical);
		Package = AsyncPackageLookup.FindRef(Desc->GetAsyncPackageId());
		if (!Package)
		{
			// ⭐⭐⭐⭐⭐⭐⭐
			Package = CreateAsyncPackage(*Desc);
			// ⭐⭐⭐⭐⭐⭐⭐

			checkf(Package, TEXT("Failed to create async package %s"), *Desc->DiskPackageName.ToString());
			Package->AddRef();
			ActiveAsyncPackagesCounter.Increment();
			AsyncPackageLookup.Add(Desc->GetAsyncPackageId(), Package);
			bInserted = true;
		}
		else
		{
			if (Desc->RequestID > 0)
			{
				Package->AddRequestID(Desc->RequestID);
			}
			if (Desc->Priority > Package->Desc.Priority)
			{
				UpdatePackagePriority(Package, Desc->Priority);
			}
		}
		if (Desc->PackageLoadedDelegate.IsValid())
		{
			Package->AddCompletionCallback(MoveTemp(Desc->PackageLoadedDelegate));
		}
	}
	return Package;
}

// ⭐⭐⭐⭐⭐⭐⭐
FAsyncPackage2* CreateAsyncPackage(const FAsyncPackageDesc2& Desc)
// ⭐⭐⭐⭐⭐⭐⭐
{
	TRACE_CPUPROFILER_EVENT_SCOPE(CreateAsyncPackage);
	UE_ASYNC_PACKAGE_DEBUG(Desc);
	checkf(Desc.StoreEntry, TEXT("No package store entry for package %s"), *Desc.DiskPackageName.ToString());

	FAsyncPackageData Data;
	Data.ExportCount = Desc.StoreEntry->ExportCount;
	Data.ExportBundleCount = Desc.StoreEntry->ExportBundleCount;

	const int32 ExportBundleNodeCount = Data.ExportBundleCount * EEventLoadNode2::ExportBundle_NumPhases;
	const int32 ImportedPackageCount = Desc.StoreEntry->ImportedPackages.Num();
	const int32 NodeCount = EEventLoadNode2::Package_NumPhases + ExportBundleNodeCount;

	const uint64 ExportBundleHeadersSize = sizeof(FExportBundleHeader) * Data.ExportBundleCount;
	const uint64 ExportBundleEntriesSize = sizeof(FExportBundleEntry) * Data.ExportCount * FExportBundleEntry::ExportCommandType_Count;
	Data.ExportBundlesMetaSize = ExportBundleHeadersSize + ExportBundleEntriesSize;

	const uint64 AsyncPackageMemSize = Align(sizeof(FAsyncPackage2), 8);
	const uint64 ExportBundlesMetaMemSize = Align(Data.ExportBundlesMetaSize, 8);
	const uint64 ExportsMemSize = Align(sizeof(FExportObject) * Data.ExportCount, 8);
	const uint64 ImportedPackagesMemSize = Align(sizeof(FAsyncPackage2*) * ImportedPackageCount, 8);
	const uint64 PackageNodesMemSize = Align(sizeof(FEventLoadNode2) * NodeCount, 8);
	const uint64 MemoryBufferSize =
		AsyncPackageMemSize +
		ExportBundlesMetaMemSize +
		ExportsMemSize +
		ImportedPackagesMemSize +
		PackageNodesMemSize;

	uint8* MemoryBuffer = reinterpret_cast<uint8*>(FMemory::Malloc(MemoryBufferSize));

	Data.ExportBundlesMetaMemory = MemoryBuffer + AsyncPackageMemSize;
	Data.ExportBundleHeaders = reinterpret_cast<const FExportBundleHeader*>(Data.ExportBundlesMetaMemory);
	Data.ExportBundleEntries = reinterpret_cast<const FExportBundleEntry*>(Data.ExportBundleHeaders + Data.ExportBundleCount);

	Data.Exports = MakeArrayView(reinterpret_cast<FExportObject*>(MemoryBuffer + AsyncPackageMemSize + ExportBundlesMetaMemSize), Data.ExportCount);
	Data.ImportedAsyncPackages = MakeArrayView(reinterpret_cast<FAsyncPackage2**>(MemoryBuffer + AsyncPackageMemSize + ExportBundlesMetaMemSize + ExportsMemSize), 0);
	Data.PackageNodes = MakeArrayView(reinterpret_cast<FEventLoadNode2*>(MemoryBuffer + AsyncPackageMemSize + ExportBundlesMetaMemSize + ExportsMemSize + ImportedPackagesMemSize), NodeCount);
	Data.ExportBundleNodes = MakeArrayView(&Data.PackageNodes[EEventLoadNode2::Package_NumPhases], ExportBundleNodeCount);

	ExistingAsyncPackagesCounter.Increment();

	// ⭐
	return new (MemoryBuffer) FAsyncPackage2(Desc, Data, *this, GraphAllocator, EventSpecs.GetData());
	// ⭐
}

void FAsyncPackage2::StartLoading()
{
	TRACE_CPUPROFILER_EVENT_SCOPE(StartLoading);
	TRACE_LOADTIME_BEGIN_LOAD_ASYNC_PACKAGE(this);
	check(AsyncPackageLoadingState == EAsyncPackageLoadingState2::ImportPackagesDone);

	LoadStartTime = FPlatformTime::Seconds();

	// ⭐
	AsyncLoadingThread.AddBundleIoRequest(this);
	// ⭐
	AsyncPackageLoadingState = EAsyncPackageLoadingState2::WaitingForIo;
}

void FAsyncLoadingThread2::AddBundleIoRequest(FAsyncPackage2* Package)
{
	WaitingForIoBundleCounter.Increment();
	// ⭐⭐⭐⭐⭐⭐⭐
	// FAsyncPackage를 IO 요청 대기 큐에 넣는다.
	WaitingIoRequests.HeapPush({ Package });
	// ⭐⭐⭐⭐⭐⭐⭐
}

void FAsyncLoadingThread2::StartBundleIoRequests()
{
	TRACE_CPUPROFILER_EVENT_SCOPE(StartBundleIoRequests);
	constexpr uint64 MaxPendingRequestsSize = 256 << 20;

	// ⭐
	// 동기화를 위해 IO 요청들을 묶기 위해 사용하는 클래스이다.
	FIoBatch IoBatch = IoDispatcher.NewBatch();
	// ⭐

	while (WaitingIoRequests.Num())
	{
		FBundleIoRequest& BundleIoRequest = WaitingIoRequests.HeapTop();
		FAsyncPackage2* Package = BundleIoRequest.Package;
		if (PendingBundleIoRequestsTotalSize > 0 && PendingBundleIoRequestsTotalSize + Package->ExportBundlesSize > MaxPendingRequestsSize)
		{
			break;
		}
		PendingBundleIoRequestsTotalSize += Package->ExportBundlesSize;
		WaitingIoRequests.HeapPop(BundleIoRequest, false);

		FIoReadOptions ReadOptions;

		// ⭐⭐⭐⭐⭐⭐⭐
		// FIoBatch를 통해 패키지에 대한 IO를 수행한다.
		//
		//
		// struct FAsyncPackageDesc2
		// {
		//		...
		//		...
		//		...
		//
		// 		The package store entry with meta data about the actual disk package
		//		// 실제 디스크에 존재하는 패키지에 대한 메타데이터를 가지고 있는 구조체
		// 		const FPackageStoreEntry* StoreEntry;
		//
		// 		The disk package id corresponding to the StoreEntry
		// 		It is used by the loader for io chunks and to handle ref tracking of loaded packages and import objects.
		//		// StoreEntry와 대응되는 디스크 패키지 ID
		//		// 
		// 		FPackageId DiskPackageId; 
		//   	
		//		...
		//		...
		//		...
		//	}
		//
		Package->IoRequest = IoBatch.ReadWithCallback(CreateIoChunkId(Package->Desc.DiskPackageId.Value(), 0, EIoChunkType::ExportBundleData),
			ReadOptions,
			Package->Desc.Priority,
			[Package](TIoStatusOr<FIoBuffer> Result)
		{
			if (Result.IsOk())
			{
				Package->IoBuffer = Result.ConsumeValueOrDie();
			}
			else
			{
				UE_ASYNC_PACKAGE_LOG(Warning, Package->Desc, TEXT("StartBundleIoRequests: FailedRead"),
					TEXT("Failed reading chunk for package: %s"), *Result.Status().ToString());
				Package->bLoadHasFailed = true;
			}
			Package->AsyncLoadingThread.WaitingForIoBundleCounter.Decrement();

			// ⭐
			// FAsyncPackage2::Event_ProcessPackageSummary 함수를 수행한다.
			// 디스크에서 읽어온 패키지 데이터를 가지고, 언리얼에서 사용할 UPackage 오브젝트를 생성한다.
			Package->GetPackageNode(EEventLoadNode2::Package_ProcessSummary).ReleaseBarrier();
			// ⭐

		});
		// ⭐⭐⭐⭐⭐⭐⭐

		TRACE_COUNTER_DECREMENT(PendingBundleIoRequests);
	}
	IoBatch.Issue();
}
```

```cpp
EAsyncPackageState::Type FAsyncPackage2::Event_ProcessPackageSummary(FAsyncLoadingThreadState2& ThreadState, FAsyncPackage2* Package, int32)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(Event_ProcessPackageSummary);
	UE_ASYNC_PACKAGE_DEBUG(Package->Desc);
	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::WaitingForIo);
	Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::ProcessPackageSummary;

	FScopedAsyncPackageEvent2 Scope(Package);

	if (Package->bLoadHasFailed)
	{
		if (Package->Desc.CanBeImported())
		{
			FLoadedPackageRef* PackageRef = Package->ImportStore.GlobalPackageStore.LoadedPackageStore.FindPackageRef(Package->Desc.DiskPackageId);
			check(PackageRef);
			PackageRef->SetHasFailed();
		}
	}
	else
	{
		check(Package->ExportBundleEntryIndex == 0);

		const uint8* PackageSummaryData = Package->IoBuffer.Data();
		const FPackageSummary* PackageSummary = reinterpret_cast<const FPackageSummary*>(PackageSummaryData);
		const uint8* GraphData = PackageSummaryData + PackageSummary->GraphDataOffset;
		const uint64 PackageSummarySize = GraphData + PackageSummary->GraphDataSize - PackageSummaryData;

		if (PackageSummary->NameMapNamesSize)
		{
			TRACE_CPUPROFILER_EVENT_SCOPE(LoadPackageNameMap);
			const uint8* NameMapNamesData = PackageSummaryData + PackageSummary->NameMapNamesOffset;
			const uint8* NameMapHashesData = PackageSummaryData + PackageSummary->NameMapHashesOffset;
			Package->NameMap.Load(
				TArrayView<const uint8>(NameMapNamesData, PackageSummary->NameMapNamesSize),
				TArrayView<const uint8>(NameMapHashesData, PackageSummary->NameMapHashesSize),
				FMappedName::EType::Package);
		}

		{
			FName PackageName = Package->NameMap.GetName(PackageSummary->Name);
			// Don't apply any redirects in editor builds
#if !WITH_IOSTORE_IN_EDITOR
			if (PackageSummary->SourceName != PackageSummary->Name)
			{
				FName SourcePackageName = Package->NameMap.GetName(PackageSummary->SourceName);
				Package->Desc.SetDiskPackageName(PackageName, SourcePackageName);
			}
			else
#endif
			{
				Package->Desc.SetDiskPackageName(PackageName);
			}
		}

		Package->CookedHeaderSize = PackageSummary->CookedHeaderSize;
		Package->ImportStore.ImportMap = TArrayView<const FPackageObjectIndex>(
				reinterpret_cast<const FPackageObjectIndex*>(PackageSummaryData + PackageSummary->ImportMapOffset),
				(PackageSummary->ExportMapOffset - PackageSummary->ImportMapOffset) / sizeof(FPackageObjectIndex));
		Package->ExportMap = reinterpret_cast<const FExportMapEntry*>(PackageSummaryData + PackageSummary->ExportMapOffset);
		
		FMemory::Memcpy(Package->Data.ExportBundlesMetaMemory, PackageSummaryData + PackageSummary->ExportBundlesOffset, Package->Data.ExportBundlesMetaSize);

		// ⭐⭐⭐⭐⭐⭐⭐
		Package->CreateUPackage(PackageSummary);
		// ⭐⭐⭐⭐⭐⭐⭐

		Package->SetupSerializedArcs(GraphData, PackageSummary->GraphDataSize);

		Package->AllExportDataPtr = PackageSummaryData + PackageSummarySize;
		Package->CurrentExportDataPtr = Package->AllExportDataPtr;

		TRACE_LOADTIME_PACKAGE_SUMMARY(Package, PackageSummarySize, Package->ImportStore.ImportMap.Num(), Package->Data.ExportCount);
	}

	if (GIsInitialLoad)
	{
		Package->SetupScriptDependencies();
	}

	// ⭐
	// FAsyncPackage2::Event_ProcessExportBundle 동작을 수행한다.
	Package->GetExportBundleNode(ExportBundle_Process, 0).ReleaseBarrier();
	// ⭐

	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::ProcessPackageSummary);
	Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::ProcessExportBundles;
	return EAsyncPackageState::Complete;
}

EAsyncPackageState::Type FAsyncPackage2::Event_ProcessExportBundle(FAsyncLoadingThreadState2& ThreadState, FAsyncPackage2* Package, int32 ExportBundleIndex)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(Event_ProcessExportBundle);
	UE_ASYNC_PACKAGE_DEBUG(Package->Desc);
	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::ProcessExportBundles);

	FScopedAsyncPackageEvent2 Scope(Package);

	auto FilterExport = [](const EExportFilterFlags FilterFlags) -> bool
	{
#if UE_SERVER
		return !!(static_cast<uint32>(FilterFlags) & static_cast<uint32>(EExportFilterFlags::NotForServer));
#elif !WITH_SERVER_CODE
		return !!(static_cast<uint32>(FilterFlags) & static_cast<uint32>(EExportFilterFlags::NotForClient));
#else
		static const bool bIsDedicatedServer = !GIsClient && GIsServer;
		static const bool bIsClientOnly = GIsClient && !GIsServer;

		if (bIsDedicatedServer && static_cast<uint32>(FilterFlags) & static_cast<uint32>(EExportFilterFlags::NotForServer))
		{
			return true;
		}

		if (bIsClientOnly && static_cast<uint32>(FilterFlags) & static_cast<uint32>(EExportFilterFlags::NotForClient))
		{
			return true;
		}

		return false;
#endif
	};

	check(ExportBundleIndex < Package->Data.ExportBundleCount);
	
	if (!Package->bLoadHasFailed)
	{
		const uint64 AllExportDataSize = Package->IoBuffer.DataSize() - (Package->AllExportDataPtr - Package->IoBuffer.Data());
		FExportArchive Ar(Package->AllExportDataPtr, Package->CurrentExportDataPtr, AllExportDataSize);
		{
			Ar.SetUE4Ver(Package->LinkerRoot->LinkerPackageVersion);
			Ar.SetLicenseeUE4Ver(Package->LinkerRoot->LinkerLicenseeVersion);
			// Ar.SetEngineVer(Summary.SavedByEngineVersion); // very old versioning scheme
			// Ar.SetCustomVersions(LinkerRoot->LinkerCustomVersion); // only if not cooking with -unversioned
			Ar.SetUseUnversionedPropertySerialization((Package->LinkerRoot->GetPackageFlags() & PKG_UnversionedProperties) != 0);
			Ar.SetIsLoading(true);
			Ar.SetIsPersistent(true);
			if (Package->LinkerRoot->GetPackageFlags() & PKG_FilterEditorOnly)
			{
				Ar.SetFilterEditorOnly(true);
			}
			Ar.ArAllowLazyLoading = true;

			// FExportArchive special fields
			Ar.CookedHeaderSize = Package->CookedHeaderSize;
			Ar.PackageDesc = &Package->Desc;
			Ar.NameMap = &Package->NameMap;
			Ar.ImportStore = &Package->ImportStore;
			Ar.Exports = Package->Data.Exports;
			Ar.ExportMap = Package->ExportMap;
			Ar.ExternalReadDependencies = &Package->ExternalReadDependencies;
		}
		const FExportBundleHeader* ExportBundle = Package->Data.ExportBundleHeaders + ExportBundleIndex;
		const FExportBundleEntry* BundleEntries = Package->Data.ExportBundleEntries + ExportBundle->FirstEntryIndex;
		const FExportBundleEntry* BundleEntry = BundleEntries + Package->ExportBundleEntryIndex;
		const FExportBundleEntry* BundleEntryEnd = BundleEntries + ExportBundle->EntryCount;
		check(BundleEntry <= BundleEntryEnd);
		while (BundleEntry < BundleEntryEnd)
		{
			if (ThreadState.IsTimeLimitExceeded(TEXT("Event_ProcessExportBundle")))
			{
				return EAsyncPackageState::TimeOut;
			}
			const FExportMapEntry& ExportMapEntry = Package->ExportMap[BundleEntry->LocalExportIndex];
			FExportObject& Export = Package->Data.Exports[BundleEntry->LocalExportIndex];
			Export.bFiltered = FilterExport(ExportMapEntry.FilterFlags);

			if (BundleEntry->CommandType == FExportBundleEntry::ExportCommandType_Create)
			{
				Package->EventDrivenCreateExport(BundleEntry->LocalExportIndex);
			}
			else
			{
				check(BundleEntry->CommandType == FExportBundleEntry::ExportCommandType_Serialize);

				const uint64 CookedSerialSize = ExportMapEntry.CookedSerialSize;
				UObject* Object = Export.Object;

				check(Package->CurrentExportDataPtr + CookedSerialSize <= Package->IoBuffer.Data() + Package->IoBuffer.DataSize());
				check(Object || Export.bFiltered || Export.bExportLoadFailed);

				Ar.ExportBufferBegin(Object, ExportMapEntry.CookedSerialOffset, ExportMapEntry.CookedSerialSize);

				const int64 Pos = Ar.Tell();
				UE_ASYNC_PACKAGE_CLOG(
					CookedSerialSize > uint64(Ar.TotalSize() - Pos), Fatal, Package->Desc, TEXT("ObjectSerializationError"),
					TEXT("%s: Serial size mismatch: Expected read size %d, Remaining archive size: %d"),
					Object ? *Object->GetFullName() : TEXT("null"), CookedSerialSize, uint64(Ar.TotalSize() - Pos));

				const bool bSerialized = Package->EventDrivenSerializeExport(BundleEntry->LocalExportIndex, Ar);
				if (!bSerialized)
				{
					Ar.Skip(CookedSerialSize);
				}
				UE_ASYNC_PACKAGE_CLOG(
					CookedSerialSize != uint64(Ar.Tell() - Pos), Fatal, Package->Desc, TEXT("ObjectSerializationError"),
					TEXT("%s: Serial size mismatch: Expected read size %d, Actual read size %d"),
					Object ? *Object->GetFullName() : TEXT("null"), CookedSerialSize, uint64(Ar.Tell() - Pos));

				Ar.ExportBufferEnd();

				check((Object && !Object->HasAnyFlags(RF_NeedLoad)) || Export.bFiltered || Export.bExportLoadFailed);

				Package->CurrentExportDataPtr += CookedSerialSize;
			}
			++BundleEntry;
			++Package->ExportBundleEntryIndex;
		}
	}
	
	Package->ExportBundleEntryIndex = 0;

	if (ExportBundleIndex + 1 < Package->Data.ExportBundleCount)
	{
		Package->GetExportBundleNode(ExportBundle_Process, ExportBundleIndex + 1).ReleaseBarrier();
	}
	else
	{
		Package->ImportStore.ImportMap = TArrayView<const FPackageObjectIndex>();
		Package->IoBuffer = FIoBuffer();

		if (Package->ExternalReadDependencies.Num() == 0)
		{
			check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::ProcessExportBundles);
			Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::ExportsDone;

			// ⭐
			// 현재 패키지에 대한 로드가 끝났다.
			// FAsyncPackage2::Event_ExportsDone 동작을 수행한다.
			Package->GetPackageNode(Package_ExportsSerialized).ReleaseBarrier(&ThreadState);
			// ⭐
		}
		else
		{
			// ⭐
			// 현재 패키지에 대해 종속성을 가진 패키지가 있는 경우 그것들을 먼저 처리한다.
			check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::ProcessExportBundles);
			Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::WaitingForExternalReads;
			Package->AsyncLoadingThread.ExternalReadQueue.Enqueue(Package);
			// ⭐
		}
	}

	if (ExportBundleIndex == 0)
	{
		Package->AsyncLoadingThread.BundleIoRequestCompleted(Package);
	}

	return EAsyncPackageState::Complete;
}

EAsyncPackageState::Type FAsyncPackage2::Event_ExportsDone(FAsyncLoadingThreadState2& ThreadState, FAsyncPackage2* Package, int32)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(Event_ExportsDone);
	UE_ASYNC_PACKAGE_DEBUG(Package->Desc);
	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::ExportsDone);

	if (!Package->bLoadHasFailed && Package->Desc.CanBeImported())
	{
		FLoadedPackageRef& PackageRef =
			Package->AsyncLoadingThread.GlobalPackageStore.LoadedPackageStore.GetPackageRef((Package->Desc.DiskPackageId));
		PackageRef.SetAllPublicExportsLoaded();
	}

	Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::PostLoad;

	// ⭐
	Package->GetExportBundleNode(EEventLoadNode2::ExportBundle_PostLoad, 0).ReleaseBarrier();
	// ⭐

	return EAsyncPackageState::Complete;
}

EAsyncPackageState::Type FAsyncPackage2::Event_PostLoadExportBundle(FAsyncLoadingThreadState2& ThreadState, FAsyncPackage2* Package, int32 ExportBundleIndex)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(Event_PostLoad);
	UE_ASYNC_PACKAGE_DEBUG(Package->Desc);
	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::PostLoad);
	check(Package->ExternalReadDependencies.Num() == 0);
	
	FAsyncPackageScope2 PackageScope(Package);

	check(ExportBundleIndex < Package->Data.ExportBundleCount);

	EAsyncPackageState::Type LoadingState = EAsyncPackageState::Complete;

	if (!Package->bLoadHasFailed)
	{
		// Begin async loading, simulates BeginLoad
		Package->BeginAsyncLoad();

		SCOPED_LOADTIMER(PostLoadObjectsTime);

		FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();
		TGuardValue<bool> GuardIsRoutingPostLoad(ThreadContext.IsRoutingPostLoad, true);

		const bool bAsyncPostLoadEnabled = FAsyncLoadingThreadSettings::Get().bAsyncPostLoadEnabled;
		const bool bIsMultithreaded = Package->AsyncLoadingThread.IsMultithreaded();

		const FExportBundleHeader* ExportBundle = Package->Data.ExportBundleHeaders + ExportBundleIndex;
		const FExportBundleEntry* BundleEntries = Package->Data.ExportBundleEntries + ExportBundle->FirstEntryIndex;
		const FExportBundleEntry* BundleEntry = BundleEntries + Package->ExportBundleEntryIndex;
		const FExportBundleEntry* BundleEntryEnd = BundleEntries + ExportBundle->EntryCount;
		check(BundleEntry <= BundleEntryEnd);
		while (BundleEntry < BundleEntryEnd)
		{
			if (ThreadState.IsTimeLimitExceeded(TEXT("Event_PostLoadExportBundle")))
			{
				LoadingState = EAsyncPackageState::TimeOut;
				break;
			}
			
			if (BundleEntry->CommandType == FExportBundleEntry::ExportCommandType_Serialize)
			{
				do
				{
					FExportObject& Export = Package->Data.Exports[BundleEntry->LocalExportIndex];
					if (Export.bFiltered | Export.bExportLoadFailed)
					{
						break;
					}

					UObject* Object = Export.Object;
					check(Object);
					check(!Object->HasAnyFlags(RF_NeedLoad));
					if (!Object->HasAnyFlags(RF_NeedPostLoad))
					{
						break;
					}

					check(Object->IsReadyForAsyncPostLoad());

					// ⭐
					// Async Load Thread가 Disabled되어 있어 현재 코드가 게임 스레드에서 수행되는 경우 ( 게임 스레드에서 수행되니, PostLoad해도 안전 ),
					// 혹은 Async Load Thread가 Enabled되고 PostLoad 동작을 Async로 수행하는 옵션이 Enabled되어 있는 경우 ( AsyncPostLoad 옵션이 켜져 있는 경우 )에 수행된다.
					//
					// 그 외의 경우에는 아래 FAsyncPackage2::Event_DeferredPostLoadExportBundle에서 게임 스레드가 PostLoad를 수행한다.
					if (!bIsMultithreaded || (bAsyncPostLoadEnabled && CanPostLoadOnAsyncLoadingThread(Object)))
					{
						ThreadContext.CurrentlyPostLoadedObjectByALT = Object;
						{
							TRACE_LOADTIME_POSTLOAD_EXPORT_SCOPE(Object);
							Object->ConditionalPostLoad();
						}
						ThreadContext.CurrentlyPostLoadedObjectByALT = nullptr;
					}
					// ⭐

				} while (false);
			}
			++BundleEntry;
			++Package->ExportBundleEntryIndex;
		}

		// ⭐
		// End async loading, simulates EndLoad
		Package->EndAsyncLoad();
		// ⭐

	}
	
	if (LoadingState == EAsyncPackageState::TimeOut)
	{
		return LoadingState;
	}

	Package->ExportBundleEntryIndex = 0;

	if (ExportBundleIndex + 1 < Package->Data.ExportBundleCount)
	{
		Package->GetExportBundleNode(ExportBundle_PostLoad, ExportBundleIndex + 1).ReleaseBarrier();
	}
	else
	{
		if (Package->LinkerRoot && !Package->bLoadHasFailed)
		{
			UE_ASYNC_PACKAGE_LOG(Verbose, Package->Desc, TEXT("AsyncThread: FullyLoaded"),
				TEXT("Async loading of package is done, and UPackage is marked as fully loaded."));
			// mimic old loader behavior for now, but this is more correctly also done in FinishUPackage
			// called from ProcessLoadedPackagesFromGameThread just before complection callbacks
			Package->LinkerRoot->MarkAsFullyLoaded();
		}

		check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::PostLoad);
		Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::DeferredPostLoad;

		// ⭐
		// DeferredPostLoad 단계는 메인 스레드 ( 게임 스레드 )에서 수행된다.
		// FAsyncPackage2::Event_DeferredPostLoadExportBundle
		Package->GetExportBundleNode(ExportBundle_DeferredPostLoad, 0).ReleaseBarrier();
		// ⭐
	}

	return EAsyncPackageState::Complete;
}



EAsyncPackageState::Type FAsyncPackage2::Event_DeferredPostLoadExportBundle(FAsyncLoadingThreadState2& ThreadState, FAsyncPackage2* Package, int32 ExportBundleIndex)
{
	SCOPE_CYCLE_COUNTER(STAT_FAsyncPackage_PostLoadObjectsGameThread);
	TRACE_CPUPROFILER_EVENT_SCOPE(Event_DeferredPostLoad);
	UE_ASYNC_PACKAGE_DEBUG(Package->Desc);
	check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::DeferredPostLoad);

	FAsyncPackageScope2 PackageScope(Package);

	check(ExportBundleIndex < Package->Data.ExportBundleCount);
	EAsyncPackageState::Type LoadingState = EAsyncPackageState::Complete;

	if (Package->bLoadHasFailed)
	{
		FSoftObjectPath::InvalidateTag();
		FUniqueObjectGuid::InvalidateTag();
	}
	else
	{
		TGuardValue<bool> GuardIsRoutingPostLoad(PackageScope.ThreadContext.IsRoutingPostLoad, true);
		FAsyncLoadingTickScope2 InAsyncLoadingTick(Package->AsyncLoadingThread);

		const FExportBundleHeader* ExportBundle = Package->Data.ExportBundleHeaders + ExportBundleIndex;
		const FExportBundleEntry* BundleEntries = Package->Data.ExportBundleEntries + ExportBundle->FirstEntryIndex;
		const FExportBundleEntry* BundleEntry = BundleEntries + Package->ExportBundleEntryIndex;
		const FExportBundleEntry* BundleEntryEnd = BundleEntries + ExportBundle->EntryCount;
		check(BundleEntry <= BundleEntryEnd);
		while (BundleEntry < BundleEntryEnd)
		{
			if (ThreadState.IsTimeLimitExceeded(TEXT("Event_DeferredPostLoadExportBundle")))
			{
				LoadingState = EAsyncPackageState::TimeOut;
				break;
			}

			if (BundleEntry->CommandType == FExportBundleEntry::ExportCommandType_Serialize)
			{
				do
				{
					FExportObject& Export = Package->Data.Exports[BundleEntry->LocalExportIndex];
					if (Export.bFiltered | Export.bExportLoadFailed)
					{
						break;
					}

					UObject* Object = Export.Object;
					check(Object);
					check(!Object->HasAnyFlags(RF_NeedLoad));
					if (Object->HasAnyFlags(RF_NeedPostLoad))
					{
						PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = Object;
						{
							TRACE_LOADTIME_POSTLOAD_EXPORT_SCOPE(Object);
							FScopeCycleCounterUObject ConstructorScope(Object, GET_STATID(STAT_FAsyncPackage_PostLoadObjectsGameThread));

							// ⭐⭐⭐⭐⭐⭐⭐
							// 로드된 오브젝트에 대해 PostLoad 동작을 수행한다.
							Object->ConditionalPostLoad();
							// ⭐⭐⭐⭐⭐⭐⭐
						}
						PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = nullptr;
					}
				} while (false);
			}
			++BundleEntry;
			++Package->ExportBundleEntryIndex;
		}
	}

	if (LoadingState == EAsyncPackageState::TimeOut)
	{
		return LoadingState;
	}

	Package->ExportBundleEntryIndex = 0;

	if (ExportBundleIndex + 1 < Package->Data.ExportBundleCount)
	{
		Package->GetExportBundleNode(ExportBundle_DeferredPostLoad, ExportBundleIndex + 1).ReleaseBarrier();
	}
	else
	{
		check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState2::DeferredPostLoad);
		Package->AsyncPackageLoadingState = EAsyncPackageLoadingState2::DeferredPostLoadDone;
		Package->AsyncLoadingThread.LoadedPackagesToProcess.Add(Package);
	}

	return EAsyncPackageState::Complete;
}
```



references : [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/)      