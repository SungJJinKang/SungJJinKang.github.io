---
layout: post
title:  "2021, 2022년 엔진 구현 목표"
date:   2021-09-29
categories: ComputerScience
---

2021년

- [clReflect](https://github.com/Celtoys/clReflect)를 이용하여 엔진 리플랙션 기능 구현. 
- C#을 이용해서 프로젝트 폴더 분석하고 자동으로 clReflect를 통한 리플랙션 데이터 자동 생성 툴 만들기 ( 프로젝트 파일 분석해서 자동으로 clReflect csv 파일 생성 후 한 파일로 merge까지 )
- imgui 연동 ( 위에서 구현한 리플랙션 기능 이용 )
- Masked SW Occlusion Culling 구현 완료
- ~~오브젝트 Front to Back Sorting 최적화 ( 지금 너무 느림. 아마도 멀티스레딩으로 해결해볼 예정 )~~ -> [해결!](https://sungjjinkang.github.io/computerscience/2021/10/12/MultiThread_SortFrontToBack.html)                 
- ~~빌드 시간 단축 ( 지금 너무 느림 )~~ -> Thank you! [Resharper](https://www.jetbrains.com/help/resharper/Analyzing_Includes.html#includees-view)          
- ~~물리, 충돌 처리 성능 개선~~

------------------------

2022년

- 포트나이트 느낌의 Imposter 구현, 관련 에디터 툴도 구현 ( [https://shaderbits.com/blog/octahedral-impostors](https://shaderbits.com/blog/octahedral-impostors) )
- Hierachy Shadow Map
- 나만의 자료구조 만들기 ( EASTL 같이 할당자를 런타임에 전달해서 할당자가 달라도 같은 타입으로 간주되게 구현 )          
- 나만의 메모리 풀 ( 할당자 ) 만들기
- 메가 텍스쳐 ( 메모리 맵 IO 이용 )
- 오브젝트 렌더링 Bathcing 기능 구현

