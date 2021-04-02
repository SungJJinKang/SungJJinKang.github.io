---
layout: Normalt
title:  "ViewFrustum Culling(렌더링 빨리 하기)"
date:   2021-04-02
categories: Doom
---

어떻게 하면 렌더링을 빨리 할 수 있을 까?? 다양한 연구와 자료들을 공부하였다.    
여러 방법들이 있지만 대표적으로 Culling이라는 기법이 있다.     
쉽게 말해서 그릴 필요가 없는 것은 안그린다는 것이다.    
렌더링이 느려지는 원인은 결국 Graphics API 콜이다.           
그래서 최대한 Graphics API 콜을 줄이는 것이는 것이 렌더링 속도에 가장 중요하다.     

그 중에서도 View Frustum Culling과 Occlusion Culling이 있다.    
이번 글에서는 ViewFrustum Culling에 대해 소개해보고자 한다.       

보통 검색을 하면 나오는 대표적인 방법이 BVH와 같은 Accleration Strucuture을 이용한 기법이다.    
쉽게 말해서 각 엔티티들을 위치에 따라 트리 형태로 분리하는 것이다.     
예를 들어보면 Entity A(0.0, 0.0, 0.0)과 B(1.0, 0.0, 0.0), C(100.0, 100.0, 100.0) 이 세개의 Entity가 존재한다고 가정해보자.    
이때 A, B를 묶어서 하나의 Entity로 취급한다고 해보자. 그러면 이 하나로 묶은 Entity가 카메라 Frustum안에 들어가는지만 확인하면 A, B Entity를 그릴지 안그릴지를 판단할 수 있다.     
이렇게 위치를 기반으로 여러 Entity를 묶어서 화면에 그릴지를 판단하면 렌더링 속도를 높일 수 있다.(월드 상 대부분의 Entity는 그려지지 않는 경우가 많기 때문이다.) ( 사실 세세한 내용을 더 말할 수 있지만 여기서는 매우 간단하게 쉬운 예를 들어서 설명하였다. 따로 찾아 보기 바랍니다. )     


하지만 필자는 위에 나타난 BVH같은 Accleration Structure가 아닌 다른 방법을 사용하려고 한다.     

이 방법은 EA의 대표 게임엔진인 [Frostbite엔진 개발자가 발표한 내용](https://www.ea.com/frostbite/news/culling-the-battlefield-data-oriented-design-in-practice)으로 멀티스레드를 이용하는 방법이다.(아직도 이 방법을 사용하는지는 모른다).    
이 방법의 가장 핵심적인 요소는 두가지 이다.       
1. 게임 내 데이터들을 Linear(연속되게)하게 배치하여서 캐시 Hit률을 높이고 거기에 더해 SIMD사용하여 성능 향상을 노리는 것이다.    
2. 게임 내 Entity들을 EntityBlock내에 일정한 개수만큼 나누어서 이 Entity묶음(Block)들을 서브 스레드들에 넘겨주어 해당 묶음 내 Entity들의 Culling여부를 결정하는 연산을 한다. ( Data race가 생기지 않아 매우 빠르게 연산 가능 )

우선 1번째 요소에 대해 살펴보자.    
SIMD 사용에 최적화된 형태로 데이터들을 Linear하게 배치한 후 SIMD 연산을 통해 연산속도를 최대한으로 끌어내는 방법이다.      
SIMD에 대해서는 [내가 쓴 글](https://sungjjinkang.github.io/c++/2021/03/22/SIMD.html)을 참고해보기 바란다.    
SIMD 명령어는 적게는 128bit의 데이터가 연속되어 있어야 하는 데 float형(4byte)으로는 4개의 float 데이터가 연속되어 있어야 한다.     

그 중에서도 Frostbite의 View Frustum Culling의 핵심 연산부분을 소개해 보겠다. 
```
Frustum Plane1-NormalX      Frustum Plane2-NormalX      Frustum Plane3-NormalX      Frustum Plane4-NormalX
Frustum Plane1-NormalY      Frustum Plane2-NormalY      Frustum Plane3-NormalY      Frustum Plane4-NormalY
Frustum Plane1-NormalZ      Frustum Plane2-NormalZ      Frustum Plane3-NormalZ      Frustum Plane4-NormalZ
  Frustum Plane1-Dist        Frustum Plane2-Dist         Frustum Plane3-Dist         Frustum Plane4-Dist

    Ojbect PointX               Ojbect PointX               Ojbect PointX               Ojbect PointX
    Ojbect PointY               Ojbect PointY               Ojbect PointY               Ojbect PointY
    Ojbect PointZ               Ojbect PointZ               Ojbect PointZ               Ojbect PointZ
          1                           1                           1                            1
   
                                      |
                                      |
                                      V
   
SIMD MultiPly Each Row Computation And Add All Row --> Dot Result of Each Plane and Object
```

위에 보면 Frustum Plane1- NormalX, Frustum Plane2-NormalX 이게 무슨 의미냐 하면 카메라 Frustum의 각 Plane(면)의 Normal값을 Linear하게 배치한 것이다.    
메모리 배치 상으로 보면 일반적으로는 아래와 같이 배치하는 경우가 많다.
```
Plane1 NormalX, Plane1 NormalY, Plane1 NormalZ, Plane1 Distance, Plane2 NormalX, Plane2 NormalY, Plane2 NormalZ, Plane2 Distance....
```
각 Plane 별로 한 Plane의 데이터를 다 배치하고 그 다음에 다음 Plane의 데이터를 배치하는 이런식이다.      
그렇지만 이 방법은 그렇게 SIMD연산에 최적화되어 있지 못하다.     
생각을 해보자 우선 Entity의 Point가 Frustum Plane 안쪽에 있는지 확인하기 위해서는 Dot연산이 필요하다.(왜 Dot연산이 필요한지는 [이 글](https://cgvr.informatik.uni-bremen.de/teaching/cg_literatur/lighthouse3d_view_frustum_culling/index.html)을 참고하기 바란다.)     
데이터가 Plane1 NormalX, Plane1 NormalY, Plane1 NormalZ, Plane1 Distance와 같이 배치되어 있는 경우를 보자
```
Plane1 NormalX   Plane1 NormalY   Plane1 NormalZ   Plane1 Distance     
Ojbect PointX    Ojbect PointY    Ojbect PointZ          1

Plane2 NormalX   Plane2 NormalY   Plane2 NormalZ   Plane2 Distance     
Ojbect PointX    Ojbect PointY    Ojbect PointZ          1
```
언뜻 보기에는 그냥 Plane과 Object Point의 각 Component를 곱해서 더 하면 될 것 같다.           
그렇지만 이 경우에는 Plane1 NormalX * Ojbect PointX + Plane1 NormalY * Ojbect PointY + Plane1 NormalZ * Ojbect PointZ + Plane1 Distance 이렇게 각 Component를 더하는 연산이 필요하다. 6개의 Plane에 대해 EntityPoint의 Dot연산을 구하고 일일이 양수인지 확인 하는것은 그렇게 빠르지 않아보인다.            

반면 아래의 SIMD 사용을 극대화한 코드를 보자.                    

```
Plane1 NormalX    Plane2 NormalX    Plane3 NormalX    Plane4 NormalX     
Plane1 NormalY    Plane2 NormalY    Plane3 NormalY    Plane4 NormalY     
Plane1 NormalZ    Plane2 NormalZ    Plane3 NormalZ    Plane4 NormalZ     
Plane1 Distance   Plane2 Distance   Plane3 Distance   Plane4 Distance     

Ojbect PointX     Ojbect PointX     Ojbect PointX      Ojbect PointX
Ojbect PointY     Ojbect PointY     Ojbect PointY      Ojbect PointY
Ojbect PointZ     Ojbect PointZ     Ojbect PointZ      Ojbect PointZ
      1                 1                 1                  1

                            |
                            |
                            V
                                    
Plane1 NormalX * Ojbect PointX    Plane2 NormalX * Ojbect PointX    Plane3 NormalX * Ojbect PointX    Plane4 NormalX * Ojbect PointX     
Plane1 NormalY * Ojbect PointY    Plane2 NormalY * Ojbect PointY    Plane3 NormalY * Ojbect PointY    Plane4 NormalY * Ojbect PointY
Plane1 NormalZ * Ojbect PointZ    Plane2 NormalZ * Ojbect PointZ    Plane3 NormalZ * Ojbect PointZ    Plane4 NormalZ * Ojbect PointZ
     Plane1 Distance * 1               Plane2 Distance * 1               Plane3 Distance * 1               Plane4 Distance * 1   

                            |
                            |
                            V
                        
Plane1 Dot Object      Plane2 Dot Object      Plane3 Dot Object      Plane4 Dot Object
Plane5 Dot Object      Plane6 Dot Object      Plane5 Dot Object      Plane6 Dot Object

                            |
                            |   If Dot is larget than 0
                            V
                  1      1      1      1
                  1      1      1      1

                            |
                            |   Result
                            V

                        Culled!!!!!
```

```c++
inline char CheckInFrustumSIMDWithTwoPoint(math::Vector<4, float>* eightPlanes, const math::Vector<4, float>* twoPoint)
{
		//We can't use M256F. because two twoPoint isn't aligned to 32 byte

	const M128F* m128f_eightPlanes = reinterpret_cast<const M128F*>(eightPlanes); // x of plane 0, 1, 2, 3  and y of plane 0, 1, 2, 3 
	const M128F* m128f_2Point = reinterpret_cast<const M128F*>(twoPoint);

	M128F posA_xxxx = M128F_REPLICATE(m128f_2Point[0], 0); // xxxx of first twoPoint and xxxx of second twoPoint
	M128F posA_yyyy = M128F_REPLICATE(m128f_2Point[0], 1); // yyyy of first twoPoint and yyyy of second twoPoint
	M128F posA_zzzz = M128F_REPLICATE(m128f_2Point[0], 2); // zzzz of first twoPoint and zzzz of second twoPoint
		
	M128F posA_rrrr = M128F_REPLICATE(m128f_2Point[0], 3); // rrrr of first twoPoint and rrrr of second twoPoint

	M128F dotPosA = M128F_MUL_AND_ADD(posA_zzzz, m128f_eightPlanes[2], m128f_eightPlanes[3]);
	dotPosA = M128F_MUL_AND_ADD(posA_yyyy, m128f_eightPlanes[1], dotPosA);
	dotPosA = M128F_MUL_AND_ADD(posA_xxxx, m128f_eightPlanes[0], dotPosA); // dot Pos A with Plane 0, dot Pos A with Plane 1, dot Pos A with Plane 2, dot Pos A with Plane 3

	M128F posB_xxxx = M128F_REPLICATE(m128f_2Point[1], 0); // xxxx of first twoPoint and xxxx of second twoPoint
	M128F posB_yyyy = M128F_REPLICATE(m128f_2Point[1], 1); // yyyy of first twoPoint and yyyy of second twoPoint
	M128F posB_zzzz = M128F_REPLICATE(m128f_2Point[1], 2); // zzzz of first twoPoint and zzzz of second twoPoint
		
	M128F posB_rrrr = M128F_REPLICATE(m128f_2Point[1], 3); // rrrr of first twoPoint and rrrr of second twoPoint

	M128F dotPosB = M128F_MUL_AND_ADD(posB_zzzz, m128f_eightPlanes[2], m128f_eightPlanes[3]);
	dotPosB = M128F_MUL_AND_ADD(posB_yyyy, m128f_eightPlanes[1], dotPosB);
	dotPosB = M128F_MUL_AND_ADD(posB_xxxx, m128f_eightPlanes[0], dotPosB); // dot Pos B with Plane 0, dot Pos B with Plane 1, dot Pos B with Plane 2, dot Pos B with Plane 3

	//https://software.intel.com/sites/landingpage/IntrinsicsGuide/#expand=69,124,4167,4167,447,447,3148,3148&techs=SSE,SSE2,SSE3,SSSE3,SSE4_1,SSE4_2,AVX&text=insert
	M128F posAB_xxxx = _mm_shuffle_ps(m128f_2Point[0], m128f_2Point[1], SHUFFLEMASK(0, 0, 0, 0)); // x of twoPoint[0] , x of twoPoint[0], x of twoPoint[1] , x of twoPoint[1]
	M128F posAB_yyyy = _mm_shuffle_ps(m128f_2Point[0], m128f_2Point[1], SHUFFLEMASK(1, 1, 1, 1)); // y of twoPoint[0] , y of twoPoint[0], y of twoPoint[1] , y of twoPoint[1]
	M128F posAB_zzzz = _mm_shuffle_ps(m128f_2Point[0], m128f_2Point[1], SHUFFLEMASK(2, 2, 2, 2)); // z of twoPoint[0] , z of twoPoint[0], z of twoPoint[1] , z of twoPoint[1]
		
	M128F posAB_rrrr = _mm_shuffle_ps(m128f_2Point[0], m128f_2Point[1], SHUFFLEMASK(3, 3, 3, 3)); // r of twoPoint[0] , r of twoPoint[1], w of twoPoint[1] , w of twoPoint[1]

	M128F dotPosAB45 = M128F_MUL_AND_ADD(posAB_zzzz, m128f_eightPlanes[6], m128f_eightPlanes[7]);
	dotPosAB45 = M128F_MUL_AND_ADD(posAB_yyyy, m128f_eightPlanes[5], dotPosAB45);
	dotPosAB45 = M128F_MUL_AND_ADD(posAB_xxxx, m128f_eightPlanes[4], dotPosAB45);

	dotPosA = _mm_cmpgt_ps(dotPosA, posA_rrrr); // if elemenet[i] have value 1, Pos A is in frustum Plane[i] ( 0 <= i < 4 )
	dotPosB = _mm_cmpgt_ps(dotPosB, posB_rrrr); // if elemenet[i] have value 1, Pos B is in frustum Plane[i] ( 0 <= i < 4 )
	dotPosAB45 = _mm_cmpgt_ps(dotPosAB45, posAB_rrrr);

	M128F dotPosA45 = _mm_blend_ps(dotPosAB45, dotPosA, SHUFFLEMASK(0, 3, 0, 0)); // Is In Plane with Plane[4], Plane[5], Plane[2], Plane[3]
	M128F dotPosB45 = _mm_blend_ps(dotPosB, dotPosAB45, SHUFFLEMASK(0, 3, 0, 0)); // Is In Plane with Plane[0], Plane[1], Plane[4], Plane[5]
		
	M128F RMaskA = _mm_and_ps(dotPosA, dotPosA45); //when everty bits is 1, PointA is in frustum
	M128F RMaskB = _mm_and_ps(dotPosB, dotPosB45);//when everty bits is 1, PointB is in frustum

	int IsPointAInFrustum = _mm_test_all_ones(*reinterpret_cast<__m128i*>(&RMaskA)); // value is 1, Point in in frustum
	int IsPointBInFrustum = _mm_test_all_ones(*reinterpret_cast<__m128i*>(&RMaskB));
		
	char IsPointABInFrustum = IsPointAInFrustum | (IsPointBInFrustum << 1);
	return IsPointABInFrustum;
}
```


그리고 Frostbite View frustum Culling 방법의 두 번째 특징으로 게임 내 Entity들을 여러 EntityBlock으로 나누어서 각 EntityBlcok들을 서브 스레드들에 넘겨서 Culling 연산을 시키는 것이다.     
여기서 중요한 것은 각 Entity들이 EntityBlock으로 분리되어 나누어져있기 때문에 Data race가 발생하지 않는다는 것이다.     
각 스레드가 서로 데이터를 동기화할 필요가 없어 그 만큼 속도가 매우 빠르다. ( 물론 마지막에는 한번 동기화를 해주어야 한다. )        

구현부분을 살짝보자면 아래와 같다.    

게임 월드 내 모든 Entity의 Position과 Bounding Sphrere의 Radius 정보 이렇게 총 4개의 floating data가 Linear하게 배치되어 한 Block을 구성한다. 중요한건 Cache Hitting률을 높이기 위해 SoA 구조로 데이터를 배치한다는 것이다.     
```c++
struct CullingBlock
{
    alignas(32) math::Vector4 mPositions[ENTITY_COUNT_IN_ENTITY_BLOCK];
    alignas(32) char mIsVisibleBitflag[ENTITY_COUNT_IN_ENTITY_BLOCK];
    EntityHandle mHandles[ENTITY_COUNT_IN_ENTITY_BLOCK];
    TransformData mTransformDatas[ENTITY_COUNT_IN_ENTITY_BLOCK];
}
```
위와 같은 CullingBlock들이 여러개 존재한다. 그럼 각 스레드에 이 CullingBlock을 넘겨주어 Culling 연산을 하면 된다.    

실험결과 렌더링 루프에서 최대한 앞쪽에 스레드들이 CullingBlock을 연산하는 작업을 넘겨주면 실제 오브젝트가 Cull 됬는지를 판단하는 부분쯤에서는 거의 모든 Culling연산이 끝나있다. ( 그 사이에 UniformBuffer나 GBuffer 바인딩 등 Entity를 그리기 위한 준비 작업들을 넣어주면 된다. ) ( 서브 스레드들이 CullingBlock 연산을 끝내기를 기다리는 경우가 거의 드물다. ).    
그럼 mIsVisibleBitflag을 통해서 해당 Entity가 그려질지 판단하고 Entity를 그리는 Graphics API 콜을 호출하면 된다. ( 이 때도 CullingBlock별로 mIsVisibleBitflag를 체크함으로서 Cache hitting률을 높이려 노력하자. )


[구현 Github](https://github.com/SungJJinKang/LinearTransformData_CullingSystem.git)     