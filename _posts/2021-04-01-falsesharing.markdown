---
layout: post
title:  "False Sharing"
date:   2021-04-01
tags: [ComputerScience, Recommend]
---

Fasle Sharing이란??

대다수의 CPU 아키텍쳐가 채용하는 캐시 시스템은 N-Way associate로 64byte로 나뉘는 n개의 address 세트가 한 캐시 라인을 사용한다. N개의 서로 다른 64byte의 데이터가 하나의 캐시 라인을 공유한다. 그럼 한 address가 캐시에 접근하려면 기존에 cache에 있던 데이터를 main memory(혹은 코어간 공유되는 캐시)로 옮기고(dirty한 경우에) 그 자리에 캐시를 쓴다.    
이러한 캐시 시스템은 속도면에서 매무 매우 효율적인 데 필요한 데이터를 main memory에 접근할 필요없이 CPU에 가까이 있는 캐시에 매우 매우 빠르게 접근할 수 있기 때문이다.   
그런데 이러한 캐시 시스템이 multithreading 환경에서 문제가 된다. multithreading 환경에서 다수의 코어(프로세서)가 한 데이터에 접근하려 할때 각각의 코어는 자신만의 cache를 가지고 있다. 이 말은 각 코어 cache를 main memory와 [동기화](https://sungjjinkang.github.io/cachecoherency) 시켜주지 않으면 같은 address에 접근함에도 각 코어가 서로 다른 값을 볼 수 있다는 의미이다. 그래서 

```cpp
// 예제가 이상해도 그냥 상황설명을 위한 예제이니 이해해주시기 바랍니다.

alignas(CacheLineSize) struct foo {
    int a[5]
    int b[5]
};

static struct foo f;

// 아래의 두 함수는 서로 다른 코어에서 동시에 실행된다.

int sum_a(void)
{
    int s = 0;
    for (int i = 0; i < 1000; ++i)
        s += f.a[i % 5];
    return s;
}

void inc_b(void)
{
    for (int i = 0; i < 1000; ++i)
        ++f.b[i % 5];
}
```

위의 경우를 보자. 각각의 함수 sum_a와 inc_b는 다른 코어에서 실행된다. 사실 f.a와 f.b는 서로 연관이 없다. 즉 sum_a에서 s+= f.a[i % 5] 부분과 inc_b에서 ++f.b[i % 5] 부분은 각 코어간에 동기화가 필요 없는 부분이다. 서로 다른 데이터를 봐도 상관이 없다는 뜻이다.    
그러니 코어간 cache의 동기화 없이(!) 빠르게 작동할 것 같이 보인다.....            

그렇지만 컴퓨터 아키텍쳐는 그렇지 않다.       
**f.a[5], f.b[5]는 각각 20바이트로 총 40바이트, 64바이트의 하나의 캐시 라인에 속한다.**    
이 때 코어는 **하드웨어단에서**(소프트웨어단에서 locking을 하는 것이 아닌) 캐시 동기화를 한다.     
**한 코어(A)가 한 캐시 라인에 접근할 때 그 캐시 라인에는 다른 코어(B)가 저장한 캐시라인이 있는 경우 다른 코어(B)가 저장한 해당 캐시 라인을 두 코어가 공유하는 캐시(혹은 Main memory)로 복사한 후 그걸 다시 현재 접근하려는 코어(A)의 캐시로 가져온 후 그 캐시 값을 참조해 사용한다.**   
L2캐시의 경우 각 코어가 같은 캐시를 사용하기 위해 어찌 보면 불가피하게 필요한 과정처럼 보인다. ( 물론 논리적으로는 f.a와 f.b는 서로 아무런 관련이 없어서 cach coherence가 필요없다. )   
심지어 캐시를 공유하기 않는 L1캐시에도 똑같은 상황이 발생한다.    

그럼 어떻게 이 문제를 해결할 수 있을까???      
어떻게 CPU 코어들이 캐시를 동기화 시키지 않고 각자의 캐시에서만 동작하게 할 수 있을까???  
f.a, f.b가 한 캐시 라인에 속하지 않게 중간에 padding을 넣어줘서 서로 다른 캐시 라인에 속하게 만들면 된다. ( alignment를 지정해주는 컴파일러의 기능을 사용해도 된다 )         

--------------------------      

언리얼 엔진에서도 "PLATFORM_CACHE_LINE_SIZE" 키워드로 검색해보면 "False Sharing"을 방지하기 위한 여러 코드들을 보실 수 있습니다.             
![20230430040338](https://user-images.githubusercontent.com/33873804/235319998-d4a53efc-1019-4105-bf07-5eba17393210.png)

