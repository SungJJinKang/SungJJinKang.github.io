---
layout: post
title:  "Non-Temporal Memory Hint"
date:   2021-06-11
categories: ComputerScience
---

얼마전 인터뷰에서 SIMD 중에 "_mm_stream_pd"에 대한 질문을 받았다. 여러 SIMD 명령어를 사용해보았지만 이건 처음 들어본 명령어다. 그래서 공부해보았고 새롭게 알게 된 내용에 대해 글을 써보겠다.       

인텔에서 해당 명렁어를 검색해보면 아래와 같은 설명이 나온다.     
```
Store 128-bits (composed of 2 packed double-precision (64-bit) floating-point elements) from a into memory using a non-temporal memory hint. mem_addr must be aligned on a 16-byte boundary or a general-protection exception may be generated.

"a non-temporal memory hint"을 사용하여 메모리에 128비트를 저장한다 
```

"a non-temporal memory hint" ??            
이게 도대체 뭔 말인가. 생전 처음 들어본다.          
그래서 스택 오버 플로우에 검색해보았고 이것이 무엇인지 말해보겠다.          

이 말은 "시간적 지역성"을 무시하라는 것이다.          
일반적으로 우리가 메모리에서 데이터를 가져올 때(LOAD)는 그 데이터가 이후 근시간내에 다시 사용될 것(시간적 지역성)이라고 예상하고 해당 데이터를 CPU 캐시에 복사해둔다.(이에 대해 이해가 가지 않는다면 [이 글](https://sungjjinkang.github.io/computerscience/2021/04/01/cachefriendly.html)을 읽어보기 바란다.)      

그런데 만약 프로그래머가 해당 데이터가 근시간 내에 사용될 가능성이 없다면 어떻게 되는가? 그러면 **이 근시간 내에 사용되지 않을 데이터를 캐시에 저장하기 위해 현재 캐시에 있는 어떤 데이터가 해당 캐시에서 쫒겨나 다음 단계의 캐시나 메모리로 옮겨져야한다. 근시간 내에 사용되지 않은 데이터를 캐시에 저장하기 위해 기존의 데이터를 쫒아낸다는 것이 얼마나 비효율적인 일인가.**                           
그래서 "a non-temporal memory hint"란 쉽게 말해 **해당 데이터를 캐시에 저장하지 말라**는 것이다.         

이렇게 "a non-temporal memory hint"란 **LOAD시에는 메모리에서 읽어온 데이터를 캐시에 저장하지 않고 WRITE시에는 캐시에 쓰지 않고 곧바로 메모리에 쓰라**는 것이다. 


references : [https://stackoverflow.com/a/37092/7138899](https://stackoverflow.com/a/37092/7138899)      