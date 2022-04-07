---
layout: post
title:  "Masked SW ( CPU ) Occlusion Culling 구현기"
date:   2021-12-31
categories: ComputerScience ComputerGraphics
---

미루고 미루다가 드디어 Masked SW ( CPU ) Occlusion Culling 구현을 끝냈다..         
6개월 전부터 구현하다가 인턴 활동, 학교 생활 때문에 미루다가 종강을 하고 구현을 마쳤다.         
생각보다 빠르게 구현한 것 같다.        

----------------

Occlusion Culling이란 쉽게 말하면 **다른 오브젝트에 의해 가려져 어차피 안보일 오브젝트는 그리지 않는** 기술이다.     
이를 위해서 먼저 Occluder ( 다른 오브젝트를 가리는 오브젝트 )를 선정해서 Depth Buffer에 Depth값을 쓴 후 Occludee의 Depth 값을 앞서 그린 Depth Buffer와 비교하여 해당 Occludee를 그릴지 말지를 결정한다.        

Occlusion Culling은 GPU를 사용하여 수행할 수도 있고 오로지 CPU에서만 수행할 수도 있다.          
GPU에서는 Occlusion Query라고 하는 방법이 있는데 쉽게 얘기하면 Occluder를 아주 간단한 쉐이더로 그린 후 ( Depth Buffer를 그릴 목적이니 Fragment Shader는 그냥 아무 색이나 찍으면 된다. ) Occludee를 그리면서 Depth Buffer와 비교한다. 여기서 조금 더 빠르게 동작하기 위해서 Occludee를 온전히 그리지 않고 Occludee의 AABB를 그려 해당 Occludee가 가려지는지 아닌지를 값싼 비용으로 테스트할 수 있다.        

다만 **GPU를 사용하는 방법**은 한계가 있는데 결국 Occludee가 그려질지를 테스트한 후 그 결과를 다신 CPU로 가져와야하는데 이걸 가져오는 **시간이 많이걸린다.** 그만큼 느려진다는 것이다. ( PCI버스는 레이턴시가 매우 높다. 다만 대역폭은 높다. )            

그래서 대부분의 엔진에서는 이 Occlusion Culling을 **CPU에서 수행**한다. **CPU에서 Occluder의 삼각형을 일일이 그려가면서 Depth Buffer을 쓰고 테스트**를 하는 것이다. GPU로 데이터가 가기 전 CPU에서 매우 일찍 컬링을 하는 것이다.                       
다만 CPU는 GPU만큼 이러한 엄청난 양의 연산에 특화되어 있지 않다보니 약간의 트릭을 사용하는데 바로 Depth Buffer를 여러 사이즈로 만들어 두고 Occludee를 비교하는 것이다. 일명 **HI(Hierarchical)-Z Occlusion Culling**이다                                
예를 들면 1920x1080이 게임의 해상도라면 일단은 전체 해상도 버전의 Dpeth 버퍼를 그리고, 그 Depth 버퍼를 축소하여서 여러 해상도의 Depth Buffer를 만들어 두는 것이다.                    
이를 위해 Depth Buffer를 축소할 때는 축소되는 Fragment들 중 최대 Depth 값을 축소된 버전의 Fragment의 Depth 값으로 채택한다.             
안그려도 될 ( 가려진 ) 오브젝트를 그리는 것 ( GPU로 Draw 명령어 전송 )은 괜찮지만, 그려야할 오브젝트를 그리지 않는 일은 발생하면 안되기 때문이다.      
그래서 **Depth Buffer는 최대 Depth 값을 취하고 Occludee는 최소 Depth 값을 취해서 이 둘을 비교**한다. **Occludee의 최소 Depth값이 Depth Buffer의 최대 Depth 값보다 크다면 그 Occludee는 Occluder에 의해 완전히 가려진 것이 확실**하다.         
이러한 

----------------------------------------

서론이 너무 길었다. 바로 **Masked SW ( CPU ) Occlusion Culling**에 대해 설명하겠다.                 
Masked SW ( CPU ) Occlusion Culling의 가장 큰 특징은 **멀티스레드 활용**, **SIMD 명령어 활용**이다.

Masked SW ( CPU ) OC과 흔히 사용하는 **HI-Z Occlusion Culling과의 가장 큰 차이**는 무엇일까?       
우선 **Masked OC은 Depth Buffer가 한개**이다. HI-Z 방식이 전체 해상도로 Depth Buffer를 하나 그리고 그 Depth Buffer를 축소한 여러 해상도의 Depth Buffer를 만드는데 비해 Masked OC은 8x4 픽셀당 하나의 Depth 값을 가진다.         
또한 다른 큰 차이점은 **Masked OC은 해당 8x4 픽셀 타일 ( 전체 해상도를 8x4 사이즈로 각각 나눔 )이 다른 삼각형에 의해 완전히 덮였다고 판단이되면 최대 Depth 값을 업데이트**하는 것이다. 이는 **HI-Z 방식이 일단은 전체 해상도를 그려야하는데 여기서 오는 엄청난 양의 연산을 줄이기 위함이다.**                  

**HI-Z OC에서 오류가 발생하지 않기 위해서는 우선 모든 Fragment들에 Depth 값을 알아야(그러야)한다.** 그래야 축소된 버전의 Dpeth Buffer에서 줄어든 Fragment들 중 최대 Depth 값을 취해 그려야할 오브젝트를 그리지 않는 오류를 방지할 수 있기 떄문이다.             

반면 Masked OC는 **각각의 8x4 타일이 삼각형에 덮여있는지를 판단하는 "Coverage Mask"라는 것이 존재**한다. 각 타일이 삼각형에 의해 완전히 덮혀있는지를 체크하여 완전히 덮혀있는 경우에만 Depth Buffer에 Depth 값을 쓰는 것이다.                    
이를 통해 일반적인 HI-Z OC가 전체 해상도의 Depth Buffer를 Rasterize해야하는데 반해 Masked OC는 HI-Z Buffer ( 낮은 해상도의 버퍼 )를 직접 연산하고 그럼에도 불구하고 줄어든 Fragment들의 최대 Depth 값을 취한다는 성질을 유지한다.             

( Coverage Mask를 쓸 때 SIMD 연산을 활용해서 32x8 총 256개의 Fragment가 삼각형에 의해 덮여져 있는지 여부를 몇개의 SIMD만으로 매우 빠르게 기록한다. )                  
자세한 알고리즘은 이 글 [Masked Occlusion Culling 알고리즘 한글 설명](https://github.com/SungJJinKang/EveryCulling/blob/main/CullingModule/MaskedSWOcclusionCulling/MaskedSWOcclusionCulling_HowWorks.md)을 참고하라.          

--------------------------------

여기서부터는 Masked OC의 각 단계를 설명하겠다.           


**첫번째 단계**는 Occluder를 선정하는 작업이다.                     
이 부분은 논문에 다루는 부분이 아니어서 필자가 고안하였다.         
오브젝트의 **AABB를 Screen Space로 Project한 후 화면 상 차지하는 넓이를 기준**으로 정하였다. 그 넓이가 일정 크기보다 큰 경우 Occluder로서의 가치가 있다고 판단하고 Occluder로 사용한다.        
여기서도 Cache를 최대한 활용하기 위해서 오브젝트들의 AABB 데이터를 연속되게 저장하였다.                
( Screen Space AABB를 멀티스레드로 연산하는데 여기서 스레드들간의 Cache Coherency에 의한 성능 저하를 방지하기 위한 데이터 구조가 따로 있는데 여기서 굳이 설명하지는 않겠다. )              
**사실 이 단계에서 Occlusion Culling의 성능이 모두 결정된다.** 매우 매우 중요한 단계이다.         
왜냐면 **Occluder를 너무 많이 선정해버리면 Occluder를 Depth Buffer에 Rasterize하기 위한 연산시간으로 인해 오히려 Occlusion Culling시 더 성능이 느려질 수 있다.** 또한 Occluder로서 가치가 없는 오브젝트를 선택해버리면 Rasterize하는데 시간만 버리고 실제 Occlude에 도움이 되지 않을 수 있다.         

                           
                             
**두번째 단계**는 **Occluder를 Bin한다**       
Bin한다는 것은 Occluder의 각각의 삼각형이 그려질 32x8 ( 8x4 타일을 8개 뭉쳐 ) 타일에 삼각형 데이터를 넣어두는 것이다.    
만약에 이 동작이 없다면 멀티스레드로 Occluder을 그릴 때 큰 성능 저하가 발생할 수 밖에 없다. 왜냐면 **여러 스레드들이 각자 맡은 삼각형의 Depth 값을 동시에 동일한 Depth Buffer에 쓰려면 당연히 Race condition이 발생할 수 있으니 불가피하게 lock을 걸어야하는데 이는 엄청난 성능 저하**를 가져온다. 렌더링 코드에서 lock과 같은 느려터진 동작은 성능에 매우 치명적이다.            
그렇기 때문에 전체 Depth Buffer를 일정한 크기로 나눈 타일( 32x8 )들에 그 타일에 그려질 삼각형을 미리 저장해두는 것이다. 그 후 **각각의 스레드들은 자신이 연산할 타일을 하나 정해 그 타일에 대해서만 삼각형을 그리는 연산을 하는 것**이다. 스레드들은 서로 다른 타일에 Depth값을 쓰니 당연히 Race condition 상황은 발생하지 않고 이를 통해 성능을 향상 시킬 수 있다.         
또한 Bin을 할 때는 가까운 오브젝트부터 Bin을 한다. Masked OC에서는 가까운 Occluder부터 Rasterize하여야 정확도가 높아지기 때문에 Sorting된 순서대로 Bin을 수행하여야 Rasterize 단계에서 Sorting된 순서대로 Rasterize를 수행할 수 있다.          
                    
                  
**세번째 단계**는 드디어 **Occluder들을 Depth Buffer에 그리는 단계 ( Rasterize )**이다.       
앞서 말했듯이 Occluder들은 각 삼각형들이 그려질 32x8 타일에 Screen Space 값으로 저장이 되어 있고 각 스레드들은 이 32x8 타일을 하나씩 맡아서 연산을 한다. 그럼 각 스레드들은 해당 32x8 타일에 삼각형을 그리고 해당 타일들의 최대 Depth 값을 쓰는 동작을 한다. ( 각 스레드가 서로 다른 타일에 대한 연산을 수행하니 당연히 Data Race가 발생하지 않고 스레드간 동기화도 전혀 없다. )             

32x8 타일은 아래와 같은 데이터 구조를 가지고 있다. 타일간의 Cache Coherency를 방지하기 위해 캐시라인에 Align되게 옵션을 추가하였다.          

```c++
struct HizData
{
    float L0MaxDepthValue // 32x8 타일의 최대 Depth 값
    culling::M256F L0SubTileMaxDepthValue; // L1 Max Depth Value. 32x8 타일 내부의 8x4 8개 서브 타일의 최대 L0 Depth 값
    culling::M256F L1SubTileMaxDepthValue; // L1 Max Depth Value. 32x8 타일 내부의 8x4 8개 서브 타일의 최대 L1 Depth 값
    culling::M256I L1CoverageMask; // CoverageMask
}

struct TriangleList
{
    // 두번째 Bin 단계에서 저장된 Screen space 삼각형들의 List
    alignas(32) float VertexX[3][BIN_TRIANGLE_CAPACITY_PER_TILE]; // SIMD 연산을 위해 SIMD 레지스터 사이즈에 align되게 설정.
    alignas(32) float VertexY[3][BIN_TRIANGLE_CAPACITY_PER_TILE];
    alignas(32) float VertexZ[3][BIN_TRIANGLE_CAPACITY_PER_TILE];
    size_t mCurrentTriangleCount = 0;
};

struct alignas(64) Tile // 하나의 32x8 타일에 대한 데이터
{
    std::uint32_t mLeftBottomTileOrginX = 0xFFFFFFFF;
    std::uint32_t mLeftBottomTileOrginY = 0xFFFFFFFF;
    
    HizData mHizDatas;
    TriangleList mBinnedTriangles;
}
```

이 단계는 약간 복잡하기 때문에 몇개의 사진으로 대체하겠다.        
**최대 Depth 값 업데이트 알고리즘.**                           

<img width="463" alt="1" src="https://user-images.githubusercontent.com/33873804/147820303-dcb21217-291b-427a-8312-9805e4fc0667.png">     
<img width="463" alt="2" src="https://user-images.githubusercontent.com/33873804/147820312-2736a009-c9e8-46c8-9374-81a78a261aee.png">      
<img width="463" alt="3" src="https://user-images.githubusercontent.com/33873804/147820321-58283b22-18d1-4c80-9601-645c2971fd5c.png">      
<img width="463" alt="4" src="https://user-images.githubusercontent.com/33873804/147820328-ce1faf11-1512-42f3-8752-c138d022f47e.png">      
<img width="463" alt="5" src="https://user-images.githubusercontent.com/33873804/147820336-a2e56563-229c-4253-810d-aef3d2f82674.png">      
                        
                         
**Z0 Max 값을 업데이트해도 될지를 판정하기 위한 Coverage Mask를 연산하는 과정. ( SIMD 명령어 활용 )**                
<img width="461" alt="3" src="https://user-images.githubusercontent.com/33873804/147820503-28bc7273-02af-4af9-9419-ecb09559943f.png">      
<img width="461" alt="4" src="https://user-images.githubusercontent.com/33873804/147820513-98a7149a-7ef9-412d-a67d-68900e0928c2.png">     
<img width="461" alt="5" src="https://user-images.githubusercontent.com/33873804/147820517-4c71f10c-4502-4b44-94ed-28ce26fd77e5.png">     
<img width="461" alt="6" src="https://user-images.githubusercontent.com/33873804/147820539-99a0bb35-915c-483e-9124-79b2d223e35b.png">      
<img width="461" alt="7" src="https://user-images.githubusercontent.com/33873804/147820544-10edf741-4446-4318-bea6-8d1f5ac7b390.png">      
<img width="461" alt="8" src="https://user-images.githubusercontent.com/33873804/147820545-cd2c1bed-0fea-4ada-ab65-f275f7a3272a.png">     
                       
                        

정말 많은 곳에서 SIMD 명령어를 활용하여 최적화를 하기 위해서 노력하였다. **논문에는 대략적인 알고리즘만 제시되어 있지 구현부분은 정확히 제시를 하고 있지 않기 때문에 필자가 최대한 빠르게 동작을 하기 위해서 어떻게 구현을 해야할지 고민을 많이하여 코드를 작성** 하였다.    
또한 빠르게 동작해야하는 렌더링 코드에서 많은 분기(Branch)는 CPU 파이프라인 측면에서 성능 저하를 불러오기 때문에 최대한 분기를 줄이기 위해 노력하였다.                       
            
             
**네번째 단계**는 **Depth Buffer와 오브젝트들의 Depth 값을 비교하면서 월드의 오브젝트들이 Culled될지를 결정하는 작업**이다.                   
이 부분도 논문에서 제안하는 방법이 없었기 때문에 따른 자료를 참고하였다. ( [https://www.slideshare.net/DICEStudio/culling-the-battlefield-data-oriented-design-in-practice](https://www.slideshare.net/DICEStudio/culling-the-battlefield-data-oriented-design-in-practice) 52페이지 참고 )                           
Frostbite 엔진에서 참고한 방식으로 **오브젝트들의 AABB의 각 Vertex의 Depth 값을 가지고 비교**를 하는 것이다.                  
물론 이 방법이 정확도면에서는 떨어진다. 실제 오브젝트는 Cull되지만 그 오브젝트의 AABB는 당연히 그 오브젝트보다 더 크니 Cull되지 않는다고 판단될 수 있기 때문이다. 정확도는 떨어질 수 있지만 **적어도 False Positive(Cull)는 발생하지 않는다.** 오브젝트의 AABB가 Cull된다면 그보다 작은 오브젝트는 당연히 Cull될 것이다. **Cull될 오브젝트를 그리는건 괜찮지만 그려져야할 오브젝트를 Cull하면 절대 안된다**는 것을 이 방법은 아주 잘 충족한다. 이 방법은 정확도를 약간 버리지만 성능은 좋은 매우 괜찮은 방법이라고 생각한다.                               


-----------------------------

Masked SW ( CPU ) Occlusion Culling를 통해 성능 향상을 이룰 수 있었다.               
또한 멀티스레드를 적극 활용한다는 점에서 개발 중인 엔진의 지향점에도 잘 맞다.           

성능 테스트를 해보니 참 묘한 결과가 나온다.          
흔히 GPU Bound한 경우 성능 향상이 크다. 반면 그렇지 않은 경우 성능 향상이 거의 없다.           
필자가 생각하기에 요즘 게임들은 대부분 GPU Bound하기 때문에 SW Occlusion Culling의 유용성 분명히 존재한다고 생각한다.        

조금 더 복잡한 씬에서의 성능 테스트도 할 예정이다.            
성능에 관한 부분은 아래 영상을 참조하기 바란다.          
[영상](https://youtu.be/4xJu6_Jtbu4)                      

위의 영상들의 돌과 같은 오브젝트들은 5000개의 가까운 Vertex를 가지고 있기 때문에 당연히 SW Rasterization이 느리다.            
즉 영상에 나온 오브젝트들은 Occlusion Culling의 Occluder로 사용하기에는 매우 나쁜 오브젝트라고 할 수 있다.          

그래서 실제로 개발사들은 디자이너가 의도적으로 Occluder를 선택하거나, 일정 개수 이하의 Vertex를 가진 Mesh에 대해서만 Occlusion Culling을 수행한다.           
실제로 포트나이트 모바일에서도 Occlusion Culling을 수행하는데 [영상](https://youtu.be/3E9ks-FV3Is?t=1586)에서 나오는 것과 같이 Vertex 개수가 적은 오브젝트들을 Occluder로 사용한다.        

필자의 경우에도 안전장치를 만들어두었는데 Rasterization을 계속 수행하다 일정 개수 이상의 삼각형을 그리면 거기서 Rasterization을 멈춘다. Rasterization을 무한히 할 수 없으니 어느 정도 선에서 멈추는 것이다.        

---------------------------------------------

이후 **2, 3 단계인 Occluder Bin 단계와 Rasterize 단계가 너무 느리다고 판단**되어서 ( 두 단계는 매우 느리고, 두 단계의 연산 소요 시간은 비슷하다. ) **두 단계를 2 프레임에 걸쳐서 번갈아 수행**되게 바꾸었다.                                   
현재 프레임에서 Occluder Bin 단계만을 수행하면, 그 다음 프레임에 이전 프레임의 Bin 결과를 가지고 Rasterize 단계를 수행하는 것이다.        
결과적으로는 Depth Buffer에 1 프레임 딜레이가 발생하여서 Occludee 테스트시 약간의 오차가 발생할 수 있다.           
**이를 통한 얻어지는 성능 향상을 생각하면 1 프레임의 오차는 용인 가능**하다 생각한다.                       



reference : [https://www.intel.com/content/dam/develop/external/us/en/documents/masked-software-occlusion-culling.pdf](https://www.intel.com/content/dam/develop/external/us/en/documents/masked-software-occlusion-culling.pdf), [https://www.slideshare.net/IntelSoftware/masked-occlusion-culling](https://www.slideshare.net/IntelSoftware/masked-occlusion-culling), [https://www.slideshare.net/IntelSoftware/masked-software-occlusion-culling](https://www.slideshare.net/IntelSoftware/masked-software-occlusion-culling)         
