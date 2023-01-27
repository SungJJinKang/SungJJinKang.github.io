---
layout: post
title:  "enum 대신 enum class를 사용할 때 성능 저하가 있을까?"
date:   2021-02-18
tags: [C++]
---

과거에는 enum class가 없어 enum 변수에 ineteger 값을 넣을 수 있었는 데 이것은 명시적 타입 캐스팅이 필요 없어 어찌 보면 편리해 보이기도 하지만 사람은 항상 실수를 하는 존재이기 때문에 알 수 없는 버그들을 유발하곤 했다.   
그래서 나온 게 enum class인 데 enum class를 사용하면 enum class 변수에 integer을 넣으려면 명시적 타입 캐스팅을 요구해서 이러한 점이 프로그래머의 실수를 줄여준다.   

필자도 진행 중인 게임 프로젝트에서 수 많은 opengl 관련 call들의 변수를 enum class로 만들어서 그래픽 api call을 할 때 매번 어떤 변수를 넣을 지 일일이 찾을 필요 없이 매개변수의 enum class type을 따라 정해져 있는 parameter을 전달해 실수도 엄청 줄이고 개발 시간도 높일 수 있었다.    

필자는 궁금했다. 과연 enum class를 사용하는 것이 혹시 성능 저하를 유발하는 것이 아닌가이다.    
실험을 해보니 enum과 enumclass의 성능저하는 없었고 어셈블리 코드에서도 똑같은 코드를 보여 주었다.   
컴파일러가 알아서 enum, enum class 값에 맞는 상수 값을 전달해준다. 즉 명령어 코드 상에는 차이가 없다.   

```cpp

enum ENUM_TEST : unsigned int
{
    A = 2,
    B,
    C,
    D,
    E
};

enum class ENUM_CLASS_TEST : unsigned int
{
    A = 2,
    B,
    C,
    D,
    E
};

unsigned int function(unsigned int para)
{
    return para + 1;
}

int main()
{
    function(ENUM_TEST::A);
    function(static_cast<unsigned int>(ENUM_CLASS_TEST::A));
}

```


```
function(unsigned int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi // edi에서 함수 내부 스택으로 데이터 옮김
        mov     eax, DWORD PTR [rbp-4] // 연산을 위해 연산 레지스터 eax로 스택에 저장된 매개변수 값을 옮김
        add     eax, 1
        pop     rbp
        ret
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, 2 //  ENUM_TEST 부분 | edi에 function의 parameter로 전달할 데이터 복사
        call    function(unsigned int)
        mov     edi, 2 //  ENUM_CLASS_TEST 부분 | edi에 function의 parameter로 전달할 데이터 복사
        call    function(unsigned int)
        mov     eax, 0
        pop     rbp
        ret
```   


번외로 여러 컴파일러로 컴파일을 해보면서 재밌는 사실도 발견하였다.   
X64 GCC와 ARM CLANG 컴파일러가 매개 변수를 전달하는 과정에서 약간은 다른 방식으로 매개 변수를 전달하는 것을 알 수 있었다.   

```cpp
function(ENUM_TEST::A); // 2
function(static_cast<unsigned int>(ENUM_CLASS_TEST::A)); // 2
```   


X64 GCC
```
push    rbp
        mov     rbp, rsp
        mov     edi, 2  // !!!!
        call    function(unsigned int)
        mov     edi, 2  // !!!!
        call    function(unsigned int)
        mov     eax, 0
        pop     rbp
        ret
```   

ARM CLANG
```
function(unsigned int):
        sub     sp, sp, #4 // 매개변수가 저장된 위치로 스택 포인터 이동
        str     r0, [sp]
        add     sp, sp, #4 // 스택 포인터 원상태로 복구
        bx      lr
main:
        push    {r11, lr}
        mov     r11, sp
        sub     sp, sp, #8
        mov     r0, #2 // 레지스터 r0에 상수 2 저장
        str     r0, [sp, #4]    // !!!! 현재 스택 포인터 - 4 에 위치에 매개변수 전달.
        bl      function(unsigned int)
        ldr     r0, [sp, #4]    // ??
        bl      function(unsigned int)
        mov     r0, #0
        mov     sp, r11
        pop     {r11, lr}
        bx      lr
```

내가 함수 function에 전달한 ENUM_TEST::A과 ENUM_CLASS_TEST::A은 엄밀히 따지만 다른 enum 형에 속한다.     
그래서 GCC는 당연히 edi 레지스터에 매개변수 2를 매번 저장하였다.      

반면 CLANG은 r0 레지스터에 2(ENUM_TEST::A가 2이다)라는 상수값을 넣고 첫 함수에서 매개 변수로 스택에 값을 전달 해준 다음 그 다음 함수인 function(static_cast<unsigned int>(ENUM_CLASS_TEST::A)); 를 호출 할 때 ~~이전에 넣어둔 함수 스택 값을 그대로 사용하는 것이다.~~ 어차피 ENUM_TEST::A과 ENUM_CLASS_TEST::A은 상수 값으로는 같으므로 그냥 이전 함수를 위해 전달했던 값을 그대로 사용하는 것이다.       
필자가 생각하기에는 CLANG의 방식의 GCC보다 빠를 것이라고 생각하는데 희한하게 CLANG은 ldr     r0, [sp, #4] 코드로 스택의 값을 다시 r0에 전달해 주었다. 도대체 왜 이게 필요한 것인지는 모르겠다....    

어쨋든 한번 궁금해서 실험해 본 것이 두 컴파일러가 같은 코드를 다른 방식으로 해결한다는 재미있는 사실을 발견하였다.   
이게 그냥 컴파일러의 차이인지 아니면 X64, ARM의 아키텍쳐의 차이 때문인지는 솔직히 잘 모르겠다.   

매개변수로 두 함수에 다른 값을 전달해주니 ARM CLANG도 GCC와 같이 매번 r0 레지스터에 상수값을 넣어주는 것을 볼 수 있다.   
```cpp
function(ENUM_TEST::A); // 2
function(static_cast<unsigned int>(ENUM_CLASS_TEST::B)); // 3
```   

ARM CLANG
```
main:
        push    {r11, lr}
        mov     r11, sp
        mov     r0, #2  // !!!!
        bl      function(unsigned int)
        mov     r0, #3  // !!!!
        bl      function(unsigned int)
        mov     r0, #0
        pop     {r11, lr}
        bx      lr
```