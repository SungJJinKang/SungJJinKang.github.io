---
layout: post
title:  "memcpy vs memmove"
date:   2021-03-23
categories: C++
---

간단히 설명하면 memcpy는 데이터를 그대로 WORD단위로 복사한다.     
memmove는 똑같은 곳을 참조해서 복사하지 않는 지 체크하고 방지한다.    
말이 어렵다 memmove의 구현 코드를 보자.    

```c++

#undef	wsize
#define	wsize	sizeof(word) // 워드의 사이즈는 항상 2의 pow이다
#undef	wmask
#define	wmask	(wsize - 1) // 글머 wmask는 2의 pow - 1 -> 모든 비트가 1로 채워짐

wmask is all 1

void *
memmove(dst0, src0, length)
#else
void
bcopy(src0, dst0, length)
#endif
#endif
	void *dst0;
	const void *src0;
	register size_t length;
{
	register char *dst = dst0;
	register const char *src = src0;
	register size_t t;

	if (length == 0 || dst == src)		/* nothing to do */
		goto done;

	/*
	 * Macros: loop-t-times; and loop-t-times, t>0
	 */
#undef	TLOOP
#define	TLOOP(s) if (t) TLOOP1(s)
#undef	TLOOP1
#define	TLOOP1(s) do { s; } while (--t)

	if ((unsigned long)dst < (unsigned long)src) {
		/*
		 * Copy forward.
		 */
		t = (int)src;	/* only need low bits */
		if ((t | (int)dst) & wmask) { //destination과 source의 하위 4바이트들 중 1인 비트가 있는 경우
			/*
			 * Try to align operands.  This cannot be done
			 * unless the low bits match.
			 */
			if ((t ^ (int)dst) & wmask || length < wsize) // destination과 source의 하위 4바이트 중 서로 다른 비트가 존재하거나(데이터 복사해야함) 복사할 byte수가 word 사이즈보다 작은 경우
				t = length; // 복사할 lenth 모두를 1바이트씩 복사
			else // 
				t = wsize - (t & wmask); // 복사할 byte수 중 초기 n바이트는 1바이트씩 일일이 복사함
			length -= t;
			TLOOP1(*dst++ = *src++); //t만큼은 WORD단위가 아닌 1바이트씩 복사
		}
		/*
		 * Copy whole words, then mop up any trailing bytes.
		 */
		t = length / wsize;
		TLOOP(*(word *)dst = *(word *)src; src += wsize; dst += wsize); // 워드 단위로 복사
		t = length & wmask;
		TLOOP(*dst++ = *src++); // 마지막에 남는 워드 사이즈 이하의 바이트는 한 바이트씩 복사
	} else {
		/*
		 * Copy backwards.  Otherwise essentially the same.
		 * Alignment works as before, except that it takes
		 * (t&wmask) bytes to align, not wsize-(t&wmask).
		 */
		src += length;
		dst += length;
		t = (int)src;
		if ((t | (int)dst) & wmask) {
			if ((t ^ (int)dst) & wmask || length <= wsize)
				t = length;
			else
				t &= wmask;
			length -= t;
			TLOOP1(*--dst = *--src);
		}
		t = length / wsize;
		TLOOP(src -= wsize; dst -= wsize; *(word *)dst = *(word *)src);
		t = length & wmask;
		TLOOP(*--dst = *--src);
	}
done:
#if defined(MEMCOPY) || defined(MEMMOVE)
	return (dst0);
#else
	return;
#endif
}
```
본론에 들어가기 전에 컴퓨터는 데이터를 1비트씩 옮기지 않고 WORD단위로 옮긴다. 이 WORD는 운영체제에 따라 다르지만 보통 4 or 8바이트이다. 그리고 WORD단위로 데이터를 옮기기 위해서는 데이터 복사의 시작 주소가 WORD사이즈의 배수여야 한다. x32체제에서는 데이터 복사의 시작 주소가 4의 배수여야한다. 

정말 간단히 설명해 보겠다.    
우선 destination 주소(d)와 source 주소(s)를 본다.     
source 주소가 destination 주소보다 큰 경우는 낮은 주소에서 큰 주소로 복사를 해나간다.  
반대의 경우는 큰주소에서 낮은 주소로 복사해 간다.       

이렇게 하는 이유는 아래의 경우를 보면 된다. 

```
        s       d
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
        ----------------> 
                --------------------> ( s에서 4바이트를 d로부터 4바이트 만큼에 복사)
```
만약 source 주소가 destination 주소 보다 낮은 데 낮은 주소에서 높은 아래의 경우를 보면 초기 4바이트인 source의 4, 5, 6, 7위치의 데이터를 8, 9, 10, 11로 옮기게 되는 데 이 경우 원래 source가 의도한 바와 달라지게 된다.     
**원래 의도는 4 ~ 11의 데이터를 8 ~ 15로 온전히 옮기려고 했는 데 첫번째 복사에서 4, 5, 6, 7의 데이터가 8, 9, 10, 11로 복사되면서 8, 9, 10, 11의 데이터가 4, 5, 6, 7의 데이터로 덮어 씌어졌다.**      
만약 이걸 프로그래머가 의도하지 않았을 경우 memmove를 사용하면 된다.              
이를 방지하기 위해 이 경우에는 memmove 호출 시 뒤에서 부터 데이터를 복사할 것이다.       

근데 여기서도 중요한게 성능 향상을 위해 WORD단위로 데이터를 옮기기 위해서는 align에 맞지 않는 초기 몇 바이트는 1바이트씩 우선 옮기고 그 후 4바이트 단위로 옮긴다는 것이다.  
```
    d       s
0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
    --------------->      
            ------------------> 
```
이 경우는 대부분의 경우 4의 배수로 시작하지 않는 앞의 2, 3 주소의 데이터는 1바이트씩 우선 복사 한후 WORD단위(4바이트)씩 복사 한 후 마지막 2바이트를 다시 1바이트씩 복사한다.           

source의 위치가 destination보다 낮은 경우도 그냥 반대로 생각해보면 이해하기 쉽다.         

-------


여담으로 컴파일러 옵션에 따라 컴파일러가 알아서 SIMD 코드 사용하여 최적화를 해준다.      
아래의 코드를 보면 moveaps 명령어와 xmm0 레지스터가 쓰이는 데 이 것들이 SIMD 함수에서 사용되는 명령어, 레지스터다.         

```c++
float* a = new float[4]{1.0f, 2.0f, 3.0f, 4.0f};
float b[4];
memcpy(b, a, sizeof(int) * 3);
```

```
    call    operator new[](unsigned long)
movaps  xmm0, XMMWORD PTR .LC0[rip] // 16바이트 짜리 상수데이터를 XMMWORD 즉 128비트(16바이트)만큼 한꺼번에 SIMD 레지스터인 XMM으로 옮긴다.
mov     edi, OFFSET FLAT:_ZSt4cout
movups  XMMWORD PTR [rax], xmm0
```

컴파일러가 너무 똑똑하다 보니 필자가 보여주고 싶은 어셈블리 코드가 제대로 안나온다.