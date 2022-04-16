---
layout: post
title:  "어떻게 언리얼 엔진4는 게임 스레드에서 렌더스레드로 데이터를 안전하게 전송할까?"
date:   2022-03-27
categories: ComputerScience ComputerGraphics UE4 UnrealEngine4
---

언리얼 엔진4에서는 CPU를 게임 스레드, 렌더링 스레드로 분리하여 렌더링을 수행한다.      
GPU까지 해서 **CPU의 게임 스레드, 렌더 스레드(Draw 스레드), GPU 3개의 스레드가 동시에 동작**한다.           
게임 스레드가 먼저 연산을 하면 다음 프레임에 렌더링 스레드가 이전 프레임의 게임 스레드의 데이터를 가지고 동작을 수행하고, 한 프레임 후 GPU가 렌더링을 수행한다.          
                                
<img width="560" alt="7" src="https://user-images.githubusercontent.com/33873804/159112192-be363c0d-f72b-490c-9612-ca1cf6579e14.png">      
           
약간은 생소한 개념인데 **게임 스레드**는 흔히 에니메이션, 오브젝트의 Transform 데이터, 물리 연산 처리, AI 연산 처리 등등 **게임 로직과 관련된 전반적인 연산들을 수행하는 스레드**이다. 게임 스레드는 CPU에서 수행된다. 후에 렌더 스레드가 활용할 여러 렌더링 관련 변수 값 셋팅도 게임 스레드에서 수행한다. 예를 들면 게임 스레드에서 Distance Culling을 위한 어떤 오브젝트의 Max Desired Draw Distance 값을 수정하면 이 값을 다음 프레임에 렌더 스레드가 Distance Culling을 수행하는데 사용한다.                                      

**렌더 스레드**도 대부분 CPU에서 수행된다. 렌더 스레드에서는 **각종 가시성 연산을 수행**한다. 쉽게 말하면 컬링 연산을 수행한다. 현대 게임들은 대부분 GPU Bound한 경우가 많기 때문에 최대한 GPU의 연산 부담을 덜어주기 위해 CPU에서 렌더링 할 필요가 없는 오브젝트들을 걸러내기 위한 연산을 수행한다. Distance 컬링, 뷰 프러스텀 컬링, (SW) 오클루전 컬링, PreComputed Visibility 등이 있다. 대부분 CPU에서 수행된다.                                    
또한 렌더 스레드에서는 **렌더링 커맨드를 생성하여 GPU에 제출**한다. 흔히 **그래픽스 API를 호출하는 곳이 이 렌더 스레드**라고 생각하면 된다.                   

마지막으로 **GPU**는 그냥 GPU이다. 렌더 스레드의 커맨드를 받아서 GPU 연산을 수행한다.          
                    
D3D12, Vulkan에 와서는 RHI 스레드라는 것을 만들어서 렌더 스레드에서는 플랫폼 독립적인 커맨드 큐를 CPU쪽에 생성해두었다가, RHI 스레드에서 목표로 하는 플랫폼(Graphics API)에 맞는 그래픽 API를 호출해준다. 그러나 필자의 게임 엔진은 D3D11, OpenGL만을 지원하기 때문에 그래픽 API를 호출하는 부분도 렌더 스레드에 넣을 생각이다.             
                
참고 자료 : [Unreal Engine4 렌더링 파이프라인 분석 - SungJJinKang](https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/02/26/ue4_rendering.html), [그래픽 프로그래밍 개요](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/Overview/), [스레디드 렌더링](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/ThreadedRendering/), [병렬 렌더링 개요](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/ParallelRendering/)            
            
----------------------------------------------------           
            
게임 스레드와 렌더 스레드를 분리하게 되면 당연히 가장 큰 문제는 **Data Race**이다.                                                
그렇다고 무턱대고 **Mutex로 Lock을 해버리면 겁나게 느려진다.......**                
게임 스레드와 렌더 스레드가 동시에(!) 쉬지(Idle 상태) 않고 도는 것이 중요하다. 한 스레드가 다른 스레드를 멈추게 구현하면 안된다.         

--------------------------------------------------           
         
불필요한 코드들은 임의로 제거하였으니 Full 코드는 직접 엔진 소스코드를 참조하기 바란다.                 
              
언리얼 엔진4에서는 게임 스레드와 렌더 스레드가 존재하고, 그 두 스레드는 동시에(Concurrent) 동작을 한다.             
게임 스레드에서 어떤 렌더링 관련 데이터를 변경하면 이 데이터는 뒤따라오는 렌더 스레드에 반영이 되어야한다.                
당연한 얘기겠지만 게임 스레드와 렌더 스레드간의 Data Race 문제를 막기 위해 언리얼 엔진4는 두 스레드가 접근할 수 있는 데이터들을 분리해두었다.         
흔히 ~Proxy로 끝나는 클래스가 렌더 스레드가 관리하는 클래스이다.                       
            
예를 들어보면 게임 스레드는 렌더링과 관련된 컴포넌트의 Base 컴포넌트로 "UPrimitiveComponent" 컴포넌트를 가진다. 게임 스레드가 이 클래스에 데이터를 쓰기 때문에 Data Race를 막기 위해 렌더 스레드는 절대로 이 클래스에 직접 데이터를 쓰면 안된다.             
비슷하게 렌더 스레드도 "FPrimitiveSceneProxy"라는 클래스로 렌더 스레드에서 사용할 데이터를 관리한다. 마찬가지로 게임 스레드에서 절대로 이 클래스에 직접 데이터를 쓰면 안된다.                    
언리얼 엔진4는 이렇게 게임 스레드, 렌더 스레드가 접근할 수 있는 데이터를 분리하고 서로 접근할 수 없도록 만들어 두 스레드가 독립적으로 Data Race없이 돌아가게 만든다.      
                 
------------------------------------------------------

흔히 가장 많이 사용되는 "UStaticMeshComponent" 컴포넌트가 어떻게 렌더링을 수행하는지를 코드를 보며 따라가보며, 언리얼 엔진4가 게임 스레드와 렌더 스레드를 활용하여 렌더링을 하는 과정에 대해 알아보겠다.             

우선 컴포넌트를 등록하게 되면 "UActorComponent::RegisterComponentWithWorld" 함수를 통해 컴포넌트를 월드에 등록한 후 해당 컴포넌트가 렌더링과 관련된 컴포넌트인 경우 "UActorComponent::CreateRenderState_Concurrent" 함수를 호출한다.         
이 "UActorComponent::CreateRenderState_Concurrent" 함수를 통해 렌더링에 필요한 여러 데이터들을 게임 스레드에서 생성하고 렌더스레드에 넘겨주어서 그 데이터를 렌더스레드의 렌더링에 반영한다.               
렌더링 관련 데이터가 변경된 경우에도 RenderState를 파괴한 후 "UActorComponent::CreateRenderState_Concurrent" 함수를 다시 호출해서 RenderState를 재생성해준다.       

```cpp
void UPrimitiveComponent::CreateRenderState_Concurrent(FRegisterComponentContext* Context) // UPrimitiveComponent는 UStaticMeshComponent의 Base 클래스이다.
{
	Super::CreateRenderState_Concurrent(Context);

	UpdateBounds();

	// If the primitive isn't hidden and the detail mode setting allows it, add it to the scene.
	if (ShouldComponentAddToScene())
	{
		if (Context != nullptr)
		{
			Context->AddPrimitive(this);
		}
		else
		{
			GetWorld()->Scene->AddPrimitive(this);
		}
	}
}
```
그럼 World의 Scene에 이 Primitive를 추가한다.        
"FScene" 클래스는 "UWorld"(게임 스레드가 소유)의 렌더스레드 버전이라고 생각하면 된다.          
"FScene"은 렌더링과 관련된 모든 데이터를 관리한다.       
절대로 "FScene"의 데이터를 게임스레드에서 Write해서는 안된다.            

Fscene에 렌더링할 Primitive를 추가하게 되면
```cpp
void FScene::AddPrimitive(UPrimitiveComponent* Primitive)
{
	// Create the primitive's scene proxy.
	FPrimitiveSceneProxy* PrimitiveSceneProxy = Primitive->CreateSceneProxy(); // UPrimitiveComponent 대한 FPrimitiveSceneProxy 행성.
	Primitive->SceneProxy = PrimitiveSceneProxy;
	if(!PrimitiveSceneProxy)
	{
		// Primitives which don't have a proxy are irrelevant to the scene manager.
		return;
	}

	// Create the primitive scene info.
	FPrimitiveSceneInfo* PrimitiveSceneInfo = new FPrimitiveSceneInfo(Primitive, this); 
	PrimitiveSceneProxy->PrimitiveSceneInfo = PrimitiveSceneInfo;

	// Cache the primitives initial transform.
	FMatrix RenderMatrix = Primitive->GetRenderMatrix();
	FVector AttachmentRootPosition(0);

	AActor* AttachmentRoot = Primitive->GetAttachmentRootActor();
	if (AttachmentRoot)
	{
		AttachmentRootPosition = AttachmentRoot->GetActorLocation();
	}

	struct FCreateRenderThreadParameters
	{
		FPrimitiveSceneProxy* PrimitiveSceneProxy;
		FMatrix RenderMatrix;
		FBoxSphereBounds WorldBounds;
		FVector AttachmentRootPosition;
		FBoxSphereBounds LocalBounds;
	};
	FCreateRenderThreadParameters Params =
	{
		PrimitiveSceneProxy,
		RenderMatrix,
		Primitive->Bounds,
		AttachmentRootPosition,
		Primitive->CalcBounds(FTransform::Identity)
	};

	// Create any RenderThreadResources required and send a command to the rendering thread to add the primitive to the scene.
	FScene* Scene = this;

	// If this primitive has a simulated previous transform, ensure that the velocity data for the scene representation is correct
	TOptional<FTransform> PreviousTransform = FMotionVectorSimulation::Get().GetPreviousTransform(Primitive);

	ENQUEUE_RENDER_COMMAND(AddPrimitiveCommand)(
		[Params = MoveTemp(Params), Scene, PrimitiveSceneInfo, PreviousTransform = MoveTemp(PreviousTransform)](FRHICommandListImmediate& RHICmdList)
		{
			FPrimitiveSceneProxy* SceneProxy = Params.PrimitiveSceneProxy;
			FScopeCycleCounter Context(SceneProxy->GetStatId());
			SceneProxy->SetTransform(Params.RenderMatrix, Params.WorldBounds, Params.LocalBounds, Params.AttachmentRootPosition);

			// Create any RenderThreadResources required.
			SceneProxy->CreateRenderThreadResources();

			Scene->AddPrimitiveSceneInfo_RenderThread(PrimitiveSceneInfo, PreviousTransform);
		});
}
```
여기까지도 아직 게임 스레드에서 동작하는 영역이다.                 
          
UPrimitiveComponent의 렌더 스레드 버전인 FPrimitiveSceneProxy를 생성한다.     
```cpp
/**
 * Encapsulates the data which is mirrored to render a UPrimitiveComponent parallel to the game thread.
 * This is intended to be subclassed to support different primitive types.  
 */
class FPrimitiveSceneProxy
```          
           
"FPrimitiveSceneProxy"는 렌더링과 관련된 "UPrimitiveComponent"의 데이터들의 복사본을 들고 있다고 생각하면 된다. ( "FPrimitiveSceneInfo"타입의 "FPrimitiveSceneProxy::PrimitiveSceneInfo" 변수를 보면 알 수 있다. )         
"UPrimitiveComponent"의 경우 "FPrimitiveSceneProxy"를 상속받은 "FStaticMeshSceneProxy" 클래스를 렌더스레드에 넘겨준다.          
"FStaticMeshSceneProxy" 클래스에는 렌더링할 메쉬와 관련된 각종 데이터들(버텍스...)이 들어 있다.              
     
```cpp           
/**
 * The renderer's internal state for a single UPrimitiveComponent.  This has a one to one mapping with FPrimitiveSceneProxy, which is in the engine module.
 */
class FPrimitiveSceneInfo : public FDeferredCleanupInterface
```  

그 후 "ENQUEUE_RENDER_COMMAND" 매크로를 통해 게임 스레드에서 생성한 렌더링 관련 데이터를 렌더스레드에 반영해준다. ( 정확히는 렌더 스레드가 수행해야할 동작(람다 코드)를 데이터와 함께 전송하여서 렌더스레드가 그 동작을 수행하게 만든다. )                                   
"ENQUEUE_RENDER_COMMAND"의 Parameter로 넘긴 람다는 렌더스레드에서 호출이 된다.              
( "ENQUEUE_RENDER_COMMAND" 매크로에 대한 내용은 밑에서 더 자세히 다룰 것이다. )            
             
             
여기서부터는 렌더스레드의 동작이다.           
             

```cpp
void FScene::AddPrimitiveSceneInfo_RenderThread(FPrimitiveSceneInfo* PrimitiveSceneInfo, const TOptional<FTransform>& PreviousTransform)
{
	AddedPrimitiveSceneInfos.Add(PrimitiveSceneInfo);
	if (PreviousTransform.IsSet())
	{
		OverridenPreviousTransforms.Add(PrimitiveSceneInfo, PreviousTransform.GetValue().ToMatrixWithScale());
	}
}
```
이후 렌더 스레드는 게임 스레드에서 전송된 Task(람다)를 처리하며 "Scene::AddPrimitiveSceneInfo_RenderThread" 함수를 호출하여 게임 스레드의 "UPrimitiveComponent"와 관련된 데이터를 반영한다.             
"FPrimitiveSceneInfo" 데이터는 임시로 "FScene::AddedPrimitiveSceneInfos"에 저장이 되었다가, 다음 프레임에 "FScene::UpdateAllPrimitiveSceneInfos" 함수가 호출되며 렌더링에 반영된다.               
"FSceneRenderer::Render" 함수에서 "FScene::UpdateAllPrimitiveSceneInfos"를 호출한다.         
FScene::UpdateAllPrimitiveSceneInfos의 코드에 대한 분석은 [이 글](https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/04/16/FMobileSceneRenderer_1.html)에서 자세히 다루었다.                                       


비슷하게 SceneComponent의 Transform 데이터 ( 위치, 회전, 스케일 )가 변경되었을 때도 이를 렌더스레드에 반영한다.         
```cpp
void FScene::UpdatePrimitiveTransform(UPrimitiveComponent* Primitive)
{
	if(Primitive->SceneProxy)
	{
		// Check if the primitive needs to recreate its proxy for the transform update.
		if(Primitive->ShouldRecreateProxyOnUpdateTransform())
		{
			// Re-add the primitive from scratch to recreate the primitive's proxy.
			RemovePrimitive(Primitive);
			AddPrimitive(Primitive);
		}
		else
		{
			FVector AttachmentRootPosition(0);

			AActor* Actor = Primitive->GetAttachmentRootActor();
			if (Actor != NULL)
			{
				AttachmentRootPosition = Actor->GetActorLocation();
			}

			struct FPrimitiveUpdateParams
			{
				FScene* Scene;
				FPrimitiveSceneProxy* PrimitiveSceneProxy;
				FBoxSphereBounds WorldBounds;
				FBoxSphereBounds LocalBounds;
				FMatrix LocalToWorld;
				TOptional<FTransform> PreviousTransform;
				FVector AttachmentRootPosition;
			};

			FPrimitiveUpdateParams UpdateParams;
			UpdateParams.Scene = this;
			UpdateParams.PrimitiveSceneProxy = Primitive->SceneProxy;
			UpdateParams.WorldBounds = Primitive->Bounds;
			UpdateParams.LocalToWorld = Primitive->GetRenderMatrix();
			UpdateParams.AttachmentRootPosition = AttachmentRootPosition;
			UpdateParams.LocalBounds = Primitive->CalcBounds(FTransform::Identity);
			UpdateParams.PreviousTransform = FMotionVectorSimulation::Get().GetPreviousTransform(Primitive);

			ENQUEUE_RENDER_COMMAND(UpdateTransformCommand)( // !!!!!!!!!!!
				[UpdateParams](FRHICommandListImmediate& RHICmdList)
				{ 
					// ⭐ 이 람다 코드는 렌더스레드에서만 호출된다. ⭐
					FScopeCycleCounter Context(UpdateParams.PrimitiveSceneProxy->GetStatId());
					UpdateParams.Scene->UpdatePrimitiveTransform_RenderThread(UpdateParams.PrimitiveSceneProxy, UpdateParams.WorldBounds, UpdateParams.LocalBounds, UpdateParams.LocalToWorld, UpdateParams.AttachmentRootPosition, UpdateParams.PreviousTransform);
				});
		}
	}
	else
	{
		// ⭐ 아직 Primitive가 SceneProxy를 가지고 있지 않은 경우, 생성해줌. ⭐
		AddPrimitive(Primitive);
	}
}
```
코드가 길지만 핵심은 "ENQUEUE_RENDER_COMMAND(UpdateTransformCommand)" 부분이다.        
게임 스레드에서 렌더스레드가 활용할 데이터(FPrimitiveUpdateParams)를 만든 후 그 데이터를 렌더스레드에 반영하는 동작을 수행하는 코드(람다)와 함께 담아 렌더 스레드에 "ENQUEUE_RENDER_COMMAND" 매크로를 통해 전송해준다.         

------------------------------------------------------                        
                            
위에서 보았듯이 언리얼 엔진4에서는 기본적으로 렌더스레드에서 수행되어야할 동작은 "ENQUEUE_RENDER_COMMAND"라는 매크로를 호출하여 안전하게 렌더스레드로 렌더링 관련 동작을 전송한다.         
게임 스레드에서 이 매크로를 통해 렌더링 관련 명령어를 호출하게 되면 뒤따라 오는 렌더스레드가 데이터 레이스 없이 그 동작을 받아 수행한다.                             

```cpp
template<typename TSTR, typename LAMBDA>
class TEnqueueUniqueRenderCommandType : public FRenderCommand
{
public:
	TEnqueueUniqueRenderCommandType(LAMBDA&& InLambda) : Lambda(Forward<LAMBDA>(InLambda)) {}

	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
}

template<typename TSTR, typename LAMBDA>
FORCEINLINE_DEBUGGABLE void EnqueueUniqueRenderCommand(LAMBDA&& Lambda)
{
	typedef TEnqueueUniqueRenderCommandType<TSTR, LAMBDA> EURCType;

	if (IsInRenderingThread())
	{
		// ⭐ 렌더스레드에서 호출한 경우 즉시 실행. ⭐
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
	else
	{
		if (LIKELY(GIsThreadedRendering || !IsInGameThread())) // !!
		{
			CheckNotBlockedOnRenderThread();
			TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda)); // ⭐ Graph Task System을 통해 렌더스레드의 큐에 삽입 ⭐
		}
		else
		{
			EURCType TempCommand(Forward<LAMBDA>(Lambda));
			FScopeCycleCounter EURCMacro_Scope(TempCommand.GetStatId());
			TempCommand.DoTask(ENamedThreads::GameThread, FGraphEventRef());
		}
	}
}

#define ENQUEUE_RENDER_COMMAND(Type) \
	struct Type##Name \
	{  \
		static const char* CStr() { return #Type; } \
		static const TCHAR* TStr() { return TEXT(#Type); } \
	}; \
	EnqueueUniqueRenderCommand<Type##Name> // ⭐ EnqueueUniqueRenderCommand 함수 호출. ⭐


ENQUEUE_RENDER_COMMAND 사용 예)            

ENQUEUE_RENDER_COMMAND(UpdateAllPrimitiveSceneInfosCmd)([Scene](FRHICommandListImmediate& RHICmdList) {
				// ⭐ 이 람다 코드는 렌더스레드에서만 호출된다. ⭐
				Scene->UpdateAllPrimitiveSceneInfos(RHICmdList);
			});

```                
           
게임 스레드에서 이 매크로를 호출한 경우      
            

```cpp
if (ShouldExecuteOnRenderThread())
{
	CheckNotBlockedOnRenderThread();
	TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));
}
```
이 부분을 통해 렌더 스레드에서 생성할 Task를 만들고, ( 중간에 많은 부분이 생략되었다.. )            

```cpp
void FBaseGraphTask::QueueTask(ENamedThreads::Type CurrentThreadIfKnown)
{
	checkThreadGraph(LifeStage.Increment() == int32(LS_Queued));
	FTaskGraphInterface::Get().QueueTask(this, ThreadToExecuteOn /* ⭐ ENamedThreads::GetRenderThread()이 들어간다. ⭐ */, CurrentThreadIfKnown);
}
```
최종적으로 언리얼의 Task Graph System의 Queue에 Task를 넣게된다. ( TaskGraph쪽도 대걔 볼 내용이 많다..... )                              

```cpp
void FTaskGraphImplementation::QueueTask(FBaseGraphTask* Task, ENamedThreads::Type ThreadToExecuteOn, ENamedThreads::Type InCurrentThreadIfKnown = ENamedThreads::AnyThread)
```

------------------------------------------------------                          

앞에서는 렌더스레드에서 호출되어야 할 동작(Task, Job)를 어떻게 게임 스레드에서 렌더 스레드로 전송하는지에 대해 알아보았다.        
이제는 언리얼 엔진4의 게임 스레드가 언제 게임 스레드에서 렌더 스레드로 데이터를 전송할지를 결정하고, 어떻게 데이터를 Data Race 없이 렌더스레드에 반영하는지에 대해 알아보겠다.           

언리얼 엔진4에서는 게임 스레드쪽 컴포넌트에서 렌더링 관련 변수 값을 수정하는 경우 그 값이 수정되었음을 3가지 변수로 판단을 한다.               


```cpp
/** Is this component in need of its whole state being sent to the renderer? */
uint8 UActorComponent::bRenderStateDirty:1;

/** Is this component's transform in need of sending to the renderer? */
uint8 UActorComponent::bRenderTransformDirty:1;

/** Is this component's dynamic data in need of sending to the renderer? */
uint8 UActorComponent::bRenderDynamicDataDirty:1;

void UActorComponent::MarkRenderStateDirty()
{
	// If registered and has a render state to mark as dirty
	if(IsRegistered() && bRenderStateCreated && (!bRenderStateDirty || !GetWorld()))
	{
		// Flag as dirty
		bRenderStateDirty = true; // !!
		MarkForNeededEndOfFrameRecreate(); // ⭐ EndOfFrame에 이 컴포넌트를 업데이트해야 한다는 것을 UWorld에 알려줌. ⭐         

		MarkRenderStateDirtyEvent.Broadcast(*this); // ⭐ 쓰이지 않으니 무시해도 좋다. ⭐     
	}
}
```
해당 컴포넌트의 렌더링 관련 데이터가 변경되었을시 "UActorComponent::MarkRenderStateDirty" 함수를 호출하여 Render State를 Dirty로 셋팅한다.                      
주목해야할 것은 "MarkForNeededEndOfFrameRecreate()" 함수의 호출이다.          

언리얼 엔진의 경우 게임 스레드에서 렌더링 관련 변수를 수정하면 해당 프레임의 마지막(EndOfFrame)에 이 값을 렌더스레드로 전송한다. ( "MarkForNeededEndOfFrameRecreate" 함수 호출 시점이 아닌 프레임의 마지막에 해당 프레임 동안 업데이트가 필요한 컴포넌트들을 한꺼번에 렌더스레드로 업데이트한다. )                              
"MarkForNeededEndOfFrameRecreate" 함수를 따라 가보면          

```cpp
UWorld::MarkActorComponentForNeededEndOfFrameUpdate(UActorComponent* Component, bool bForceGameThread) 
```
"UWorld::MarkActorComponentForNeededEndOfFrameUpdate" 함수를 통해 World에 EndOfFrame에 이 컴포넌트를 Update를 할 필요가 있다고 알린다.            
그럼 "World::ComponentsThatNeedEndOfFrameUpdate" 변수에 EndOfFrame에 업데이트할 필요가 있는 컴포넌트들을 저장한다.                 
         
여기까지도 아직 게임 스레드의 영역이다.      
          
그럼 Scene 클래스에서 프레임의 마지막에 "UWorld::SendAllEndOfFrameUpdates()" 함수를 호출하여 렌더링 관련 업데이트된 데이터를 렌더스레드에 전송한다.           

```cpp
void UWorld::SendAllEndOfFrameUpdates()
{
	static TArray<UActorComponent*> LocalComponentsThatNeedEndOfFrameUpdate; 
	{
		LocalComponentsThatNeedEndOfFrameUpdate.Append(ComponentsThatNeedEndOfFrameUpdate);
	}

	auto ParallelWork =  // ⭐ 병렬 처리할 작업. ⭐
		[](int32 Index) 
		{
			UActorComponent* NextComponent = LocalComponentsThatNeedEndOfFrameUpdate[Index];
			if (NextComponent)
			{
				if (NextComponent->IsRegistered() && !NextComponent->IsTemplate() && !NextComponent->IsPendingKill())
				{
					NextComponent->DoDeferredRenderUpdates_Concurrent(); // !!
				}
				
				FMarkComponentEndOfFrameUpdateState::Set(NextComponent, INDEX_NONE, EComponentMarkedForEndOfFrameUpdateState::Unmarked); // ⭐ 컴포넌트에서 EndOfFrame Update Flag를 제거함. ⭐
			}
		};

	auto GTWork = 
		[this]()
		{
			for (UActorComponent* Component : ComponentsThatNeedEndOfFrameUpdate_OnGameThread)
			{
				if (Component)
				{
					if (Component->IsRegistered() && !Component->IsTemplate() && !Component->IsPendingKill())
					{
						Component->DoDeferredRenderUpdates_Concurrent();
					}

					check(Component->IsPendingKill() || Component->GetMarkedForEndOfFrameUpdateState() == EComponentMarkedForEndOfFrameUpdateState::MarkedForGameThread);
					FMarkComponentEndOfFrameUpdateState::Set(Component, INDEX_NONE, EComponentMarkedForEndOfFrameUpdateState::Unmarked);
				}
			}
			ComponentsThatNeedEndOfFrameUpdate_OnGameThread.Reset();
			ComponentsThatNeedEndOfFrameUpdate.Reset();
	};

	ParallelForWithPreWork(LocalComponentsThatNeedEndOfFrameUpdate.Num(), ParallelWork, GTWork); 
	// ⭐ 
	// ParallelWork 함수가 여러 스레드에서 병렬로 처리된다. 메인 스레드 ( 게임 스레드 )는 다른 스레드와 함께 ParallelWork를 수행하기 전 GTWork 함수를 먼저 처리한다. 
	// 그 후 ParallelWork 처리를 돕는다. 
	// ⭐                       
	
	LocalComponentsThatNeedEndOfFrameUpdate.Reset();
}
```

이러한 동작들을 통해 언리얼 엔진4는 게임 스레드에서 변경된 렌더링 관련 데이터를 병렬로 렌더스레드에 반영한다.              
"UActorComponent::DoDeferredRenderUpdates_Concurrent"함수를 통해 게임 스레드에서 Dirty한 데이터가 렌더스레드에 반영이 되는데, 이 "UActorComponent::DoDeferredRenderUpdates_Concurrent"함수는 위에서 본 RenderState를 재생성하거나, 변경된 Transform Data를 렌더스레드에 반영하는 동작을 수행한다. ( "UActorComponent::CreateRenderState_Concurrent" 함수를 호출한다. )         


```cpp
void UActorComponent::RecreateRenderState_Concurrent()
{
	if(bRenderStateCreated)
	{
		DestroyRenderState_Concurrent();
	}

	if(IsRegistered() && WorldPrivate->Scene)
	{
		CreateRenderState_Concurrent(nullptr);
	}
}

void UActorComponent::DoDeferredRenderUpdates_Concurrent()
{
	if(bRenderStateDirty)
	{
		RecreateRenderState_Concurrent();
	}
	else
	{
		if(bRenderTransformDirty)
		{
			// Update the component's transform if the actor has been moved since it was last updated.
			SendRenderTransform_Concurrent();
		}

		if(bRenderDynamicDataDirty)
		{
			SendRenderDynamicData_Concurrent();
		}
	}
}
```

-----------------------------------------------------
             
이렇게 이 글에서는 게임 스레드가 렌더 스레드로 렌더링과 관련된 데이터를 어떻게 문제 없이 전송하는지에 대해 알아보았다.       
언리얼 엔진4는 앞에서 보았듯이 게임 스레드에서 사용하는 데이터와, 렌더 스레드에서 사용하는 데이터를 완전히 분리하여서 Data Race를 막고 서로간의 Dependency를 분리하여 두 스레드가 각자 독립적으로 돌아가도록 구현을 하였다.          
              
-------------------------------------------------------
          
조금 더 디테일하게 들어가면 "게임 스레드가 액터를 파괴하였을 때 이 파괴 동작이 곧바로 반영되어 렌더 스레드에 영향을 주는 것( 렌더 스레드는 항상 게임 스레드보다 1, 2 프레임 뒤쳐져서 렌더링을 하기 떄문 )을 막기 위한 동작 등등 여러 추가적인 동작들이 더 있지만 이 글에서는 다루지 못하였다.         
            
------------------------------------------------------
           
이 글에서는 언리얼 엔진4가 렌더링과 관련된 데이터를 어떻게 게임스레드에서 렌더스레드로 안전하게, 성능을 해치지 않고 전달하는지에 대해 알아보았다.                          
그럼 다음에는 게임 스레드로부터 전송된 데이터들을 가지고 렌더스레드가 어떻게 렌더링을 수행하는지에 대해 [이 글](https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/02/26/FMobileSceneRenderer.html)에서 알아보겠다.           
여러 SceneRenderer 중에서 모바일 플랫폼에서 사용되는 "FMobileSceneRenderer"에 대해 집중적으로 탐구를 해볼 예정이다.            
               
-------------------------------------------------------          
                   
추가 참고 자료 : [[UE5] MeshDrawCommand (1/2)](https://scahp.tistory.com/74?category=848072), [[UE5] MeshDrawCommand (2/2)](https://scahp.tistory.com/75?category=848072)        
              
--------------------                  
              
references : [https://ikrima.dev/ue4guide/graphics-development/render-architecture/render-thread-code-flow/](https://ikrima.dev/ue4guide/graphics-development/render-architecture/render-thread-code-flow/), [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [https://twitter.com/sebaaltonen/status/1102783202891104256](https://twitter.com/sebaaltonen/status/1102783202891104256), [https://twitter.com/timsweeneyepic/status/1084336154772680705](https://twitter.com/timsweeneyepic/status/1084336154772680705), [https://twitter.com/benjinsmith/status/1387446710289457157](https://twitter.com/benjinsmith/status/1387446710289457157)                    
