---
layout: post
title:  "lvalue, rvalue의 정확한 개념적 이해"
date:   2021-08-23
categories: C++
---

**lvalue는 메모리에서 식별할 수 있는 공간을 차지하고 있는 오브젝트**이다.       
반면 **rvalue는 메모리에서 식별할 수 있는 공간을 차지하고 있지 않은 오브젝트**이다.     
rvalue는 쉽게 말하면 일반 상수나, 레지스터에 값이 있는 경우이다.       
rvalue를 "리터럴 상수"라고 부르기도 한다. 레지스터에 값이 존재하기 때문에 단시간내에 사라질 변수이다.     

```
int* y = &x;  
여기서 y는 lvalue이다
&x는 rvalue이다. &는 lvalue를 매개변수로 전달받아 rvalue를 반환하는 연산자이다.       

int* y = &666; // 에러! -> &는 lvalue를 매개변수로 전달받는다.     
```

```
int y;
666 = y; // 에러!
666 rvalue는 특정 메모리 공간을 가지고 있지 않기 때문에 할당을 할 수 없다.
```

상수 10은 레지스터나 메모리에 저장되어 있는 것이 아니라 그냥 어셈블리 명령어 코드 자체에 상수 숫자 데이터가 들어가 있는 것이다. 그러므로 rvalue 이다.         

--------------------------

```
int global = 100;

int& setGlobal()
{
    return global;    
}

setGlobal() = 400; // 성공
```

int& 즉 **reference는 특정한 메모리 공간을 가지는 lvalue를 가리키기 때문에 reference 또한 lvalue이다.** ( 레퍼런스도 결국 내부적으로는 포인터랑 같다 )                     

lvalue reference는 무엇일까?    
말 그대로 lvalue를 가리키는 변수이다.         

```
int& yref = 10; // 에러!
```
위의 코드도 당연히 에러를 발생시킨다.        
lvalue reference는 특정한 메모리 공간을 가지는 lvalue를 가리키는 변수인데 10은 rvalue이기 때문이다.     
**특정한 메모리 공간을 가지는 lvalue를 가리키는 변수가 단시간 내에 사라질 변수인 rvalue를 가리킨다는 것은 말이 안된다.**             

```
class A
{

};
A& a = A(); // 에러!
```

위의 코드 또한 오류를 발생시킨다.          
A()는 임시 객체를 생성하는 것인데 임시 객체는 rvalue이기 때문이다.      

--------------------

```
const int& ref = 10;  // 성공!
```

위의 코드는 에러를 발생시키지 않는다. 왜일까?     
그럼 어떻게 const T&, lvalue 레퍼런스에는 rvalue를 전달할 수 있는 것인가. 이 또한 rvalue를 이해하면 쉬운데 rvalue라는 것은 상수로 바뀌어서는 안되는 수이다. 그러므로 T&는 바뀔 가능성이 있으나 const T&는 const로 보호를 해주어서 그 값이 바뀔 가능성이 없으므로 const T&에는 rvalue를 전달할 수 있는 것이다. ( 물론 이 모든 것은 컴파일러단(C++)에서 정한 규칙일 뿐이지 lvalue이든 rvalue이든 메모리 상에서 그 값을 수정하는 것은 기술적으로 불가능한 일은 아니다. )         

```
class A
{

};
const A& a = A(); // 성공!
```
임시 객체 A()는 rvalue이므로 그 값이 변경되면 안된다.         
그래서 const A&로 전달 받아야한다.       


실제로 위의 코드를 실행시키면 내부적으로는 상수 10을 저장할 숨겨진 lvalue 변수를 **스택에** 생성하고 const int&는 그 숨겨진 lvalue 변수를 가리키는 것이다. 그러니깐 따지고보면 이 경우도 겉으로는 rvalue를 가리키는 것 처럼 보이지만 내부적으로는 lvalue를 가리킨다.       

```
/ the following...
const int& ref = 10;

// 위의 코드는 내부적으로는 아래와 같이 작동한다.
int __internal_unique_name = 10; // 상수 10을 숨겨진 lvalue에 저장
const int& ref = __internal_unique_name; // reference는 실제로는 lvalue를 가리킴
```

[https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/](https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c/),                     

[https://www.internalpointers.com/post/understanding-meaning-lvalues-and-rvalues-c](https://www.internalpointers.com/post/understanding-meaning-lvalues-and-rvalues-c)         