---
layout: post
title:  "string에서 특성 character를 다른 character로 교체하기"
date:   2021-05-20
categories: C++
---

얼마 전 코딩 테스트를 보다가 재밌는 문제를 풀었다.       
string에서 하나의 character을 교체하는 문제였다.       
어찌보면 매우 쉬워보이는 문제인데 나는 당연히 이렇게 쉬운 문제를 그냥 내지는 않았을꺼 같고 빠른 방법을 찾으려 노력했다.     

매우 매우 긴 문자열이 있다고 하였을 때 특정 character를 다른 character로 교체할 때 당신은 어떤 방식을 사용할 것인가??          

총 4가지 방법을 알아 볼 것이다.          
그 전에 미리 4가지 방법의 벤치마크를 보여주겠다.       

```
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
NAIVE           20914 ns        20927 ns        37333
MY_WAY          38268 ns        38365 ns        17920
STRCHR           3444 ns         3449 ns       194783
SIMD             1626 ns         1611 ns       407273
```

-------------------------------------
-------------------------------------                

**첫번째 방법이다. 가장 Naive한 방법이 아닌가 싶다.**      
보통 character을 교체하는 코드를 짠다고 하면 이 방법을 사용할 것이다.        
아주 일반적인 방법이고 아마 대부분의 library에서는 이 방법으로 구현을 하였을 것이다.         

```c++
void NAIVE()
{
    for (auto i = 0; i < strSize; ++i) 
    {
        if (str[i] == cmpChar) {
            str[i] = replacedChar;
        }
    }
}
```
     
-------------------------------------
-------------------------------------  
        
**내가 쓴 방법이다.**
지금 생각하면 매우 매우 멍청한 방법이다.      
괜히 생각을 복잡하게 해서 매우 멍청한 방법으로 답을 냈다.......        
그때 내가 생각하기에는 character을 하나씩 비교하기 보다는 WORD 단위로(64bit에서는 8개의 character) 한꺼번에 비교를 하고자 했는 데 이 방법은 "FFFFFFFFZ" 이렇게 교체할 character 8개가 연속적으로 이어져있는 경우에만 더 빠른데 대부분의 string은 그렇지 않다. 쓸 때 없는 연산만 더 추가된 것이다.     

```c++
void MY_WAY()
{
    size_t mask = 0;
    size_t replaced = 0;
    for (size_t i = 0; i < sizeof(size_t); i++)
    {
        mask |= cmpChar << i;
        replaced |= replacedChar << i;
    }

    for (size_t i = 0; i < strSize; i++)
    {
        if ((size_t)(str + i) % sizeof(size_t) == 0 && // when char position is aligned to WORD size
            strSize - i > sizeof(size_t) &&
            *(size_t*)(str + i) == mask)
        {
            *(size_t*)(str + i) = replacedChar;
            i += sizeof(size_t);
            continue;
        }

        if (a2[i] == cmpChar)
        {
            a2[i] = replacedChar;
        }
    }
}
```
  
-------------------------------------
-------------------------------------  
      
**strchr함수를 사용하는 방법이다.**       

```c++
#include <cstring>

void STRCHR()
{
    for (char* p = a3; *p && (p = strchr(p, cmpChar)) != nullptr; ++p)
    {
        *p = replacedChar;
    }
}
```

strchr의 내부 구현은 이러하다.         

```c++
char * strchr (register const char *s, int c)
{
  do {
    if (*s == c)
      {
	return (char*)s;
      }
  } while (*s++);
  return (0);
}
```
이렇게 보면 NAIVE 방법이랑 STRCHR이랑 똑같은 코드를 실행하는 것 처럼 보인다.          
그렇지만 벤치마크를 보면 두 방법 사이의 성능차는 크다.       
이유는 **플랫폼에 따라 strchr을 하나의 assembly instruction으로 구현되어 있기 때문이다.**      
위의 strchr는 portable(플랫폼과 무관한)한 구현 방법이고 실제 많은 플랫폼에서는 하나의 instruction이기 때문에 매우 좋은 성능을 보여준다.      
실제로 strchr을 위의 portable한 함수로 대체한 경우 NAIVE로 구현한 것과 성능차이가 거의 없다.      

        

-------------------------------------
-------------------------------------           
       
**마지막으로 SIMD 명령어를 사용한 방법이다.**      
256bit 즉 32개의 character을 한꺼번에 비교 후 blend로 대입하는 방법이다.          
당연히 가장 빠른 방법이다.   

```c++
#include <immintrin.h>

void SIMD()
{
    __m256i cmpMask = _mm256_set1_epi8(cmpChar);
    __m256i replacedMask = _mm256_set1_epi8(replacedChar);

    for (char* p = a4; p < targetPointer; )
    {
        if ((size_t)p % 32 == 0 && str + strSize - p > 32)
        {// p is aligned to 256bit (SIMD!!)
            
            __m256i cmpResult = _mm256_cmpeq_epi8(*(__m256i*)p, cmpMask);
            *(__m256i*)p = _mm256_blendv_epi8(*(__m256i*)p, replacedMask, cmpResult);

            p += 32;
            continue;
        }
        else
        {
            if (*p == cmpChar) {
                *p = replacedChar;
            }
            p++;
            continue;
        }
    }
}
```

-------------------------------------
-------------------------------------        
         


위의 벤치마크는 모든 Optimization option을 끈 코드이다.        
**사실 컴파일러가 자동적으로 vectorize를 하여 NAIVE한 방법도 SIMD register를 사용해 SIMD 함수와 비슷한 성능을 낸다**       



reference : [https://stackoverflow.com/questions/67647108/how-to-replace-a-char-in-string-with-another-char-fasti-think-test-didnt-want?noredirect=1#comment119569489_67647108](https://stackoverflow.com/questions/67647108/how-to-replace-a-char-in-string-with-another-char-fasti-think-test-didnt-want?noredirect=1#comment119569489_67647108)
