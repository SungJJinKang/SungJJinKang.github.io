---
layout: post
title:  "lvalue, rvalue의 정확한 개념적 이해"
date:   2021-08-23
categories: C++
---


상수 10은 레지스터나 메모리에 저장되어 있는 것이 아니라 그냥 어셈블리 명령어 코드 자체에 상수 숫자 데이터가 들어가 있는 것이다. 

그럼 어떻게 const T&, lvalue 레퍼런스에는 rvalue를 전달할 수 있는 것인가. 이 또한 rvalue를 이해하면 쉬운데 rvalue라는 것은 상수로 바뀌어서는 안되는 수이다. 그러므로 T&는 바뀔 가능성이 있으나 const T&는 const로 보호를 해주어서 그 값이 바뀔 가능성이 없으므로 const T&에는 rvalue를 전달할 수 있는 것이다. ( 물론 이 모든 것은 컴파일러단(C++)에서 정한 규칙일 뿐이고 lvalue이든 rvalue이든 메모리 상에서 그 값을 수정하는 것은 아무런 문제가 없다. )

[https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/](https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/),                     

[https://www.internalpointers.com/post/understanding-meaning-lvalues-and-rvalues-c](https://www.internalpointers.com/post/understanding-meaning-lvalues-and-rvalues-c)         