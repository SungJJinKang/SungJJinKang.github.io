---
layout: post
title:  "UE4 Asynchronous 에셋 로드 코드 분석 ( 작성 중 )"
date:   2022-04-06
categories: UE4 UnrealEngine4 ComputerScience
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
			// ⭐ 이미 에셋이 로드된 경우 ⭐
			Existing->bAsyncLoadRequestOutstanding = false;

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

AsyncLoader는 버전 1과 버전 2, 두가지 버전이 있다.          
왜 버전 2가 나왔는지는 모르겠다. 구글에 검색해보아도 찾을 수 없었다.           
필자의 경우 UE 4.25.4 버전을 기준으로 에디터 상에서 버전 1로 작동되는 것을 확인하였기 때문에, 버전 1을 기준으로 코드를 살펴보겠다.           

이후 찾아본 바로는 버전 2의 경우 UE4 4.24 버전부터 들어온 새로운 Async 로드 구현이다.       
기본 버전 1에 비해 로딩 속도에서의 개선이 있다.          
기본적으로 버전 2가 사용되고,                
IoStore를 사용하지 않는 경우와 에디터에서만 기존 버전 1이 그대로 사용된다.                  
[참고 - IoStore](https://youtu.be/i31wSiqt-7w)            


```cpp
int32 LoadPackageAsync(const FString& InName, const FGuid* InGuid /*= nullptr*/, const TCHAR* InPackageToLoadFrom /*= nullptr*/, FLoadPackageAsyncDelegate InCompletionDelegate /*= FLoadPackageAsyncDelegate()*/, EPackageFlags InPackageFlags /*= PKG_None*/, int32 InPIEInstanceID /*= INDEX_NONE*/, int32 InPackagePriority /*= 0*/, const FLinkerInstancingContext* InstancingContext /*=nullptr*/)
{
	LLM_SCOPE(ELLMTag::AsyncLoading);
	UE_CLOG(!GAsyncLoadingAllowed && !IsInAsyncLoadingThread(), LogStreaming, Fatal, TEXT("Requesting async load of \"%s\" when async loading is not allowed (after shutdown). Please fix higher level code."), *InName);
	// ⭐ 
	// AsyncPackageLoader에 패키지 로드를 요청, FAsyncLoadingThread::LoadPackage 호출 
	// ⭐
	return GetAsyncPackageLoader().LoadPackage(InName, InGuid, InPackageToLoadFrom, InCompletionDelegate, InPackageFlags, InPIEInstanceID, InPackagePriority, InstancingContext); 
}
```

```cpp
int32 FAsyncLoadingThread::LoadPackage(const FString& InName, const FGuid* InGuid, const TCHAR* InPackageToLoadFrom, FLoadPackageAsyncDelegate InCompletionDelegate, EPackageFlags InPackageFlags, int32 InPIEInstanceID, int32 InPackagePriority, const FLinkerInstancingContext* InstancingContext)
{
	int32 RequestID = INDEX_NONE;

	// ⭐ 
	// 매개변수로 패키지명이 전달되는 것이 정상이지만, 
	// 파일명이 전달되었을 경우도 패키지명을 자동으로 찾아준다. 
	// ⭐
	FString PackageName;
	bool bValidPackageName = true;

	if (FPackageName::IsValidLongPackageName(InName, /*bIncludeReadOnlyRoots*/true))
	{
		PackageName = InName;
	}
	// PackageName got populated by the conditional function
	else if (!(FPackageName::IsPackageFilename(InName) && FPackageName::TryConvertFilenameToLongPackageName(InName, PackageName)))
	{
		// PackageName may get populated by the conditional function
		FString ClassName;

		if (!FPackageName::ParseExportTextPath(PackageName, &ClassName, &PackageName))
		{
			bValidPackageName = false;
		}
	}

	// ⭐ 
	// 로드할 패키지 명 
	// ⭐
	FString PackageNameToLoad(InPackageToLoadFrom); 

	if (bValidPackageName)
	{
		if (PackageNameToLoad.IsEmpty())
		{
			PackageNameToLoad = PackageName;
		}
		// Make sure long package name is passed to FAsyncPackage so that it doesn't attempt to 
		// create a package with short name.
		if (FPackageName::IsShortPackageName(PackageNameToLoad))
		{
			UE_LOG(LogStreaming, Warning, TEXT("Async loading code requires long package names (%s)."), *PackageNameToLoad);

			bValidPackageName = false;
		}
	}

	if (bValidPackageName)
	{
		// Generate new request ID and add it immediately to the global request list (it needs to be there before we exit
		// this function, otherwise it would be added when the packages are being processed on the async thread).
		// ⭐ 
		// 패키지 로더로부터 새로운 로드 요청 ID를 발급받는다. 에셋 로드 시작 전 로드 요청 ID를 미리 발급받아두어야한다. 
		// ⭐        
		RequestID = IAsyncPackageLoader::GetNextRequestId(); 
		AddPendingRequest(RequestID);

		// Allocate delegate on Game Thread, it is not safe to copy delegates by value on other threads
		TUniquePtr<FLoadPackageAsyncDelegate> CompletionDelegatePtr;
		if (InCompletionDelegate.IsBound())
		{
			// ⭐ 
			// 게임 스레드에서 로드 완료시 호출될 델리게이트를 생성한다. 
			// ⭐
			CompletionDelegatePtr = MakeUnique<FLoadPackageAsyncDelegate>(MoveTemp(InCompletionDelegate)); 
		}

		// ⭐ 
		// ASync 패키지 로드에 대한 정보를 담은 FAsyncPackageDesc 생성. 
		// 로드 요청 ID, 패키지명, 패키지 로드 완료시 호출될 델리게이트, 로드 우선 순위 등등 
		// ASync 스레드가 패키지를 ASync하게 로드하기 위한 각종 데이터가 담겨있다. 
		// ⭐         
		FAsyncPackageDesc PackageDesc(RequestID, *PackageName, *PackageNameToLoad, InGuid ? *InGuid : FGuid(), MoveTemp(CompletionDelegatePtr), InPackageFlags, InPIEInstanceID, InPackagePriority);
		QueuePackage(PackageDesc); // 패키지를 로드 큐에 넣는다.      
	}
	else
	{
		// ⭐ 패키지명이 유효하지 않은 경우. 무시하자. ⭐
		InCompletionDelegate.ExecuteIfBound(FName(*InName), nullptr, EAsyncLoadingResult::Failed);
	}

	return RequestID;
}
```

```cpp
void FAsyncLoadingThread::QueuePackage(FAsyncPackageDesc& Package)
{
	// ⭐ 
	// 당연한 이야기지만 게임 스레드 ( 현재 이 함수를 호출하는 스레드 ), ASyncLoad 스레드간의 DataRace를 방지하기 위해 
	// 큐에 FAsyncPackageDesc 삽입하기 전 뮤텍스 락을 건다. 
	// ⭐
	FScopeLock QueueLock(&QueueCritical); 
	QueuedPackagesCounter.Increment();
	QueuedPackages.Add(new FAsyncPackageDesc(Package, MoveTemp(Package.PackageLoadedDelegate)));
	QueuedRequestsEvent->Trigger();
}
```

자 여기까지가 게임스레드의 역할이다. 이제 ASync 로드의 운명은 ASync 스레드로 넘어갔다.          

```cpp
/**
 * // ⭐ 
   // ASync 로딩 스레드에 대한 추상화 클래스이다. 
   // 패키지에 대한 Preloads, Serializes는 ASync 스레드에서 호출되고, 패키지에 대한 PostLoads는 게임 스레드에서 호출된다. 
   // ⭐
 */
class FAsyncLoadingThread final : public FRunnable, public IAsyncPackageLoader
```

FAsyncLoadingThread에서는 게임 스레드에서 접근할 수 있는 기능들과, ASync 스레드에서 사용하는 기능들을 둘다 가지고 있다.        

어차피 우리는 ASync 스레드가 ASync를 어떻게 수행하는지가 궁금하니 ASync 스레드에서 도는 부분들만 집중적으로 보겠다.          

언리얼상에서 어떤 특정 스레드에서 동작하기 위해 만들어진 클래스들은 모두 FRunnable을 상속받는다. 해당 특정 스레드는 맡고 있는 FRunnable 오브젝트에 대해 FRunnable::Run 함수를 반복적으로 호출한다.          

```cpp
uint32 FAsyncLoadingThread::Run()
{
	// ⭐ 생략 시킨 코드가 많다.... ⭐

	if (bWasSuspendedLastFrame)
	{
		bWasSuspendedLastFrame = false;
		ThreadResumedEvent->Trigger();
	}
	if (!IsGarbageCollectionWaiting())
	{
		// ⭐ 
		// 가비지 컬렉터가 돌고 있지 않은 경우, TickAsyncThread를 호출한다. 
		// ⭐
		bool bDidSomething = false;
		TickAsyncThread(true, false, 0.033f, bDidSomething);
	}
	return 0;
}
```

```cpp
EAsyncPackageState::Type FAsyncLoadingThread::TickAsyncThread(bool bUseTimeLimit, bool bUseFullTimeLimit, float TimeLimit, bool& bDidSomething, FFlushTree* FlushTree)
{
	EAsyncPackageState::Type Result = EAsyncPackageState::Complete;
	if (!bShouldCancelLoading)
	{
		int32 ProcessedRequests = 0;
		double TickStartTime = FPlatformTime::Seconds();
		if (AsyncThreadReady.GetValue())
		{
			{
				FGCScopeGuard GCGuard;
				
				// ⭐
				// 밑에 코드를 참고하라. 
				// 위에서 게임스레드에서 FAsyncLoadingThread::QueuedPackages에 
				// 추가한 FAsyncPackageDesc를 ASync Load용 큐 ( FAsyncLoadingThread::AsyncPackages )로 옮긴다.
				// 이 함수는 동작 제한 시간이 있기 때문에 제한된 시간 이상으로 실행 시간을 잡아먹으면 반환된다.
				// ⭐
				CreateAsyncPackagesFromQueue(bUseTimeLimit, bUseFullTimeLimit, TimeLimit, FlushTree); 
			}

			float TimeUsed = (float)(FPlatformTime::Seconds() - TickStartTime);
			const float RemainingTimeLimit = FMath::Max(0.0f, TimeLimit - TimeUsed);
			if (IsGarbageCollectionWaiting() || (RemainingTimeLimit <= 0.0f && bUseTimeLimit && !IsMultithreaded()))
			{
				// ⭐ 
				// 가비지 컬렉팅 중이거나, CreateAsyncPackagesFromQueue 함수가 너무 오랫동안 돈 경우,
				// 이번 틱에서는 ASync 처리를 하지 못한다. 
				// ⭐
				Result = EAsyncPackageState::TimeOut; 
			}
			else
			{
				// ⭐ 이 경우 ASyncLoading 처리 동작을 수행한다. ⭐

				FGCScopeGuard GCGuard;

				// ⭐ 밑에 코드를 참고하라. ⭐
				Result = ProcessAsyncLoading(ProcessedRequests, bUseTimeLimit, bUseFullTimeLimit, RemainingTimeLimit, FlushTree);
				bDidSomething = bDidSomething || ProcessedRequests > 0;
			}
		}

		if (ProcessedRequests == 0 && IsMultithreaded() && Result == EAsyncPackageState::Complete)
		{
			uint32 WaitTime = 30;
			if (IsEventDrivenLoaderEnabled())
			{
				if (!EDLBootNotificationManager.IsWaitingForSomething() && !(IsGarbageCollectionWaiting() || IsGarbageCollecting()))
				{
					CheckForCycles();
					IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CheckForCycles (non-shipping)"));
				}
				else
				{
					WaitTime = 1; // we are waiting for the game thread to deal with the boot constructors, so lets spin tighter
				}
			}
			const bool bIgnoreThreadIdleStats = true;
			SCOPED_LOADTIMER(Package_Temp3);
			QueuedRequestsEvent->Wait(WaitTime, bIgnoreThreadIdleStats);
		}

	}
	else
	{
		// Blocks main thread
		double TickStartTime = FPlatformTime::Seconds();
		CancelAsyncLoadingInternal();
		IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CancelAsyncLoadingInternal"));
		bShouldCancelLoading = false;
	}

	return Result;
}
```

위에서 게임스레드에서 FAsyncLoadingThread::QueuedPackages에 추가한 FAsyncPackageDesc를 ASync Load용 큐 ( FAsyncLoadingThread::AsyncPackages )로 옮긴다.       

```cpp
int32 FAsyncLoadingThread::CreateAsyncPackagesFromQueue(bool bUseTimeLimit, bool bUseFullTimeLimit, float TimeLimit, FFlushTree* FlushTree)
{
	FAsyncLoadingTickScope InAsyncLoadingTick(*this);

	int32 NumCreated = 0;
	checkSlow(IsInAsyncLoadThread());

	int32 TimeSliceGranularity = 1; // do 4 packages at a time

	TArray<FAsyncPackageDesc*> QueueCopy;
	double TickStartTime = FPlatformTime::Seconds();
	do
	{
		{
			FScopeLock QueueLock(&QueueCritical);

			QueueCopy.Reset();
			QueueCopy.Reserve(FMath::Min(TimeSliceGranularity, QueuedPackages.Num()));

			int32 NumCopied = 0;

			for (FAsyncPackageDesc* PackageRequest : QueuedPackages)
			{
				if (NumCopied < TimeSliceGranularity)
				{
					NumCopied++;
					// ⭐ 
					// 한 루프마다 한 FAsyncPackageDesc씩 ASync Load용 큐로 옮기는 것을 시도한다. 
					// ⭐
					QueueCopy.Add(PackageRequest); 
				}
				else
				{
					break;
				}
			}
			if (NumCopied)
			{
				QueuedPackages.RemoveAt(0, NumCopied, false);
			}
			else
			{
				break;
			}
		}

		if (QueueCopy.Num() > 0)
		{
			double Timer = 0;
			{
				SCOPE_SECONDS_COUNTER(Timer);
				for (FAsyncPackageDesc* PackageRequest : QueueCopy)
				{
					ProcessAsyncPackageRequest(PackageRequest, nullptr, FlushTree);
					delete PackageRequest;
				}
			}
		}

		NumCreated += QueueCopy.Num();
	} while(!IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CreateAsyncPackagesFromQueue"))); 
	// ⭐
	// QueuedPackages에 있는 ASync Load 요청을 너무 오랬동안 처리하는 것을 막기 위해 일정한 시간 이상 처리를 했으면 종료한다. 
	// ( 여기서 말하는 처리란 ASync 로딩을 수행하는 동작이 아니라, QueuedPacakages 변수에 있는 ASync 로드 요청을 FAsyncLoadingThread::AsyncPackages 변수로 옮기는 과정을 말한다. 바로 밑에서 확인할 수 있다. )
	// ⭐

	return NumCreated;
}

void FAsyncLoadingThread::ProcessAsyncPackageRequest(FAsyncPackageDesc* InRequest, FAsyncPackage* InRootPackage, FFlushTree* FlushTree)
{
	// ⭐ 
	// 혹시 로드하려는 패키지가 이미 로드 중인 경우 전달된 로드 완료 콜백을 기존 로드에 추가한다. 
	// ⭐
	FAsyncPackage* Package = FindExistingPackageAndAddCompletionCallback(InRequest, AsyncPackageNameLookup, FlushTree); 

	if (Package)
	{
		// ⭐ 
		// 로드하려는 패키지를 이미 로드 중인 경우, 
		// 기존 로드 요청의 Priority와 로드하려고 전달된 FAsyncPackageDesc의 Priority를 비교하여 더 높은 Priority를 전달한다. 
		// ⭐
		UpdateExistingPackagePriorities(Package, InRequest->Priority);
	}
	else
	{
		// [BLOCKING] LoadedPackages are accessed on the main thread too, so lock to be able to add a completion callback
		FScopeLock LoadedLock(&LoadedPackagesCritical);
		Package = FindExistingPackageAndAddCompletionCallback(InRequest, LoadedPackagesNameLookup, FlushTree);
	}

	if (!Package)
	{
		// [BLOCKING] LoadedPackagesToProcess are modified on the main thread, so lock to be able to add a completion callback
		FScopeLock LoadedLock(&LoadedPackagesToProcessCritical);
		Package = FindExistingPackageAndAddCompletionCallback(InRequest, LoadedPackagesToProcessNameLookup, FlushTree);
	}

	if (!Package)
	{
		// New package that needs to be loaded or a package has already been loaded long time ago
		{
			// GC can't run in here
			FGCScopeGuard GCGuard;
			// ⭐ 
			// 로드를 위한 FAsyncPackage를 생성한다. 
			// ⭐
			Package = new FAsyncPackage(*this, *InRequest, EDLBootNotificationManager); 
		}
		if (InRequest->PackageLoadedDelegate.IsValid())
		{
			const bool bInternalCallback = false;
			Package->AddCompletionCallback(MoveTemp(InRequest->PackageLoadedDelegate), bInternalCallback);
		}

		// ⭐ 
		// FAsyncPackage의 Root FAsyncPackage를 셋팅한다. 
		// 로드하려는 패키지가 다른 패키지에 Dependency를 가진 경우 해당 Dependency FAsyncPackage를 셋팅한다. 
		// ⭐
		Package->SetDependencyRootPackage(InRootPackage);

		if (FlushTree)
		{
			Package->PopulateFlushTree(FlushTree);
		}

		// Add to queue according to priority.
		// ⭐ FAsyncPackage를 ASync 로드될 FAsyncLoadingThread::AsyncPackages 리스트 변수에 넣는다. ⭐
		InsertPackage(Package, false, EAsyncPackageInsertMode::InsertAfterMatchingPriorities); 

		// For all other cases this is handled in FindExistingPackageAndAddCompletionCallback
		const int32 QueuedPackagesCount = QueuedPackagesCounter.Decrement();
		NotifyAsyncLoadingStateHasMaybeChanged();
		check(QueuedPackagesCount >= 0);
	}
}
```

```cpp
void FAsyncLoadingThread::InsertPackage(FAsyncPackage* Package, bool bReinsert, EAsyncPackageInsertMode InsertMode)
{
	{
#if THREADSAFE_UOBJECTS
		FScopeLock LockAsyncPackages(&AsyncPackagesCritical);
#endif
		if (bReinsert)
		{
			AsyncPackages.RemoveSingle(Package);
		}

		if (GEventDrivenLoaderEnabled)
		{
			AsyncPackages.Add(Package);
		}
		else
		{
			...
		}

		if (!bReinsert)
		{
			AsyncPackageNameLookup.Add(Package->GetPackageName(), Package);
			if (GEventDrivenLoaderEnabled)
			{
				// @todo If this is a reinsert for some priority thing, well we don't go back and retract the stuff in flight to adjust the priority of events
				// ⭐ 
				// 이 부분이 중요하다! 
				// ⭐
				QueueEvent_CreateLinker(Package, FAsyncLoadEvent::EventSystemPriority_MAX); 
			}
		}
	}
	check(!GEventDrivenLoaderEnabled || GetPackage(WeakPtr) == Package);
}
```
             
이 함수 내부에서 프로젝트 셋팅 중 "Event Driven Loader" 셋팅에 따라 분기가 나뉘는데, "Event Driven Loader" 셋팅이 기본적으로 Enabled되어 있으니 Enabled되어 있는 경우를 기준으로 코드를 설명하겠다.        
사실 "Event Driven Loader" ( EDL )이 무엇인지 모르겠다. ( 나중에 조사를 더 해보겠다. ) ( UE4에서는 대부분의 경우 EDL 옵션을 켜둔 경우)          

```cpp
EAsyncPackageState::Type FAsyncLoadingThread::ProcessAsyncLoading(int32& OutPackagesProcessed, bool bUseTimeLimit /*= false*/, bool bUseFullTimeLimit /*= false*/, float TimeLimit /*= 0.0f*/, FFlushTree* FlushTree)
{
	double TickStartTime = FPlatformTime::Seconds();

	FAsyncLoadingTickScope InAsyncLoadingTick(*this);
	uint32 LoopIterations = 0;

	if (GEventDrivenLoaderEnabled)
	{
		while (true)
		{
			{
				const float RemainingTimeLimit = FMath::Max(0.0f, TimeLimit - (float)(FPlatformTime::Seconds() - TickStartTime));
				// ⭐ 
				// 위에서 본 함수이다. 여기서도 다시 한번 호출해준다. 
				// ⭐
				int32 NumCreated = CreateAsyncPackagesFromQueue(bUseTimeLimit, bUseFullTimeLimit, RemainingTimeLimit); 
				OutPackagesProcessed += NumCreated;
				bDidSomething = NumCreated > 0 || bDidSomething;
				if (IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("CreateAsyncPackagesFromQueue")))
				{
					return EAsyncPackageState::TimeOut;
				}
			}
			if (bDidSomething)
			{
				continue;
			}

			{
				FAsyncLoadEventArgs Args;
				Args.bUseTimeLimit = bUseTimeLimit;
				Args.TickStartTime = TickStartTime;
				Args.TimeLimit = TimeLimit;

				Args.OutLastTypeOfWorkPerformed = nullptr;
				Args.OutLastObjectWorkWasPerformedOn = nullptr;
				 // ⭐ 
				 // EDL 기반 이벤트를 처리해준다. 
				 // ⭐
				if (EventQueue.PopAndExecute(Args))
				{
					OutPackagesProcessed++;
					if (IsTimeLimitExceeded(Args.TickStartTime, Args.bUseTimeLimit, Args.TimeLimit, Args.OutLastTypeOfWorkPerformed, Args.OutLastObjectWorkWasPerformedOn))
					{
						return EAsyncPackageState::TimeOut;
					}
					bDidSomething = true;
				}
			}
			if (bDidSomething)
			{
				continue;
			}
			if (AsyncPackagesReadyForTick.Num())
			{
				// ⭐ 이 Scope가 우리가 관심이 있는 ASync 로드 부분이다. ⭐

				OutPackagesProcessed++;
				bDidSomething = true;
				FAsyncPackage* Package = AsyncPackagesReadyForTick[0];
				check(Package->AsyncPackageLoadingState == EAsyncPackageLoadingState::PostLoad_Etc);
				SCOPED_LOADTIMER(ProcessAsyncLoadingTime);

				EAsyncPackageState::Type LocalLoadingState = EAsyncPackageState::Complete;
				if (Package->HasFinishedLoading() == false)
				{
					float RemainingTimeLimit = FMath::Max(0.0f, TimeLimit - (float)(FPlatformTime::Seconds() - TickStartTime));

					// ⭐⭐⭐⭐⭐⭐⭐ 
					// ASync 로드할 패키지에 대한 TickAsyncPackage를 호출해준다.
					// ⭐⭐⭐⭐⭐⭐⭐    
					LocalLoadingState = Package->TickAsyncPackage(bUseTimeLimit, bUseFullTimeLimit, RemainingTimeLimit, FlushTree); 

					if (LocalLoadingState == EAsyncPackageState::TimeOut)
					{
						if (IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("TickAsyncPackage")))
						{
							return EAsyncPackageState::TimeOut;
						}
						UE_LOG(LogStreaming, Error, TEXT("Should not have a timeout when the time limit is not exceeded."));
						continue;
					}
				}
				else
				{
					check(0); // if it has finished loading, it should not be in AsyncPackagesReadyForTick
				}
				if (LocalLoadingState == EAsyncPackageState::Complete)
				{
					// ⭐ 패키지를 완전히 로드한 경우 ⭐
					{
	#if THREADSAFE_UOBJECTS
						FScopeLock LockAsyncPackages(&AsyncPackagesCritical);
	#endif
						AsyncPackageNameLookup.Remove(Package->GetPackageName());
						int32 PackageIndex = AsyncPackages.Find(Package);
						AsyncPackages.RemoveAt(PackageIndex);
						AsyncPackagesReadyForTick.RemoveAt(0, 1, false); //@todoio this should maybe be a heap or something to avoid the removal cost
					}

					// We're done, at least on this thread, so we can remove the package now.
					// ⭐ 
					// 로드 된 패키지에 대해서는 게임 스레드에서 PostLoad를 수행해야하기 때문에 
					// 우선 로드된 패키지를 LoadedPackages로 옮김. 
					// ⭐
					AddToLoadedPackages(Package); 
				}
				if (IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("TickAsyncPackage")))
				{
					return EAsyncPackageState::TimeOut;
				}
			}
			if (bDidSomething)
			{
				continue;
			}
			bool bAnyIOOutstanding = GetPrecacheHandler().AnyIOOutstanding();
			if (bAnyIOOutstanding)
			{
				SCOPED_LOADTIMER(Package_EventIOWait);
				double StartTime = FPlatformTime::Seconds();
				if (bUseTimeLimit)
				{
					if (bUseFullTimeLimit)
					{
						const float RemainingTimeLimit = FMath::Max(0.0f, TimeLimit - (float)(FPlatformTime::Seconds() - TickStartTime));
						if (RemainingTimeLimit > 0.0f)
						{
							bool bGotIO = GetPrecacheHandler().WaitForIO(RemainingTimeLimit);
							if (bGotIO)
							{
								OutPackagesProcessed++;
								continue; // we got some IO, so start processing at the top
							}
							{
								float ThisTime = float(FPlatformTime::Seconds() - StartTime);
							}
						}
					}
					//UE_LOG(LogStreaming, Error, TEXT("Not using full time limit, waiting for IO..."));
					return EAsyncPackageState::TimeOut;
				}
				else
				{
					bool bGotIO = GetPrecacheHandler().WaitForIO(10.0f); // wait "forever"
					if (!bGotIO)
					{
						//UE_LOG(LogStreaming, Error, TEXT("Waited for 10 seconds on IO...."));
						FPlatformMisc::LowLevelOutputDebugString(TEXT("Waited for 10 seconds on IO...."));
					}
					{
						OutPackagesProcessed++;
					}
				}
			}
			else
			{
				LoadingState = EAsyncPackageState::Complete;
				break;
			}
		}
	}
	return LoadingState;
}
```

```cpp
// ⭐⭐⭐⭐⭐⭐⭐ 
EAsyncPackageState::Type FAsyncPackage::TickAsyncPackage(bool InbUseTimeLimit, bool InbUseFullTimeLimit, float& InOutTimeLimit, FFlushTree* FlushTree)
// ⭐⭐⭐⭐⭐⭐⭐ 
{
	// Whether we should execute the next step.
	EAsyncPackageState::Type LoadingState = EAsyncPackageState::Complete;

	// Set up tick relevant variables.
	bUseTimeLimit = InbUseTimeLimit;
	bUseFullTimeLimit = InbUseFullTimeLimit;
	bTimeLimitExceeded = false;
	TimeLimit = InOutTimeLimit;
	TickStartTime = FPlatformTime::Seconds();

	// Keep track of time when we start loading.
	if (LoadStartTime == 0.0)
	{
		LoadStartTime = TickStartTime;

		// If we are a dependency of another package, we need to tell that package when its first dependent started loading,
		// otherwise because that package loads last it'll not include the entire load time of all its dependencies
		if (DependencyRootPackage)
		{
			// Only the first dependent needs to register the start time
			if (DependencyRootPackage->GetLoadStartTime() == 0.0)
			{
				DependencyRootPackage->LoadStartTime = TickStartTime;
			}
		}
	}

	FAsyncPackageScope PackageScope(this);

	// Make sure we finish our work if there's no time limit. The loop is required as PostLoad
	// might cause more objects to be loaded in which case we need to Preload them again.
	do
	{
		// Reset value to true at beginning of loop.
		LoadingState = EAsyncPackageState::Complete;

		// Begin async loading, simulates BeginLoad
		BeginAsyncLoad();

		// We have begun loading a package that we know the name of. Let the package time tracker know.
		FExclusiveLoadPackageTimeTracker::PushLoadPackage(Desc.NameToLoad);

		if (LoadingState == EAsyncPackageState::Complete && !bLoadHasFailed)
		{
			SCOPED_LOADTIMER(Package_ExternalReadDependencies);
			LoadingState = FinishExternalReadDependencies();
		}

		// Call PostLoad on objects, this could cause new objects to be loaded that require
		// another iteration of the PreLoad loop.
		if (LoadingState == EAsyncPackageState::Complete && !bLoadHasFailed)
		{
			SCOPED_LOADTIMER(Package_PostLoadObjects);
			LoadingState = PostLoadObjects();
		}

		// We are done loading the package for now. Whether it is done or not, let the package time tracker know.
		FExclusiveLoadPackageTimeTracker::PopLoadPackage(Linker ? Linker->LinkerRoot : nullptr);

		// End async loading, simulates EndLoad
		EndAsyncLoad();

		// Finish objects (removing EInternalObjectFlags::AsyncLoading, dissociate imports and forced exports, 
		// call completion callback, ...
		// If the load has failed, perform completion callbacks and then quit
		if (LoadingState == EAsyncPackageState::Complete || bLoadHasFailed)
		{
			LoadingState = FinishObjects();
		}
	} while (!IsTimeLimitExceeded() && LoadingState == EAsyncPackageState::TimeOut);

	if (LinkerRoot && LoadingState == EAsyncPackageState::Complete)
	{
		LinkerRoot->MarkAsFullyLoaded();
	}

	// We can't have a reference to a UObject.
	LastObjectWorkWasPerformedOn = nullptr;
	// Reset type of work performed.
	LastTypeOfWorkPerformed = nullptr;
	// Mark this package as loaded if everything completed.
	bLoadHasFinished = (LoadingState == EAsyncPackageState::Complete);

	if (bLoadHasFinished && GEventDrivenLoaderEnabled)
	{
		check(AsyncPackageLoadingState == EAsyncPackageLoadingState::PostLoad_Etc);
		AsyncPackageLoadingState = EAsyncPackageLoadingState::PackageComplete;
	}

	// Subtract the time it took to load this package from the global limit.
	InOutTimeLimit = (float)FMath::Max(0.0, InOutTimeLimit - (FPlatformTime::Seconds() - TickStartTime));

	ReentryCount--;
	check(ReentryCount >= 0);

	// true means that we're done loading this package.
	return LoadingState;
}
```

**ASync 스레드**는 로딩이 끝난 오브젝트들에 대해 PostLoad를 호출해준다.

```cpp
EAsyncPackageState::Type FAsyncPackage::PostLoadObjects()
{
	SCOPE_CYCLE_COUNTER(STAT_FAsyncPackage_PostLoadObjects);

	SCOPED_LOADTIMER(PostLoadObjectsTime);

	// GC can't run in here
	FGCScopeGuard GCGuard;

	FUObjectThreadContext& ThreadContext = FUObjectThreadContext::Get();
	TGuardValue<bool> GuardIsRoutingPostLoad(ThreadContext.IsRoutingPostLoad, true);

	FUObjectSerializeContext* LoadContext = GetSerializeContext();
	TArray<UObject*>& ThreadObjLoaded = LoadContext->PRIVATE_GetObjectsLoadedInternalUseOnly();
	if (ThreadObjLoaded.Num())
	{
		// New objects have been loaded. They need to go through PreLoad first so exit now and come back after they've been preloaded.
		PackageObjLoaded.Append(ThreadObjLoaded);
		ThreadObjLoaded.Reset();
		return EAsyncPackageState::TimeOut;
	}

	if (GEventDrivenLoaderEnabled)
	{
		PreLoadIndex = PackageObjLoaded.Num(); // we did preloading in a different way and never incremented this
	}

	const bool bAsyncPostLoadEnabled = FAsyncLoadingThreadSettings::Get().bAsyncPostLoadEnabled;
	const bool bIsMultithreaded = AsyncLoadingThread.IsMultithreaded();

	// PostLoad objects.
	while (PostLoadIndex < PackageObjLoaded.Num() && PostLoadIndex < PreLoadIndex && !IsTimeLimitExceeded())
	{
		UObject* Object = PackageObjLoaded[PostLoadIndex++];
		if (Object)
		{
			if (!Object->IsReadyForAsyncPostLoad())
			{
				--PostLoadIndex;
				break;
			}
			else if (!bIsMultithreaded || (bAsyncPostLoadEnabled && CanPostLoadOnAsyncLoadingThread(Object)))
			{
				SCOPED_ACCUM_LOADTIME(PostLoad, StaticGetNativeClassName(Object->GetClass()));

				FScopeCycleCounterUObject ConstructorScope(Object, GET_STATID(STAT_FAsyncPackage_PostLoadObjects));

				// We want this check only with EDL enabled
				check(!GEventDrivenLoaderEnabled || !Object->HasAnyFlags(RF_NeedLoad));

				ThreadContext.CurrentlyPostLoadedObjectByALT = Object;
				{
					TRACE_LOADTIME_POSTLOAD_EXPORT_SCOPE(Object);
					Object->ConditionalPostLoad();
				}
				ThreadContext.CurrentlyPostLoadedObjectByALT = nullptr;

				LastObjectWorkWasPerformedOn = Object;
				LastTypeOfWorkPerformed = TEXT("postloading_async");

				if (ThreadObjLoaded.Num())
				{
					// New objects have been loaded. They need to go through PreLoad first so exit now and come back after they've been preloaded.
					PackageObjLoaded.Append(ThreadObjLoaded);
					ThreadObjLoaded.Reset();
					return EAsyncPackageState::TimeOut;
				}
			}
			else
			{
				DeferredPostLoadObjects.Add(Object);
			}
			// All object must be finalized on the game thread
			DeferredFinalizeObjects.Add(Object);
			check(Object->IsValidLowLevelFast());
			// Make sure all objects in DeferredFinalizeObjects are referenced too
			AddObjectReference(Object);
		}
	}

	PackageObjLoaded.Append(ThreadObjLoaded);
	ThreadObjLoaded.Reset();

	// New objects might have been loaded during PostLoad.
	EAsyncPackageState::Type Result = (PreLoadIndex == PackageObjLoaded.Num()) && (PostLoadIndex == PackageObjLoaded.Num()) ? EAsyncPackageState::Complete : EAsyncPackageState::TimeOut;
	return Result;
}
```

이제 **ASync 스레드**가 패키지를 디스크에서 메모리로 가져오는 작업을 끝내고 FAsyncLoadingThread::LoadedPackage에 그 패키지를 담았다.       
             
이제 **게임 스레드**는 로딩이 끝난 패키지 ( FAsyncLoadingThread::LoadedPackage )에 대해 PostLoad 동작을 수행한다.              

```cpp
// 게임스레드가 호출하는 함수이다.
/**
* [GAME THREAD] Ticks game thread side of async loading.
*
* @param bUseTimeLimit True if time limit should be used [time-slicing].
* @param bUseFullTimeLimit True if full time limit should be used [time-slicing].
* @param TimeLimit Maximum amount of time that can be spent in this call [time-slicing].
* @param FlushTree Package dependency tree to be flushed
* @return The current state of async loading
*/
EAsyncPackageState::Type FAsyncLoadingThread::TickAsyncLoading(bool bUseTimeLimit, bool bUseFullTimeLimit, float TimeLimit, FFlushTree* FlushTree)
{
	LLM_SCOPE(ELLMTag::AsyncLoading);

	check(IsInGameThread());
	check(!IsGarbageCollecting());

	const bool bLoadingSuspended = IsAsyncLoadingSuspendedInternal();
	EAsyncPackageState::Type Result = bLoadingSuspended ? EAsyncPackageState::PendingImports : EAsyncPackageState::Complete;

	if (!bLoadingSuspended)
	{
		// First make sure there's no objects pending to be unhashed. This is important in uncooked builds since we don't 
		// detach linkers immediately there and we may end up in getting unreachable objects from Linkers in CreateImports
		if (FPlatformProperties::RequiresCookedData() == false && IsIncrementalUnhashPending() && IsAsyncLoadingPackages())
		{
			// Call ConditionalBeginDestroy on all pending objects. CBD is where linkers get detached from objects.
			UnhashUnreachableObjects(false);
		}

		const bool bIsMultithreaded = FAsyncLoadingThread::IsMultithreaded();
		double TickStartTime = FPlatformTime::Seconds();
		double TimeLimitUsedForProcessLoaded = 0;

		bool bDidSomething = false;
		{
			// ⭐
			Result = ProcessLoadedPackages(bUseTimeLimit, bUseFullTimeLimit, TimeLimit, bDidSomething, FlushTree);
			TimeLimitUsedForProcessLoaded = FPlatformTime::Seconds() - TickStartTime;
			UE_CLOG(!GIsEditor && bUseTimeLimit && TimeLimitUsedForProcessLoaded > .1f, LogStreaming, Warning, TEXT("Took %6.2fms to ProcessLoadedPackages"), float(TimeLimitUsedForProcessLoaded) * 1000.0f);
		}

		...
		...
		...
		...
	}
	return Result;
}

EAsyncPackageState::Type FAsyncLoadingThread::ProcessLoadedPackages(bool bUseTimeLimit, bool bUseFullTimeLimit, float TimeLimit, bool& bDidSomething, FFlushTree* FlushTree)
{
	EAsyncPackageState::Type Result = EAsyncPackageState::Complete;

	double TickStartTime = FPlatformTime::Seconds();

	{
#if THREADSAFE_UOBJECTS
		FScopeLock LoadedPackagesLock(&LoadedPackagesCritical);
		FScopeLock LoadedPackagesToProcessLock(&LoadedPackagesToProcessCritical);
#endif
		if (LoadedPackages.Num() != 0)
		{
			LoadedPackagesToProcess.Append(LoadedPackages);
			LoadedPackages.Reset();
		}
		if (LoadedPackagesNameLookup.Num() != 0)
		{
			LoadedPackagesToProcessNameLookup.Append(LoadedPackagesNameLookup);
			LoadedPackagesNameLookup.Reset();
		}
	}

	bDidSomething = LoadedPackagesToProcess.Num() > 0;
	for (int32 PackageIndex = 0; PackageIndex < LoadedPackagesToProcess.Num() && !IsAsyncLoadingSuspendedInternal(); ++PackageIndex)
	{
		FAsyncPackage* Package = LoadedPackagesToProcess[PackageIndex];
		if (Package->GetDependencyRefCount() == 0)
		{
			Result = Package->PostLoadDeferredObjects(TickStartTime, bUseTimeLimit, TimeLimit);
			if (Result == EAsyncPackageState::Complete)
			{
				// Remove the package from the list before we trigger the callbacks, 
				// this is to ensure we can re-enter FlushAsyncLoading from any of the callbacks
				{
					FScopeLock LoadedLock(&LoadedPackagesToProcessCritical);
					LoadedPackagesToProcess.RemoveAt(PackageIndex--);
					LoadedPackagesToProcessNameLookup.Remove(Package->GetPackageName());

					if (FPlatformProperties::RequiresCookedData())
					{
						// Emulates ResetLoaders on the package linker's linkerroot.
						if (!Package->IsBeingProcessedRecursively())
						{
							Package->ResetLoader();
						}
					}
					else
					{
						// Detach linker in mutex scope to make sure that if something requests this package
						// before it's been deleted does not try to associate the new async package with the old linker
						// while this async package is still bound to it.
						Package->DetachLinker();
					}

					// Close linkers opened by synchronous loads during async loading
					Package->CloseDelayedLinkers();
				}

				// Incremented on the Async Thread, now decrement as we're done with this package				
				const int32 NewExistingAsyncPackagesCounterValue = ExistingAsyncPackagesCounter.Decrement();
				NotifyAsyncLoadingStateHasMaybeChanged();

				UE_CLOG(NewExistingAsyncPackagesCounterValue < 0, LogStreaming, Fatal, TEXT("ExistingAsyncPackagesCounter is negative, this means we loaded more packages then requested so there must be a bug in async loading code."));

				// Call external callbacks
				const bool bInternalCallbacks = false;
				const EAsyncLoadingResult::Type LoadingResult = Package->HasLoadFailed() ? EAsyncLoadingResult::Failed : EAsyncLoadingResult::Succeeded;
				Package->CallCompletionCallbacks(bInternalCallbacks, LoadingResult);

				// We don't need the package anymore
				PackagesToDelete.AddUnique(Package);
				Package->MarkRequestIDsAsComplete();

				TRACE_LOADTIME_END_LOAD_ASYNC_PACKAGE(Package);

				if (IsTimeLimitExceeded(TickStartTime, bUseTimeLimit, TimeLimit, TEXT("ProcessLoadedPackages Misc")) || (FlushTree && !ContainsRequestID(FlushTree->RequestId)))
				{
					// The only package we care about has finished loading, so we're good to exit
					break;
				}
			}
			else
			{
				break;
			}
		}
		else
		{
			Result = EAsyncPackageState::PendingImports;
			// Break immediately, we want to keep the order of processing when packages get here
			break;
		}
	}

	return Result;
}


EAsyncPackageState::Type FAsyncPackage::PostLoadDeferredObjects(double InTickStartTime, bool bInUseTimeLimit, float& InOutTimeLimit)
{
	SCOPE_CYCLE_COUNTER(STAT_FAsyncPackage_PostLoadObjectsGameThread);
	SCOPED_LOADTIMER(PostLoadDeferredObjectsTime);

	FAsyncPackageScope PackageScope(this);

	EAsyncPackageState::Type Result = EAsyncPackageState::Complete;
	TGuardValue<bool> GuardIsRoutingPostLoad(PackageScope.ThreadContext.IsRoutingPostLoad, true);
	FAsyncLoadingTickScope InAsyncLoadingTick(AsyncLoadingThread);

	FUObjectSerializeContext* LoadContext = GetSerializeContext();
	TArray<UObject*>& ObjLoadedInPostLoad = LoadContext->PRIVATE_GetObjectsLoadedInternalUseOnly();
	TArray<UObject*> ObjLoadedInPostLoadLocal;

	STAT(double PostLoadStartTime = FPlatformTime::Seconds());

	while (DeferredPostLoadIndex < DeferredPostLoadObjects.Num() && 
		!AsyncLoadingThread.IsAsyncLoadingSuspendedInternal() &&
		!::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn))
	{
		UObject* Object = DeferredPostLoadObjects[DeferredPostLoadIndex++];
		check(Object);

		if (!Object->IsReadyForAsyncPostLoad())
		{
			--DeferredPostLoadIndex;
			break;
		}

		LastObjectWorkWasPerformedOn = Object;
		LastTypeOfWorkPerformed = TEXT("postloading_gamethread");

		FScopeCycleCounterUObject ConstructorScope(Object, GET_STATID(STAT_FAsyncPackage_PostLoadObjectsGameThread));

		PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = Object;
		{
			TRACE_LOADTIME_POSTLOAD_EXPORT_SCOPE(Object);
			Object->ConditionalPostLoad();
		}
		PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = nullptr;

		if (ObjLoadedInPostLoad.Num())
		{
			// If there were any LoadObject calls inside of PostLoad, we need to pre-load those objects here. 
			// There's no going back to the async tick loop from here.
			UE_LOG(LogStreaming, Warning, TEXT("Detected %d objects loaded in PostLoad while streaming, this may cause hitches as we're blocking async loading to pre-load them."), ObjLoadedInPostLoad.Num());
			
			// Copy to local array because ObjLoadedInPostLoad can change while we're iterating over it
			ObjLoadedInPostLoadLocal.Append(ObjLoadedInPostLoad);
			ObjLoadedInPostLoad.Reset();

			while (ObjLoadedInPostLoadLocal.Num())
			{
				// Make sure all objects loaded in PostLoad get post-loaded too
				DeferredPostLoadObjects.Append(ObjLoadedInPostLoadLocal);

				// Preload (aka serialize) the objects loaded in PostLoad.
				for (UObject* PreLoadObject : ObjLoadedInPostLoadLocal)
				{
					if (PreLoadObject && PreLoadObject->GetLinker())
					{
						PreLoadObject->GetLinker()->Preload(PreLoadObject);
					}
				}

				// Other objects could've been loaded while we were preloading, continue until we've processed all of them.
				ObjLoadedInPostLoadLocal.Reset();
				ObjLoadedInPostLoadLocal.Append(ObjLoadedInPostLoad);
				ObjLoadedInPostLoad.Reset();
			}			
		}

		LastObjectWorkWasPerformedOn = Object;		

		UpdateLoadPercentage();
	}

	INC_FLOAT_STAT_BY(STAT_FAsyncPackage_TotalPostLoadGameThread, (float)(FPlatformTime::Seconds() - PostLoadStartTime));

	// New objects might have been loaded during PostLoad.
	Result = (DeferredPostLoadIndex == DeferredPostLoadObjects.Num()) ? EAsyncPackageState::Complete : EAsyncPackageState::TimeOut;
	if (Result == EAsyncPackageState::Complete)
	{
		LastObjectWorkWasPerformedOn = nullptr;
		LastTypeOfWorkPerformed = TEXT("DeferredFinalizeObjects");
		TArray<UObject*> CDODefaultSubobjects;
		// Clear async loading flags (we still want RF_Async, but EInternalObjectFlags::AsyncLoading can be cleared)
		while (DeferredFinalizeIndex < DeferredFinalizeObjects.Num() &&
			(DeferredPostLoadIndex % 100 != 0 || (!AsyncLoadingThread.IsAsyncLoadingSuspendedInternal() && !::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn))))
		{
			UObject* Object = DeferredFinalizeObjects[DeferredFinalizeIndex++];
			if (Object)
			{
				Object->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
			}

			// CDO need special handling, no matter if it's listed in DeferredFinalizeObjects or created here for DynamicClass
			UObject* CDOToHandle = nullptr;

			// Dynamic Class doesn't require/use pre-loading (or post-loading). 
			// The CDO is created at this point, because now it's safe to solve cyclic dependencies.
			if (UDynamicClass* DynamicClass = Cast<UDynamicClass>(Object))
			{
				check((DynamicClass->ClassFlags & CLASS_Constructed) != 0);

				if (GEventDrivenLoaderEnabled)
				{
					//native blueprint 

					check(DynamicClass->HasAnyClassFlags(CLASS_TokenStreamAssembled));
					// this block should be removed entirely when and if we add the CDO to the fake export table
					CDOToHandle = DynamicClass->GetDefaultObject(false);
					UE_CLOG(!CDOToHandle, LogStreaming, Fatal, TEXT("EDL did not create the CDO for %s before it finished loading."), *DynamicClass->GetFullName());
					CDOToHandle->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
				}
				else
				{
					UObject* OldCDO = DynamicClass->GetDefaultObject(false);
					UObject* NewCDO = DynamicClass->GetDefaultObject(true);
					const bool bCDOWasJustCreated = (OldCDO != NewCDO);
					if (bCDOWasJustCreated && (NewCDO != nullptr))
					{
						NewCDO->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
						CDOToHandle = NewCDO;
					}
				}
			}
			else
			{
				CDOToHandle = ((Object != nullptr) && Object->HasAnyFlags(RF_ClassDefaultObject)) ? Object : nullptr;
			}

			// Clear AsyncLoading in CDO's subobjects.
			if(CDOToHandle != nullptr)
			{
				CDOToHandle->GetDefaultSubobjects(CDODefaultSubobjects);
				for (UObject* SubObject : CDODefaultSubobjects)
				{
					if (SubObject && SubObject->HasAnyInternalFlags(EInternalObjectFlags::AsyncLoading))
					{
						SubObject->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
					}
				}
				CDODefaultSubobjects.Reset();
			}
		}
		::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn);
		if (DeferredFinalizeIndex == DeferredFinalizeObjects.Num())
		{
			DeferredFinalizeIndex = 0;
			DeferredFinalizeObjects.Reset();
			Result = EAsyncPackageState::Complete;
		}
		else
		{
			Result = EAsyncPackageState::TimeOut;
		}

		// Mark package as having been fully loaded and update load time.
		if (Result == EAsyncPackageState::Complete && LinkerRoot && !bLoadHasFailed)
		{
			LastObjectWorkWasPerformedOn = LinkerRoot;
			LastTypeOfWorkPerformed = TEXT("CreateClustersFromPackage");
			LinkerRoot->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
			LinkerRoot->MarkAsFullyLoaded();			
			LinkerRoot->SetLoadTime(FPlatformTime::Seconds() - LoadStartTime);

			if (Linker)
			{
				CreateClustersFromPackage(Linker, DeferredClusterObjects);
			}
			::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn);
		}

		FSoftObjectPath::InvalidateTag();
		FUniqueObjectGuid::InvalidateTag();
	}

	return Result;
}

EAsyncPackageState::Type FAsyncPackage::PostLoadDeferredObjects(double InTickStartTime, bool bInUseTimeLimit, float& InOutTimeLimit)
{
	SCOPE_CYCLE_COUNTER(STAT_FAsyncPackage_PostLoadObjectsGameThread);
	SCOPED_LOADTIMER(PostLoadDeferredObjectsTime);

	FAsyncPackageScope PackageScope(this);

	EAsyncPackageState::Type Result = EAsyncPackageState::Complete;
	TGuardValue<bool> GuardIsRoutingPostLoad(PackageScope.ThreadContext.IsRoutingPostLoad, true);
	FAsyncLoadingTickScope InAsyncLoadingTick(AsyncLoadingThread);

	FUObjectSerializeContext* LoadContext = GetSerializeContext();
	TArray<UObject*>& ObjLoadedInPostLoad = LoadContext->PRIVATE_GetObjectsLoadedInternalUseOnly();
	TArray<UObject*> ObjLoadedInPostLoadLocal;

	STAT(double PostLoadStartTime = FPlatformTime::Seconds());

	while (DeferredPostLoadIndex < DeferredPostLoadObjects.Num() && 
		!AsyncLoadingThread.IsAsyncLoadingSuspendedInternal() &&
		!::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn))
	{
		UObject* Object = DeferredPostLoadObjects[DeferredPostLoadIndex++];
		check(Object);

		if (!Object->IsReadyForAsyncPostLoad())
		{
			--DeferredPostLoadIndex;
			break;
		}

		LastObjectWorkWasPerformedOn = Object;
		LastTypeOfWorkPerformed = TEXT("postloading_gamethread");

		FScopeCycleCounterUObject ConstructorScope(Object, GET_STATID(STAT_FAsyncPackage_PostLoadObjectsGameThread));

		PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = Object;
		{
			TRACE_LOADTIME_POSTLOAD_EXPORT_SCOPE(Object);
			// ⭐⭐⭐⭐⭐⭐⭐
			// 드디어 오브젝트에 대한 PostLoad 호출!
			Object->ConditionalPostLoad();
			// ⭐⭐⭐⭐⭐⭐⭐
		}
		PackageScope.ThreadContext.CurrentlyPostLoadedObjectByALT = nullptr;

		if (ObjLoadedInPostLoad.Num())
		{
			// If there were any LoadObject calls inside of PostLoad, we need to pre-load those objects here. 
			// There's no going back to the async tick loop from here.
			UE_LOG(LogStreaming, Warning, TEXT("Detected %d objects loaded in PostLoad while streaming, this may cause hitches as we're blocking async loading to pre-load them."), ObjLoadedInPostLoad.Num());
			
			// Copy to local array because ObjLoadedInPostLoad can change while we're iterating over it
			ObjLoadedInPostLoadLocal.Append(ObjLoadedInPostLoad);
			ObjLoadedInPostLoad.Reset();

			while (ObjLoadedInPostLoadLocal.Num())
			{
				// Make sure all objects loaded in PostLoad get post-loaded too
				DeferredPostLoadObjects.Append(ObjLoadedInPostLoadLocal);

				// Preload (aka serialize) the objects loaded in PostLoad.
				for (UObject* PreLoadObject : ObjLoadedInPostLoadLocal)
				{
					if (PreLoadObject && PreLoadObject->GetLinker())
					{
						PreLoadObject->GetLinker()->Preload(PreLoadObject);
					}
				}

				// Other objects could've been loaded while we were preloading, continue until we've processed all of them.
				ObjLoadedInPostLoadLocal.Reset();
				ObjLoadedInPostLoadLocal.Append(ObjLoadedInPostLoad);
				ObjLoadedInPostLoad.Reset();
			}			
		}

		LastObjectWorkWasPerformedOn = Object;		

		UpdateLoadPercentage();
	}

	INC_FLOAT_STAT_BY(STAT_FAsyncPackage_TotalPostLoadGameThread, (float)(FPlatformTime::Seconds() - PostLoadStartTime));

	// New objects might have been loaded during PostLoad.
	Result = (DeferredPostLoadIndex == DeferredPostLoadObjects.Num()) ? EAsyncPackageState::Complete : EAsyncPackageState::TimeOut;
	if (Result == EAsyncPackageState::Complete)
	{
		LastObjectWorkWasPerformedOn = nullptr;
		LastTypeOfWorkPerformed = TEXT("DeferredFinalizeObjects");
		TArray<UObject*> CDODefaultSubobjects;
		// Clear async loading flags (we still want RF_Async, but EInternalObjectFlags::AsyncLoading can be cleared)
		while (DeferredFinalizeIndex < DeferredFinalizeObjects.Num() &&
			(DeferredPostLoadIndex % 100 != 0 || (!AsyncLoadingThread.IsAsyncLoadingSuspendedInternal() && !::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn))))
		{
			UObject* Object = DeferredFinalizeObjects[DeferredFinalizeIndex++];
			if (Object)
			{
				Object->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
			}

			// CDO need special handling, no matter if it's listed in DeferredFinalizeObjects or created here for DynamicClass
			UObject* CDOToHandle = nullptr;

			// Dynamic Class doesn't require/use pre-loading (or post-loading). 
			// The CDO is created at this point, because now it's safe to solve cyclic dependencies.
			if (UDynamicClass* DynamicClass = Cast<UDynamicClass>(Object))
			{
				check((DynamicClass->ClassFlags & CLASS_Constructed) != 0);

				if (GEventDrivenLoaderEnabled)
				{
					//native blueprint 

					check(DynamicClass->HasAnyClassFlags(CLASS_TokenStreamAssembled));
					// this block should be removed entirely when and if we add the CDO to the fake export table
					CDOToHandle = DynamicClass->GetDefaultObject(false);
					UE_CLOG(!CDOToHandle, LogStreaming, Fatal, TEXT("EDL did not create the CDO for %s before it finished loading."), *DynamicClass->GetFullName());
					CDOToHandle->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
				}
				else
				{
					UObject* OldCDO = DynamicClass->GetDefaultObject(false);
					UObject* NewCDO = DynamicClass->GetDefaultObject(true);
					const bool bCDOWasJustCreated = (OldCDO != NewCDO);
					if (bCDOWasJustCreated && (NewCDO != nullptr))
					{
						NewCDO->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
						CDOToHandle = NewCDO;
					}
				}
			}
			else
			{
				CDOToHandle = ((Object != nullptr) && Object->HasAnyFlags(RF_ClassDefaultObject)) ? Object : nullptr;
			}

			// Clear AsyncLoading in CDO's subobjects.
			if(CDOToHandle != nullptr)
			{
				CDOToHandle->GetDefaultSubobjects(CDODefaultSubobjects);
				for (UObject* SubObject : CDODefaultSubobjects)
				{
					if (SubObject && SubObject->HasAnyInternalFlags(EInternalObjectFlags::AsyncLoading))
					{
						SubObject->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
					}
				}
				CDODefaultSubobjects.Reset();
			}
		}
		::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn);
		if (DeferredFinalizeIndex == DeferredFinalizeObjects.Num())
		{
			DeferredFinalizeIndex = 0;
			DeferredFinalizeObjects.Reset();
			Result = EAsyncPackageState::Complete;
		}
		else
		{
			Result = EAsyncPackageState::TimeOut;
		}

		// Mark package as having been fully loaded and update load time.
		if (Result == EAsyncPackageState::Complete && LinkerRoot && !bLoadHasFailed)
		{
			LastObjectWorkWasPerformedOn = LinkerRoot;
			LastTypeOfWorkPerformed = TEXT("CreateClustersFromPackage");
			LinkerRoot->AtomicallyClearInternalFlags(EInternalObjectFlags::AsyncLoading);
			LinkerRoot->MarkAsFullyLoaded();			
			LinkerRoot->SetLoadTime(FPlatformTime::Seconds() - LoadStartTime);

			if (Linker)
			{
				CreateClustersFromPackage(Linker, DeferredClusterObjects);
			}
			::IsTimeLimitExceeded(InTickStartTime, bInUseTimeLimit, InOutTimeLimit, LastTypeOfWorkPerformed, LastObjectWorkWasPerformedOn);
		}

		FSoftObjectPath::InvalidateTag();
		FUniqueObjectGuid::InvalidateTag();
	}

	return Result;
}
```

references : [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/)      