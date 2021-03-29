---
layout: post
title:  "Memory Alignment"
date:   2021-03-28
categories: ComputerScience
---

Resources :      
https://docs.microsoft.com/en-us/cpp/cpp/alignment-cpp-declarations?view=vs-2019      
https://stackoverflow.com/questions/381244/purpose-of-memory-alignment     
https://developer.ibm.com/technologies/systems/articles/pa-dalign/       

**컴퓨터가 메모리에서 데이터를 가져오는 데 한번에 워드 단위로(64비트 환경에서는 8바이트, 32비트 환경에서는 4바이트)만 가져올 수 있다.**  
또한 **컴퓨터는 메모리에서 데이터를 가져올 때 가져올 메모리 데이터의 시작 주소는 꼭 워드의 배수여야 한다.** 즉 워드가 4바이트이 환경에서 3번주소부터 4바이트를 가져오는 게 불가능하다는 것이다.  
그렇다면 만약 데이터가 1바이트만 필요하다 하면 우선 목표하는 데이터랑 뒤에 7바이트까지 같이 올 수 밖에 없다. 근데 뒤에 7바이트는 필요가 없는 데이터들이므로 이 7바이트를 짤라내야하는 데 이 짤라내는 과정에서 연산이 더 필요하다.     
그래서 애초에 데이터를 구성할 때 8바이트 단위로 구성되게 만들어서 뒤 몇바이트를 짤라내는 과정이 필요없게 만든다.  

예를 들어 워드가 4바이트인 환경에서 주소 2부터 4바이트를 가져온다하자.
```
1  2  3  4  5  6  7  8  9  10
X  O  O  O  O  X  X  X  X  X
```
그럼 우선 1부터 4바이트까지 가져와 왼쪽 쉬프트 연산을 한번 수행한다.   
그리고 4바이트 부터 8바이트 오른쪽 쉬프트 3 연산을 수행한다. 그리고 두 데이터를 하나의 레지스터로 합친다..   
이렇게 memory가 alignment되어 있지 않으면 추가적인 연산이 발생한다.     

----------------------

<img width="431" alt="memoryalignmnet" src="https://user-images.githubusercontent.com/33873804/112888017-cffd8480-910e-11eb-8145-4b05f3c83b7e.PNG">    

위의 사진은 워드가 2바이트인 환경에서 1바이트씩 메모리에 읽어오는 것과 2바이트씩 읽어오는 경우의 성능차이를 나타낸 것이다.    
당연히도 워드가 2바이트인 환경에서 1바이트씩 메모리에 접근하는 것은 추가적인 작업이 필요하기 때문에 더 느리다..    
그리고 2바이트씩 접근할 경우에도 데이터가 1바이트에 alignment되어 있는 경우.    

즉 데이터의 시작 주소가 1인 경우(1부터 시작) or 3인 경우(물론 2인 경우도 가능하다) 추가적인 작업이 필요하다.    
예를 들어 데이터가 3부터 2바이트인 경우를 생각해보자.    
그럼 우선 시작 주소 2부터 2바이트를 읽어와서 앞에 1바이트를 짜르고(쉬프트 연산), 뒤에 시작 주소 4부터 2바이트를 읽어와 뒤의 1바이트를 짤라서 그 둘을 합치는 작업을 해야한다. 매우 매우 비효율적이다. 1바이트씩 읽어오는 경우와 큰 차이가 없다.

-------------------

그래서 컴파일러는 성능향상을 위해 암묵적으로 중간에 pad를 넣어서 메모리 alignment를 만들어준다.
```c++
struct A
{
    char a; // 1바이트
    int b; // 4바이트
    float c; // 4바이트
};
```
위의 struct A의 총 사이즈는 얼마일까?? 9바이트인가?? 아니다!!!!   
정답은 12바이트다.     
왜 그럴까???

멤버 변수 b에 접근한다 생각해보자. oops... b가 4바이트에 alignment되어 있지 않다. 앞에 char형의 a 때문에 2번째 주소부터 b가 시작되어서 데이터b를 가져오려면 시작 주소 1부터 4바이트를 가져와 앞의 3바이트만 가져가고 뒤의 시작 주소 4부터 시작해서 4바이트를 가져와서 앞의 1바이트만 짤라서 둘을 합쳐야 한다. ( 시작주소가 1이라 가능 ) 매우 비효율적이다.     

그래서 컴파일러는 char형 a 뒤에 3바이트 padding을 넣어 주었다. 그럼 멤버 변수 b를 가져올 때 온전히 시작 주소 4부터 4바이트를 가져와서 한번만에 b의 데이터를 가져올 수 있게 된다.    

--------------------------

임의로 프로그래머가 alignment를 설정해줄 수도 있다.    
```c++
struct alignas(16) A
{
    char a; // 1바이트
    int b; // 4바이트
    float c; // 4바이트
};

A A_Array[2];
```
alignas라는 명령문으로 struct A의 align을 16byte에 설정해주었다.    
A_Array[0]의 주소를 보면 0x003efd90이다.     
A_Array[1]의 주소는 무엇일까?? 위에서 struct A의 사이즈가 12바이트니 0x003efd9b일까??    
그렇지 않다. alignas(16) 즉 A는 16바이트에 align하므로 A_Array[1]은 A_Array[0]의 시작 주소에 16바이트를 더한 0x003efda0 이다.     

재밌는 건 A_Array[0]의 주소 0x003efd90을 decimal로 바꾸면 4128144이고 이걸 16으로 나누면 나머지 없이 딱 떨어지는 것을 알 수 있다.       
위에서도 말했 듯이 alignment란 시작 주소가 해당 alignment의 배수랑 같다는 것이다. 그래서 시작 주소 0x003efd9b이 16의 배수인 것이다.     


----------------------------------------------------------------------

```c++
struct ReallySlowStruct
{
    char c : 6 (6byte size);
    __int64 d : 64;
    int b : 32;
    char a : 8;
};

struct SlowStruct
{
    char c;
    __int64 d;
    int b;
    char a;
};

struct FastStruct
{
   __int64 d;
   __int b;
   char a;
   char c;
   char unused[2];
};
```

The examples given in the book are highly dependent on the used compiler and computer architecture. If you test them in your own program you may get totally different results than the author. I will assume a 64-bit architecture, because the author does also, from what I've read in the description. Lets look at the examples one by one:           

ReallySlowStruct IF the used compiler supports non-byte aligned struct members, the start of "d" will be at the seventh bit of the first byte of the struct. Sounds very good for memory saving. The problem with this is, that C does not allow bit-adressing. So to save newValue to the "d" member, the compiler must do a whole lot of bit shifting operations: Save the first two bits of "newValue" in byte0, shifted 6 bits to the right. Then shift "newValue" two bits to the left and save it starting at byte 1. Byte 1 is a non-aligned memory location, that means the bulk memory transfer instructions won't work, the compiler must save every byte at a time.      

SlowStruct It gets better. The compiler can get rid of all the bit-fiddling. But writing "d" will still require writing every byte at a time, because it is not aligned to the native "int" size. The native size on a 64-bit system is 8. so every memory address not divisable by 8 can only be accessed one byte at a time. And worse, if I switch off packing, I will waste a lot of memory space: every member which is followed by an int will be padded with enough bytes to let the integer start at a memory location divisable by 8. In this case: char a and c will both take up 8 bytes.         

FastStruct this is aligned to the size of int on the target machine. "d" takes up 8 bytes as it should. Because the chars are all bundled at one place, the compiler does not pad them and does not waste space. chars are only 1 byte each, so we do not need to pad them. The complete structure adds up to an overall size of 16 bytes. Divisable by 8, so no padding needed.         

In most scenarios, you never have to be concerned with alignment because the default alignment is already optimal. In some cases however, you can achieve significant performance improvements, or memory savings, by specifying a custom alignment for your data stuctures.    

In terms of memory space, the compiler pads the structure in a way that naturally aligns each element of the structure.      

```c++
struct x_
{
   char a;     // 1 byte
   int b;      // 4 bytes
   short c;    // 2 bytes
   char d;     // 1 byte
} bar[3];
```
struct x_ is padded by the compiler and thus becomes:

// Shows the actual memory layout
```c++
struct x_
{
   char a;           // 1 byte
   char _pad0[3];    // padding to put 'b' on 4-byte boundary
   int b;            // 4 bytes
   short c;          // 2 bytes
   char d;           // 1 byte
   char _pad1[1];    // padding to make sizeof(x_) multiple of 4
} bar[3];
```
