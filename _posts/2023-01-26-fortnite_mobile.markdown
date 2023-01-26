---
layout: post
title:  "A Year in a Fortnite"
date:   2023-01-26
categories: UE UnrealEngine ComputerScience ComputerGraphics
---
               
OpenGLES에서 적용할 수(도움이 될만한) 있을만한 것들만 정리해보면..              
               
                    
![20230127023943](https://user-images.githubusercontent.com/33873804/214918655-784eea93-a25f-4af8-b5fc-670b433e7a12.png)          
언리얼 기본 동작도 페이지를 곧바로 OS로 반환하지는 않았던 것으로 기억하는데 이 풀 사이즈가 64MB보다 작았는지는 모르겠음...           
                 
                              
![20230127024212](https://user-images.githubusercontent.com/33873804/214918661-51a1d62b-1e8a-4335-9e93-76ff93e6035f.png)          
아마 이 내용은 현재 OpenGLES, Vulakn쪽 모두 이미 적용되어 있는 것으로 기억함.            
                         

![20230127023823](https://user-images.githubusercontent.com/33873804/214918668-3c0e12bb-00a0-4344-b357-52e81d4acbd2.png)             
"Optimizing Resources"가 뭘 의미하는지 모르겠음.. 나중에 확인해보아야 함.         
                              
                     
![20230127023846](https://user-images.githubusercontent.com/33873804/214918672-15c24fce-d020-40e4-9d47-3fe2799715cb.png)            
Vulkan이나 Metal 같은 최신 Graphics API는 렌더링에 필요한 모든 State(쉐이더, Depth Test 정책, Depth Write 정책, Blend 정책, 등등)을 한꺼번에 GPU에 제출할 수 있는 것으로 앎.       
반면 OpenGL의 경우 일일이 별도의 커맨드로 설정을 해주어야 함.(OpenGL이 최신 API들보다 CPU쪽 비용이 큰 이유 중 하나)                
그래서 언리얼쪽 PSO 캐시 구현만 보아도 OpenGL의 경우에는 쉐이더를 기준으로 PSO를 분류하지만(쉐이더 하나당 PSO도 하나), 최신 API(Vulkan, Metal)의 경우 쉐이더+각종 State들의 조합을 하나의 PSO로 분류한다.      
```
Ex)                   
1번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less(현재 Depth 값보다 작은 경우 Depth Test 통과) DepthMask는 false(Depth 테스트를 통과해도 Depth 버퍼에 Depth를 쓰지는 않음),              
2번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less, DepthMask는 true,                         
3번 오브젝트를 렌더링하는데 쉐이더 A를 사용하고 Depth Test는 Less Equal DepthMask는 false"라고 하자.              
               
언리얼 엔진에서 위의 세 오브젝트를 렌더링했을 때 몇 개의 PSO개 생성될까?             
OpenGL의 경우 1개이다. 쉐이더만을 기준으로 PSO를 분류하기 떄문이다.        
Vulkan, Metal의 경우에는 3개이다. 사용되는 쉐이더는 같지만 다른 State들의 조합이 3경우 모두 다르기 때문이다.         
```
           
           

references : [https://developer.samsung.com/galaxy-gamedev/gamedev-blog/fortnite.html](https://developer.samsung.com/galaxy-gamedev/gamedev-blog/fortnite.html)