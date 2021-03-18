---
layout: post
title:  "View Frustum Culling, Occlusion Culling 여정"
date:   2021-03-19
categories: Doom
---

어떻게 하면 렌더링을 빨리 할 수 있을 까?? 다양한 연구와 자료들을 공부하였다.    
후술할 내용들은 내가 공부한 결과이고 얼마나 렌더링이 빨라졌는지를 보여 줄 것이다.

같은 텍스쳐, 메쉬끼리는 묶어서 렌더링 하자(텍스쳐들 Bind하는 비용 줄어듬)

Occlusion Culling : 쉽게 생각하면 어떤 오브젝트들을 그리려고 할 때 그 오브젝트의 Data를 GPU에 넘기고, State 셋팅까지 다 했는 데 결국 Depth test를 통과하지 못하였다고 생각해보자. 그럼 Depth Test를 그리는 과정까지의 수 많은 Vertex Shading, Geometry Shading, Vertex Post Processing, Primitive Assembly, Rasterization 등등 수 많은 그래픽 파이프 라인 과정[OpenGL Graphic Pipline](https://www.khronos.org/opengl/wiki/Rendering_Pipeline_Overview)들이 헛수고이 된 것이다. ( 물론 Early Depth Test에서 걸러지는 경우는 덜 하다. ) 이 얼마나 시간 낭비를 한 것인지....... 그래서 Occlusion Culling은 Depth test를 통과 못할 오브젝트들을 미리 걸러주는 것이라고 할 수 있다.      
어떻게? 그 오브젝트 똑같은 위치에 아주 단순한 그냥 상자 단순한 쉐이더를 가지고 미리 그려보자. 그 상자가 depth test를 통과하였다? 그럼 복잡한 오브젝트를 그려도 된다는 뜻이다. 

Occlusion Culling는 GPU가 할 수도 CPU가 할 수도 있다. 전자의 경우 HW Occlusion Culling, 후자는 SW Occlusion Culling이라 한다.   
필자는 두 방법 모두 사용할 것이다.