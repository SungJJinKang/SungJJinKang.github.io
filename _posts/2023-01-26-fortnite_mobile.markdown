---
layout: post
title:  "A Year in a Fortnite"
date:   2023-01-26
tags: [UE]
---
               
OpenGLES에서 적용할 수(도움이 될만한) 있을만한 것들만 정리해보면..              
                     
----------------------------------------------                     
                              
![20230127023943](https://user-images.githubusercontent.com/33873804/214918655-784eea93-a25f-4af8-b5fc-670b433e7a12.png)          
언리얼 기본 동작도 페이지를 곧바로 OS로 반환하지는 않았던 것으로 기억하는데 이 풀 사이즈가 64MB보다 작았는지는 모르겠음...           
                     
----------------------------------------------                     
                              
![20230127024212](https://user-images.githubusercontent.com/33873804/214918661-51a1d62b-1e8a-4335-9e93-76ff93e6035f.png)          
아마 이 내용은 현재 OpenGLES, Vulakn쪽 모두 이미 적용되어 있는 것으로 기억함.            
                     
----------------------------------------------                     
                              
![20230127023823](https://user-images.githubusercontent.com/33873804/214918668-3c0e12bb-00a0-4344-b357-52e81d4acbd2.png)             
"Optimizing Resources"가 뭘 의미하는지 모르겠음.. 나중에 확인해보아야 함.         
                     
----------------------------------------------                     
                              
![20230127023846](https://user-images.githubusercontent.com/33873804/214918672-15c24fce-d020-40e4-9d47-3fe2799715cb.png)        
데칼이 화면 상에 보이느냐에 따라 Depth-Stencil State를 달리했는데, 이로 인해 관련 PipelineStateObject 개수가 두 배가 됨.              
데칼이 화면 상에 보이느냐에 따라 Depth-Stencil State를 구분하는 동작을 없애 필요한 PSO 개수를 절반으로 줄였다는 얘기.              
Vulkan이나 Metal 같은 최신 Graphics API는 하나의 드로우 콜에 필요한 모든 State(쉐이더, Depth Test 정책, Depth Write 정책, Blend 정책, 등등)들의 조합을 하나의 PSO로 생성(컴파일)한 후 캐싱하여 이후에 재사용한다.                          
그러나 안드로이드 기기의 특성상 생성되어 있는(캐싱 해둔) PSO의 개수가 일정 개수 이상이 되면 렌더링이 비정상적으로 동작함(GPU 드라이버에서 PSO를 위한 힙 메모리 사이즈를 제한해둠).           
그래서 언리얼 엔진은 이 제한된 사이즈를 넘지 않기 위해 엔진 코드단에서 PSO의 개수가 일정 개수 이상이 되면 가장 오래 전에 사용된(LRU) PSO를 축출(파괴)하는 동작을 수행함.         
포트나이트에서는 사용되는 PSO의 개수가 많아 PSO 캐시의 캐시 사이즈를 넘는 상황이 자주 발생했고, 이로 인해 캐싱해둔 PSO가 축출되는 상황이 발생. 이후 축출된 PSO가 다시 필요해져 한 프레임에 많은 수의 PSO가 생성되는 경우가 발생하였고, 이는 곧 히치로 이어졌음.(PSO를 생성하는 비용이 작지 않음)                                                              
데칼이 화면 상에 보이느냐에 Depth-Stencil State를 구분하는 것이 하나의 드로우 콜의 관점에서는 최적일지는 몰라도(불필요한 연산을 제거해서), 결과적으로는 더 많은 히치를 유발하게 되었고(Depth-Stencil을 구분하기 위해 PSO가 추가로 필요해졌고 LRU PSO 캐시 정책으로 PSO가 재사용되지 못하고 파괴되었다가 다시 생성됨) 이를 완화하기 위해 데칼 여부에 따라 Depth-Stencil State를 구분하는 동작을 제거하였다고 한다...                      
       
--------------------- 

PSO에 대한 추가 설명         
                  
Vulkan이나 Metal 같은 최신 Graphics API는 하나의 드로우 콜에 필요한 모든 State(쉐이더, Depth Test 정책, Depth Write 정책, Blend 정책, 등등)들의 조합을 하나의 PSO로 생성한 후 캐싱하여 이후에 재사용한다. 하나의 드로우콜을 위한 각종 State들을 한번의 커맨드로 한방에 셋팅할 수 있다는 장점이 있지만(CPU쪽 비용 감소를 가져옴), 셋팅된 PSO를 너무 자주 변경할 경우 이로 인한 비용이 발생한다(CPU쪽 비용뿐만아니라, GPU에서도 State가 바뀌면 바뀌기 전 State를 기반으로 생성한 드로우들이 끝날 때까지 GPU가 대기해야함)         
 
반면 OpenGL의 경우 일일이 별도의 커맨드로 설정을 해주어야 함.(OpenGL이 최신 API들보다 CPU쪽 비용이 큰 이유 중 하나)                
그래서 언리얼쪽 PSO 캐시 구현만 보아도 OpenGL의 경우에는 쉐이더만을 기준으로 PSO를 분류하지만(쉐이더 하나당 PSO도 하나), 최신 API(Vulkan, Metal)의 경우 쉐이더+각종 State들의 조합을 하나의 PSO로 분류한다.      
```
Ex)                   
1번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less(현재 Depth 값보다 작은 경우 Depth Test 통과) DepthMask는 false(Depth 테스트를 통과해도 Depth 버퍼에 Depth를 쓰지는 않음),              
2번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less, DepthMask는 true,                         
3번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less Equal DepthMask는 false"라고 하자.              
               
언리얼 엔진에서 위의 세 오브젝트를 렌더링했을 때 몇 개의 PSO개 생성될까?             
OpenGL의 경우 1개이다. 쉐이더만을 기준으로 PSO를 분류하기 떄문이다.        
Vulkan, Metal의 경우에는 3개이다. 사용되는 쉐이더는 같지만 다른 State들의 조합이 3경우 모두 다르기 때문이다.         
```
                     
----------------------------------------------                     
                              
references : [https://developer.samsung.com/galaxy-gamedev/gamedev-blog/fortnite.html](https://developer.samsung.com/galaxy-gamedev/gamedev-blog/fortnite.html)
