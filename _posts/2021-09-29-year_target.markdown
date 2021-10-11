---
layout: post
title:  "2021년 엔진 구현 목표"
date:   2021-09-29
categories: ComputerScience
---

우선 목표 

- Masked SW Occlusion Culling 구현 완료
- 오브젝트 Front to Back Sorting 최적화 ( 지금 너무 느림. 아마도 멀티스레딩으로 해결해볼 예정 )
- ~~빌드 시간 단축 ( 지금 너무 느림 )~~ -> Thank you! [Resharper](https://www.jetbrains.com/help/resharper/Analyzing_Includes.html#includees-view)          
- 간단한 물리, 충돌 처리
- DX11 적용
- 오브젝트 렌더링 Bathcing 기능 구현

------------------------

가능하면 

- Hierachy Shadow Map
- 나만의 자료구조 만들기 ( EASTL 같이 할당자를 런타임에 전달해서 할당자가 달라도 같은 타입으로 간주되게 구현 )          
- 나만의 메모리 풀 ( 할당자 ) 만들기
- 메가 텍스쳐 ( 메모리 맵 IO 이용 )

