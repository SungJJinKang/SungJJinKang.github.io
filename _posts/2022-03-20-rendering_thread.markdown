---
layout: post
title:  "자체 게임 엔진에서 게임 스레드, 렌더링 스레드 분리하기 ( 개발 중 )"
date:   2022-03-20
categories: ComputerScience ComputerGraphics
---

언리얼 엔진4에서는 CPU를 게임 스레드, 렌더링 스레드로 분리하여 렌더링을 수행한다.      
GPU까지 해서 **CPU의 게임 스레드, 렌더 스레드(Draw 스레드), GPU 3개의 스레드가 동시에 동작**한다.           
게임 스레드가 먼저 연산을 하면 다음 프레임에 렌더링 스레드가 이전 프레임의 게임 스레드의 데이터를 가지고 동작을 수행하고, 한 프레임 후 GPU가 렌더링을 수행한다.          
                                
<img width="560" alt="7" src="https://user-images.githubusercontent.com/33873804/159112192-be363c0d-f72b-490c-9612-ca1cf6579e14.png">      
           
약간은 생소한 개념인데 **게임 스레드**는 흔히 에니메이션, 오브젝트의 Transform 데이터, 물리 연산 처리, AI 연산 처리 등등 **게임 로직과 관련된 전반적인 연산들을 수행하는 스레드**이다. 게임 스레드는 CPU에서 수행된다. 후에 렌더 스레드가 활용할 여러 렌더링 관련 변수 값 셋팅도 게임 스레드에서 수행한다. 예를 들면 게임 스레드에서 Distance Culling을 위한 어떤 오브젝트의 Max Desired Draw Distance 값을 수정하면 이 값을 다음 프레임에 렌더 스레드가 Distance Culling을 수행하는데 사용한다.                                      

**렌더 스레드**도 대부분 CPU에서 수행된다. 렌더 스레드에서는 **각종 가시성 연산을 수행**한다. 쉽게 말하면 컬링 연산을 수행한다. 현대 게임들은 대부분 GPU Bound한 경우가 많기 때문에 최대한 GPU의 연산 부담을 덜어주기 위해 CPU에서 렌더링 할 필요가 없는 오브젝트들을 걸러내기 위한 연산을 수행한다. Distance 컬링, 뷰 프러스텀 컬링, (SW) 오클루전 컬링, PreComputed Visibility 등이 있다. 대부분 CPU에서 수행된다. 필자는 이미 [EveryCulling](https://github.com/SungJJinKang/EveryCulling)이라는 라이브러리로 Distance 컬링, 뷰 프러스텀 컬링, CPU 오클루전 컬링은 멀티스레드로 처리하는 기능을 모듈화해두었다.                                   
또한 렌더 스레드에서는 **렌더링 커맨드를 생성하여 GPU에 제출**한다. 흔히 **그래픽스 API를 호출하는 곳이 이 렌더 스레드**라고 생각하면 된다.                   

마지막으로 **GPU**는 그냥 GPU이다. 렌더 스레드의 커맨드를 받아서 GPU 연산을 수행한다.          
                    
D3D12, Vulkan에 와서는 RHI 스레드라는 것을 만들어서 렌더 스레드에서는 플랫폼 독립적인 커맨드 큐를 CPU쪽에 생성해두었다가, RHI 스레드에서 목표로 하는 플랫폼(Graphics API)에 맞는 그래픽 API를 호출해준다. 그러나 필자의 게임 엔진은 D3D11, OpenGL만을 지원하기 때문에 그래픽 API를 호출하는 부분도 렌더 스레드에 넣을 생각이다.             
                
참고 자료 : [Unreal Engine4 렌더링 파이프라인 분석 - SungJJinKang](https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/02/26/ue4_rendering.html), [그래픽 프로그래밍 개요](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/Overview/), [스레디드 렌더링](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/ThreadedRendering/), [병렬 렌더링 개요](https://docs.unrealengine.com/4.27/ko/ProgrammingAndScripting/Rendering/ParallelRendering/)            
            
-------------------------------------------------           
             
사실 이렇게 게임 스레드와 렌더 스레드를 분리한다고 해도 얼마나 성능이 향상될지는 알 수 없다.                
언리얼 엔진4 정도의 수준까지 구현을 하는 것은 당연히 불가능하기 때문에 **오히려 싱글 스레드에서 모두 연산을 하는 것보다 더 느릴수도 있다.**                    
그렇지만 **언리얼 엔진4의 렌더링이 어떻게 동작하는지를 공부할 겸**해서 [내 게임 엔진](https://github.com/SungJJinKang/DoomsEngine)에서 구현을 해보면 언리얼 엔진을 이해하는데 큰 도움이 될 것 같아 이러한 결정을 하게 되었다.          
            
----------------------------------------------------           
            
게임 스레드와 렌더 스레드를 분리하게 되면 당연히 가장 큰 문제는 **Data Race**이다.                                                
그렇다고 무턱대고 **Mutex로 Lock을 해버리면 겁나게 느려진다.......**                
게임 스레드와 렌더 스레드가 동시에(!) 쉬지(Idle 상태) 않고 도는 것이 중요하다. 한 스레드가 다른 스레드를 멈추게 구현하면 안된다.         

--------------------------------------------------           

언리얼 엔진4의 구현을 분석하는 것부터 시작해서 내 엔진에 구현을 해보겠다.        
언리얼 엔진4에서는 기본적으로 렌더스레드에서 수행되어야할 동작은 "ENQUEUE_RENDER_COMMAND"라는 매크로를 통해 수행한다.           
```
template<typename TSTR, typename LAMBDA>
FORCEINLINE_DEBUGGABLE void EnqueueUniqueRenderCommand(LAMBDA&& Lambda)
{
	QUICK_SCOPE_CYCLE_COUNTER(STAT_EnqueueUniqueRenderCommand);
	typedef TEnqueueUniqueRenderCommandType<TSTR, LAMBDA> EURCType;

#if 0 // UE_SERVER && UE_BUILD_DEBUG
	UE_LOG(LogRHI, Warning, TEXT("Render command '%s' is being executed on a dedicated server."), TSTR::TStr())
#endif
	if (IsInRenderingThread())
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
	else
	{
		if (ShouldExecuteOnRenderThread())
		{
			CheckNotBlockedOnRenderThread();
			TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));
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
	EnqueueUniqueRenderCommand<Type##Name>
```

언리얼 엔진4에서 렌더링 관련 변수 값을 수정하는 경우 그 값이 수정되었음을 뒤따라 오는 렌더링 스레드에 알려주기 위해 3가지 변수를 사용한다.       

```
/** Is this component in need of its whole state being sent to the renderer? */
uint8 UActorComponent::bRenderStateDirty:1;

/** Is this component's transform in need of sending to the renderer? */
uint8 UActorComponent::bRenderTransformDirty:1;

/** Is this component's dynamic data in need of sending to the renderer? */
uint8 UActorComponent::bRenderDynamicDataDirty:1;
```
UActorComponent::bRenderStateDirty는 

```
void UActorComponent::MarkRenderStateDirty()
{
	// If registered and has a render state to mark as dirty
	if(IsRegistered() && bRenderStateCreated && (!bRenderStateDirty || !GetWorld()))
	{
		// Flag as dirty
		bRenderStateDirty = true;
		MarkForNeededEndOfFrameRecreate();

		MarkRenderStateDirtyEvent.Broadcast(*this);
	}
}
```
 




references : [https://ikrima.dev/ue4guide/graphics-development/render-architecture/render-thread-code-flow/](https://ikrima.dev/ue4guide/graphics-development/render-architecture/render-thread-code-flow/), [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [https://twitter.com/sebaaltonen/status/1102783202891104256](https://twitter.com/sebaaltonen/status/1102783202891104256), [https://twitter.com/timsweeneyepic/status/1084336154772680705](https://twitter.com/timsweeneyepic/status/1084336154772680705), [https://twitter.com/benjinsmith/status/1387446710289457157](https://twitter.com/benjinsmith/status/1387446710289457157)                    