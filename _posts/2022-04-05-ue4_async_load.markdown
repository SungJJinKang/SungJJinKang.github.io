---
layout: post
title:  "UE4 Asynchronous 에셋 로드 코드 분석"
date:   2022-04-02
categories: UE4 UnrealEngine4 ComputerScience
---


```
TSharedPtr<FStreamableHandle> FStreamableManager::RequestAsyncLoad(TArray<FSoftObjectPath> TargetsToStream, FStreamableDelegate DelegateToCall, TAsyncLoadPriority Priority, bool bManageActiveHandle, bool bStartStalled, FString DebugName)
{
    // FStreamableHandle : 동기적, 비동기적 에셋 로드와 관련된 데이터를 가짐. 로드 후 호출 델리게이트, 로드 우선 순위, 로드 완료 여부, 등등 유저 코드에서 접근할 수 있는 에셋 로딩과 관련된 모든 데이터가 들어 있다.               
	TSharedRef<FStreamableHandle> NewRequest = MakeShareable(new FStreamableHandle());
	NewRequest->CompleteDelegate = DelegateToCall;
	NewRequest->OwningManager = this;
	NewRequest->RequestedAssets = MoveTemp(TargetsToStream); // 로드할 에셋들에 대한 경로를 FStreamableHandle에 저장.
#if (!PLATFORM_IOS && !PLATFORM_ANDROID)
	NewRequest->DebugName = MoveTemp(DebugName); // 디버깅용 Name을 전달.
#endif
	NewRequest->Priority = Priority;

	int32 NumValidRequests = NewRequest->RequestedAssets.Num();
	
	TSet<FSoftObjectPath> TargetSet;
	TargetSet.Reserve(NumValidRequests);

    // 전달된 에셋 경로 중 유효한 것들만 걸러냄.
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
#if UE_BUILD_DEBUG
		FString RequestedSet;

		for (const FSoftObjectPath& Asset : NewRequest->RequestedAssets)
		{
			if (!RequestedSet.IsEmpty())
			{
				RequestedSet += TEXT(", ");
			}
			RequestedSet += Asset.ToString();
		}

		UE_LOG(LogStreamableManager, Verbose, TEXT("RequestAsyncLoad called with duplicate assets, duplicates removed from %s!"), *RequestedSet);
#endif

		NewRequest->RequestedAssets = TargetSet.Array();
	}

	if (bManageActiveHandle)
	{
		// This keeps a reference around until explicitly released
		ManagedActiveHandles.Add(NewRequest);
	}

	if (bStartStalled)
	{
		NewRequest->bStalled = true;
	}
	else
	{
		StartHandleRequests(NewRequest);
	}

	return NewRequest;
}
```



references : [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/Assets/AsyncLoading/)      