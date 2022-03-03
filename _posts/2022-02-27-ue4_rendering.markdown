---
layout: post
title:  "Unreal Engine4 렌더링 파이프라인 분석"
date:   2022-02-26
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---

- UE4의 렌더링 파이프라인을 살펴보고 싶으면 **FDeferredShadingSceneRenderer::Render**이나, **FMobileSceneRenderer::Render** ( 모바일용 )을 살펴보아라.  
- **렌더링 코드들은 모듈화**되어 렌더링 코드가 바뀌면 렌더링 모듈만 바꿔끼우면 된다. 개발 iteration이 빠름. 렌더링 코드쪽에서는 엔진쪽 코드를 직접적으로 많이 호출하지만 그 반대는 여러 인터페이스를 통함 ( IRendererModule, FSceneInterface )....                   

-------------------------------------                
Scene          

- 씬은 여려 Primitive 컴포넌트들과 다양한 구조들로 이루어짐. Octree가 그 중 하나임 ( 공간 분할에 사용 )       
- **UE4에는 게임 스레드와 렌더링 스레드가 병렬로 동시에 동작**함.             
![20220227031229](https://user-images.githubusercontent.com/33873804/155854426-4f31bac3-8551-4052-91fa-1eeef27bbbad.png)            
- **UPrimitiveComponent ( 게임 스레드 소속 )**가 화면상에 그려지거나, 물리 처리되는 모든 것들의 Base 클래스이다. 컬링이나 여러 렌더링 관련 여러 셋팅 값들을 가짐.
- Primitive Component들이 가시성이나 관련 결정들의 기초 유닛이다. 예를 들면 오클루전, 프러스텀 컬링은 각 Primitive를 기준으로 동작한다. 각 Primitive Component는 컬링, 쉐도우 Casting, 라이트의 영향을 받는지 유무를 결정 등등을 위해 Bound를 가지고 있다. ( 그냥 공간 분할할 때 사용하는 AABB나 Bounding Sphere 생각하면 된다. ).
- 게임 스레드에서 Primitive Component들의 변수 값을 바꾸는 경우 반드시 MarkRenderStateDirty()를 호출해주어야 렌더 스레드에 반영됨.        
- **FPrimitiveSceneProxy** ( 렌더 스레드 소속 ) : UPrimitiveComponent의 렌더 스레드 버전 정도로 생각하면 된다. 마찬가지로 이 클래스 상속받아서 여러 렌더링 기능들을 구현.              
- **FPrimitiveSceneInfo** : 렌더러 모듈에 Private한 Primitive Component 상태이다.                 
- 렌더링 순서는 파이프라인에서 정의한 순서대로 수행됨.           
- **Relevance** : FPrimitiveViewRelevance는 해당 Primitive와 연관된 Pass가 무엇인지에 대한 정보를 가지고 있음.         
- Primitive는 다양한 Relevance를 가진 Elements들을 가지고 있을 수 있다. 그러므로 FPrimitiveViewRelevance는 그 모든 Relevance의 OR 값이다. 이 말은 하나의 Primitive가 불투명, 투명, 정적, 비정적 Relevance와 같은 상호배타적인 Relevance를 동시에 가지고 있을 수도 있다는 말이다.              
- 또한 FPrimitiveViewRelevance는 하나의 Primitive가 Dynamic Rendering Path를 사용해야하는지, Static Rendering Path를 사용해야하는지에 대한 데이터도 가지고 있다.           
- **Drawing Policies** : Drawing Policies는 특정 패스 쉐이더를 가진 메쉬들을 렌더하는 로직을 가짐 ( 예를 들면 Deferred Rendering의 두번째 패스에서 2D 판때기를 그리는 것과 같은 ). 그리고 그러한 메쉬를 추상화하기 위해 FVertexFactory 인터페이스, 메터리얼을 추상화하기 위해 FMaterial을 사용한다. 가장 아랫단에서 Drawing Policy는 Vertex Factory 버퍼, 메터리얼 쉐이더들을 가지고 그것들의 쉐이더, 버텍스 버퍼를 **RHI ( Rendering Hardware Interface )**에 바인딩에서 드로우콜을 날린다.      
![20220227035317](https://user-images.githubusercontent.com/33873804/155855597-87423717-c8bd-498e-8c77-8eea3e4a5fb2.png)            

--------------------------------------------            
Rendering Paths           

- UE4는 순회시 상대적으로 느린 Dynamic Rendering Path와 RHI와 가능한 가까운 순회를 캐싱하는 Static Rendering Path가 존재.          
- 둘다 로우 레벨단에서는 Drawing Policy들을 사용하기 때문에 이 둘의 차이는 윗단에서 생김.          

- **Dynamic Rendering Path** : Dynamic Path를 렌더링에 사용하는 Primitive들의 집합은 FViewInfo::VisibleDynamicPrimitives에 의해 추적됨. 각각의 렌더링 패스는 이 FViewInfo::VisibleDynamicPrimitives Array를 순회한다. 그리고 각각의 Primitive의 Proxy마다 DrawDynamicElements를 호출. Proxy의 DrawDynamicElements 는 그것이 필요한 만큼 FMeshElements를 모으고 GPU에 넘긴다. 
- Dynamic Rendering Path는 각각의 Proxy마다 DrawDynamicElements 호출시 콜백 함수가 호출되어 각각의 컴포넌트 타입에 맞는 로직을 콜백함수에 담아서 호출할 수 있다는 점에서 유연하고 새로운 Primitive를 추가하는 것도 값이 싸다. 그러나 GPU 상태를 정렬하지 않고 ( 같은 상태끼리 뭉치지 않아 상태 변환이 빈번하게 발생해 성능 저하가 있음 ), 캐싱을 해두지 않기 때문에 **순회시 매우 느리다.**           

- **Static Rendering Path** : Static Draw List를 통해 구현된다. 메쉬가 씬에 추가될 때 Draw List에 메쉬가 추가됨. 새로운 메쉬가 추가될 때, 해당 Proxy에 대한 DrawStaticElements가 FStaticMeshElements를 수집하기 위해 호출됨. 새로운 Drawing Policty가 만들어진다. 새로운 Drawing Policy는 그것의 Compare, Matches 함수를 기반으로 Draw List 내의 적절한 위치에 삽입됨. initViews에서 Static Draw List에 대한 가시성 ( 렌더링 될지 안될지 ) 데이터가 셋팅되고, Draw List가 실제로 그려지는 TStaticMeshDrawList::DrawVisible로 전달된다. DrawShared는 서로 일치하는 Drawing Policy 묶음에 대해 오직 한번만 호출됨. SetMeshRenderState, DrawMesh는 각각의 FStaticMeshElement마다 호출됨.            
- Static Rendering Path는 처음 붙을 때 많은 작업을 하는데 이는 순회시 성능 향상을 위함이다. Static Draw List 렌더링은 렌더링 스레드에서 약 3배 정도 빠르다. 왜냐면 Static Draw List는 처음 붙을 때 데이터들을 캐싱하기 떄문이다. 이러한 Static Draw List의 특징 떄문에 한번 삽입되면 왠만해서는 다시 제거될 일이 없고 자주 렌더링 되는 Primitive들에 사용하기 매우 좋다.           
- Static Rendering Path는 씬에서 추가되는 순서나, 렌더링 순서에 영향을 받기 때문에 버그 발생시 디버깅하기 어렵다. 그래서 Lighting only나 Unlit모드로 Primitive들을 강제로 Dynamic Path로 처리했을 때 버그가 없어지면 Drawing Policy의 DrawShared나 Match 함수의 문제라 생각하면된다.       

---------------------------------

- 아래는 FDeferredShadingSceneRenderer의 고레벨 렌더링 Flow이다. FDeferredShadingSceneRenderer::Render함수를 참고하면서 소스 코드를 분석해보아라.             
![20220227040723](https://user-images.githubusercontent.com/33873804/155855981-f3180872-3baa-4a5e-8d95-e97c902ef259.png)             

------------------------------        
Render Hardware Interface (RHI)          

- RHI는 특정 Graphics API 위의 얆은 레이어이다. RHI는 모든 플랫폼에서 동작할 수 있게 짜여져 있다.            
- RHI는 SM5, SM4, ES3_1 세가지 종류가 있고 원하는 플랫폼에 맞추어서 사용해라.          
- 렌더 State들은 그들이 그래픽스 파이프라인 중 어떤 부분에 영향을 주느냐에 따라 Grouping이 되어 있다. 예를들면 RHISetDepthState는 Depth 버퍼링 부분에 영향을 주는 렌더 State들의 묶음이다.        
- 이러한 렌더 State들도 종류가 너무 많기 때문에 매번 다시 셋팅해주는 것은 당연히 바보 같은 짓이다. 그래서 UE4는 자주 사용하는 렌더 State들 중 일부는 기본값으로 가지고 있다.       
- RHISetRenderTargets, RHISetBoundShaderState, RHISetDepthState, RHISetBlendState, RHISetRasterizerState와 같은 것들이 있다. 이름에서 보았을 떄 D3D11에서 많은 영향을 받은 것 처럼 보인다. ( OpenGL에서 사용하는 용어와는 조금 달라보인다. )          

------------------------------------
렌더링 스레드          

- **모든 렌더러들은 렌더링 스레드에서만 동작**한다. 이 **렌더링 스레드는 대개 게임 스레드 보다 한, 두 프레임 뒤쳐져서 동작**한다.          
- 렌더링을 하기 위해 여러 데이터를 쓰거나 읽을 때 렌더링 스레드와 게임 스레드간에 Race condition이 발생하지 않도록 조심해야한다. 이건 디버깅하기도 겁나 어려우니, 두 스레드의 동작을 완전히 이해하는 것이 중요하다.         
- 이러한 Race condition 방지를 위해 UE4는 어떤 특정 스레드에 의해 완전히 소유되는 구조체를 만들어서 사용한다. 
- 예를들어보면 UPrimitiveComponent는 렌더링 관련 데이터를 가지고 있고 렌더링 될 수 있는 것들의 부모 클래스이지만 게임 스레드에 의해 소유된다. 그러므로 렌더링 스레드는 절대 이 UPrimitiveComponent의 데이터를 읽어서도 써서도 안된다. 그렇기 때문에 렌더링 스레드는 FPrimitiveSceneProxy라는 것을 활용하는데 UPrimitiveComponent에 RegisterComponent되어서 씬에 등록될 때 FPrimitiveSceneProxy가 같이 생성된다. 그럼 이 FPrimitiveSceneProxy이 생성되면 ( UPrimitiveComponent가 등록되면 ), 게임 스레드는 FPrimitiveSceneProxy의 데이터를 절대 건들 수 없다. UPrimitiveComponent의 여러 렌더링 관련 셋팅 값들이 변경되면 위에서 말한 것처럼 MarkRenderStateDirty를 호출해주어야 한다.       
- 매 Tick() 함수의 끝에 게임 스레드는 렌더링 스레드가 한, 두 프레임 뒤까지 따라 잡을 떄 까지는 블록된다. **렌더링 스레드가 너무 뒤쳐지기 때문에, 렌더링 스레드가 게임 스레드를 완전히 따라잡을 수 있을만큼 게임 스레드를 블록해서는 안된다. ( 한, 두 프레임 정도는 항상 게임 스레드가 앞서가는 것이 이상적이다. ).** 이를 위해 게임 스레드는 왠만해서는 블록되는 동작을 수행하는 것을 최소화해야한다. ( 렌더 스레드와의 일정한 간격 유지를 위해 ). 그래서 UE4는 게임 스레드가 블록되는 것을 방지하기 위해 다양한 기능들을 제공한다.             

-------------------------------
게임스레드와 렌더스레드간 통신           

[글 참고](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/)           
 


references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
