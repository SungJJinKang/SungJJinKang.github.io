---
layout: post
title:  "Memory Alignment"
date:   2021-03-28
categories: ComputerScience
---

컴퓨터가 메모리에서 데이터를 가져오는 데 한번에 8바이트씩(64비트 환경에서는 8바이트, 32비트 환경에서는 4바이트)만 가져올 수 있다.    

그렇다면 만약 데이터가 1바이트만 필요하다 하면 우선 목표하는 데이터랑 뒤에 7바이트까지 같이 올 수 밖에 없다. 근데 뒤에 7바이트는 필요가 없는 데이터들이므로 이 7바이트를 짤라내야하는 데 이 짤라내는 과정에서 연산이 더 필요하다.     
그래서 애초에 데이터를 구성할 때 8바이트 단위로 구성되게 만들어서 뒤 몇바이트를 짤라내는 과정이 필요없게 만든다.        

( 데이터의 시작 주소가 8의 배수여야 한다고 하는 데 이건 잘못된 정보인 듯 )        

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
Source: https://docs.microsoft.com/en-us/cpp/cpp/alignment-cpp-declarations?view=vs-2019