---
layout: post
title:  "Pointer Type cast와 Value Type cast의 차이"
date:   2021-03-22
categories: C++
---

```c++
class A
{
    ~~~~
}

class B
{
    ~~~~
}

A a{};

(B)a
*(B*)&a
```

위 (B)a와 *(B*)&a의 차이는 무엇인가.      
첫번째 (B)a는 a 오브젝트를 typecasting하여서 새로운 클래스B의 오브젝트를 만든 것이다.     
반면 *(B*)&a는A a라는 오브젝트는 그대로 있고 a를 B라는 타입으로 컴파일러가 보는 것이다.    
어떠한 새로운 오브젝트도 만들어지지 않는다.         

reference : https://stackoverflow.com/questions/13746136/what-does-a-c-cast-really-do   
