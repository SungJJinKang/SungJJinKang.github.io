---
layout: post
title:  "View Frustum Culling, Occlusion Culling 여정"
date:   2021-03-19
categories: Doom
---

어떻게 하면 렌더링을 빨리 할 수 있을 까?? 다양한 연구와 자료들을 공부하였다.    
후술할 내용들은 내가 공부한 결과이고 얼마나 렌더링이 빨라졌는지를 보여 줄 것이다.

우선 View Frustum 

BSP 등 Acceleration Sturcuture가 매우 중요하다. 일반 Tree 만들 듯이 Pointer을 이용해서 Node들을 연결하면 매우매우 느리다. 특히 새로 추가되는 노드를 new를 통해 새로운 힙 영역에 allocate하는 것은 정말 정신나간 짓이다.
모든 노드들을 array형태로 Configous하게 Allocate해서 Cache hit 가능성을 최대한 높임
SIMD로 최적화 -> SIMD를 이용하려면 모든 오브젝트들의 AABB 혹은 Sphere 등등 Culling 테스트를 위한 데이터들이 Vectorize 되어 있어야함 ( 서로 Packed 되게 인접해 있어야 한다 ) Matrix4X4 * Matrix4X4랑 Matrix4X4 * Vector4 연산은 내가 만든 수학 라이브러리에서 SIMD 명령어를 사용해 이미 최적화 되어있음

Hierarchy하게 오브젝트들 렌더링 전 순회를 돌면서 오브젝트가 그려질지 안그려질지를 미리 정해둠. Tree 구조에서 조상 Node가 그려지지 않으면 그 조상 Node의 모든 자식 Node들은 Culling Test를 할 필요도 없이 그려지지 않은 것으로 처리.


Occlusion Culling : 쉽게 생각하면 어떤 오브젝트들을 그리려고 할 때 그 오브젝트의 Data를 GPU에 넘기고, State 셋팅까지 다 했는 데 결국 Depth test를 통과하지 못하였다고 생각해보자. 그럼 Depth Test를 그리는 과정까지의 수 많은 Vertex Shading, Geometry Shading, Vertex Post Processing, Primitive Assembly, Rasterization 등등 수 많은 그래픽 파이프 라인 과정[OpenGL Graphic Pipline](https://www.khronos.org/opengl/wiki/Rendering_Pipeline_Overview)들이 헛수고이 된 것이다. ( 물론 Early Depth Test에서 걸러지는 경우는 덜 하다. ) 이 얼마나 시간 낭비를 한 것인지....... 그래서 Occlusion Culling은 Depth test를 통과 못할 오브젝트들을 미리 걸러주는 것이라고 할 수 있다.      
어떻게? 그 오브젝트 똑같은 위치에 아주 단순한 그냥 상자 단순한 쉐이더를 가지고 미리 그려보자. 그 상자가 depth test를 통과하였다? 그럼 복잡한 오브젝트를 그려도 된다는 뜻이다. 

Occlusion Culling는 GPU가 할 수도 CPU가 할 수도 있다. 전자의 경우 HW Occlusion Culling, 후자는 SW Occlusion Culling이라 한다.   
필자는 두 방법 모두 사용할 것이다.


reference :      
https://cgvr.informatik.uni-bremen.de/teaching/cg_literatur/lighthouse3d_view_frustum_culling/index.html     
https://www.gdcvault.com/play/1014491/Culling-the-Battlefield-Data-Oriented     
https://www.slideshare.net/DICEStudio/culling-the-battlefield-data-oriented-design-in-practice     
https://webpages.uncc.edu/krs/courses/5010/ged/lectures/cull_lod1.pdf       
https://www.slideshare.net/dgtman/sw-occlusion-culling     
https://computergraphics.stackexchange.com/questions/7828/difference-between-bvh-and-octree-k-d-trees         
Unreal Engine 소스코드