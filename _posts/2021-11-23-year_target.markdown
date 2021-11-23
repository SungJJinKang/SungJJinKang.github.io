---
layout: post
title:  "2021, 2022년 엔진 구현 목표 ( 2021-11-23 업데이트 )"
date:   2021-11-23
categories: ComputerScience GameEngine
---

2021년 우선 순위

- ~~[clReflect](https://github.com/Celtoys/clReflect)를 이용하여 엔진 리플랙션 기능 구현, 리플랙션용 자체 데이터 컨테이너 타입, 기능 구현.~~  -> 구현 완료
- ~~C#을 이용해서 프로젝트 폴더 분석하고 자동으로 clReflect를 통한 리플랙션 데이터 자동 생성 툴 ( 파이프라인 ) 만들기 ( 프로젝트 파일 분석해서 자동으로 clReflect csv 파일 생성 후 한 파일로 merge까지 )~~  -> 구현 완료     
- DirextX12  공부 후 현재 엔진의 그래픽 API를 DirectX12로 변경 ( 현재는 OpenGL 사용 중 )
- Masked SW Occlusion Culling 구현을 위한 소프트웨어 래스터라이저 구현      
- Masked SW Occlusion Culling 구현 완료 ( [https://github.com/SungJJinKang/EveryCulling/tree/main/CullingModule/MaskedSWOcclusionCulling](https://github.com/SungJJinKang/EveryCulling/tree/main/CullingModule/MaskedSWOcclusionCulling) )


-----------

2021년 차순위

- ~~imgui 연동 ( 위에서 구현한 리플랙션 기능 이용 )~~ -> 구현 완료 [https://youtu.be/wxZIGoTRcpo](https://youtu.be/wxZIGoTRcpo)
- ~~오브젝트 Front to Back Sorting 최적화 ( 지금 너무 느림. 아마도 멀티스레딩으로 해결해볼 예정 )~~ -> [해결!](https://sungjjinkang.github.io/computerscience/2021/10/12/MultiThread_SortFrontToBack.html)                 
- ~~빌드 시간 단축 ( 지금 너무 느림 )~~ -> Thank you! [Resharper](https://www.jetbrains.com/help/resharper/Analyzing_Includes.html#includees-view)          
- ~~물리, 충돌 처리 성능 개선~~ -> 구현 완료
- 오브젝트 렌더링 Bathcing 기능 구현

------------------------

2022년


- Hierachy Shadow Map
- 나만의 자료구조 만들기 ( EASTL 같이 할당자를 런타임에 전달해서 할당자가 달라도 같은 타입으로 간주되게 구현, 내 리플렉션 시스템 적용하려면 어차피 자체적으로 컨테이너들 만들어야함 )          
- 나만의 메모리 풀 ( 할당자 ) 만들기
- 메가 텍스쳐 ( 메모리 맵 IO 이용 )
- 포트나이트 느낌의 Imposter 구현, 관련 에디터 툴도 구현 ( [https://shaderbits.com/blog/octahedral-impostors](https://shaderbits.com/blog/octahedral-impostors) )

