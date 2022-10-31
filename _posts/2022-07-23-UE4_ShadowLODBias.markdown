---
layout: post
title:  "Unreal Engine Shadow LOD Bias ( 작성 중 )"
date:   2022-07-23
categories: UE UnrealEngine ComputerScience ComputerGraphics
---         
                
최근에 여러 렌더링 최적화 테크닉들을 공부하던 중 펍지 뉴 스테이트에서 적용한 최적화 기법을 보았다.         
간단히 말하면 Shadow Map을 그릴 때 Mesh들의 LOD를 높은 단계를 적용하여 Vertex 수를 줄여 Shadow Pass의 비용을 줄이는 것이다.          
펍지 뉴 스테이트는 UE4의 "ForceLODShadow" 기능을 사용하여 Shadow Pass에서 LOD 단계를 고정적으로 적용하였다.         
![55](https://user-images.githubusercontent.com/33873804/180611658-42c14282-fbe6-4287-9dc2-929a0923873e.png)            
               
다만 이 "ForceLODShadow" 기능은 모든 Mesh에 일괄적으로 LOD 레벨을 적용하기 때문에 자칫 Light와 가까운 Mesh의 Shadow 품질이 나쁠 수 있다는 문제가 있다.        

그래서 Shadow Pass 전용 LOD Bias를 구현해보면 어떨까 생각해보았다.       
구현해 볼 Shadow Pass LOD Bias는 당연히 Dynamic Shadow에 한해 작동한다. ( Static Shadow는 빌드 타임에 쉐도우 맵을 만들어버리니 굳이 높은 단계의 LOD를 적용할 필요가 없다...... )        
           
언리얼 엔진4의 엔진 코드 수정이 필요하다.               
                                   
----------------------------------------------
                                   
살펴볼 함수들....                                   
                                   
FMobileSceneRenderer::InitViews                                   
->                                   
FMobileSceneRenderer::InitDynamicShadows                                   
->                                   
FSceneRenderer::InitDynamicShadows                                   
->                                   
FSceneRenderer::GatherShadowDynamicMeshElements                                   
->                                   
FProjectedShadowInfo::GatherDynamicMeshElements                                   
->                                   
FProjectedShadowInfo::GatherDynamicMeshElementsArray                                   
->                                   
FPrimitiveSceneProxy::GetDynamicMeshElements                                   
->                                   
FProjectedShadowInfo::SetupMeshDrawCommandsForShadowDepth                                   
                                   
                                   
FProjectedShadowInfo::AddSubjectPrimitive                                   
->                                   
FProjectedShadowInfo::ShouldDrawStaticMeshes                                   
->                                   
FProjectedShadowInfo::AddCachedMeshDrawCommandsForPass                                   
                                   
                                   
FSceneRenderer::RenderShadowDepthMaps                                   
->                                   
FSceneRenderer::RenderShadowDepthMapAtlases                                   
->                                   
FProjectedShadowInfo::RenderDepth                                   
->                                   
FProjectedShadowInfo::RenderDepthInner                                   
->                                   
FParallelMeshDrawCommandPass::DispatchDraw                                   
                                   
                                   
references : [https://scahp.tistory.com/92](https://scahp.tistory.com/92), [https://scahp.tistory.com/74](https://scahp.tistory.com/74), [https://scahp.tistory.com/75](https://scahp.tistory.com/75), [https://developer.android.com/stories/games/new-state-mobile](https://developer.android.com/stories/games/new-state-mobile)          