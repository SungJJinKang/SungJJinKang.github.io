---
layout: post
title:  "Cache Prefetcher"
date:   2021-05-14
tags: [ComputerScience]
---

[CPPCON 영상](https://youtu.be/BP6NxVxDQIs)을 보다 Cache Prefetcher에 대해 알게되어 글을 적어보겠다.     

( 이 글의 모든 벤치마크는 x64 msvc 컴파일러에서 연산된 결과이다. )        

프로그램의 성능을 하락시키는 주 원인은 무엇일까? 바로 메모리 Fetch이다. 그래서 CPU는 캐시와 같은 더 빠른 메모리를 도입하였는데 Cache는 사이즈가 작아 Cache에 데이터가 없을 경우 메인 메모리에서 데이터를 가져와야하는데 이때 CPU가 메모리로부터 데이터를 읽어오는 것을 기다리게 되는 것을 Memory Stall이라 하며 캐시에 접근하는 경우에 비해 상대적으로 매우 오래 걸린다.                     
그래서 CPU가 어떤 데이터를 필요로 할 때 그 데이터가 캐시에 있을 확률을 높이는 것이 프로그램의 성능과 직결된다.       
이러한 목표로 나온 것이 Cache Prefetch이다.       
Cache Prefetcher는 어떠한 데이터가 필요하기 전 미리 메인 메모리에서 캐시로 가져오는 것을 말한다.     
이 Cache Prefetch가 작동하는 방식에는 HW를 통한, 혹은 SW를 통한 방식이 있다.       
자세한 작동 방식은 [위키디피아](https://en.wikipedia.org/wiki/Cache_prefetching#Hardware_vs._software_cache_prefetching)를 참고하기 바란다.    
어쨌든 핵심은 data fetch의 흐름을 보고 다음에 fetch할 데이터를 미리 캐시에 올려두는 것이다.     

그럼 벤치마크를 통해 Cache Prefetch가 어떠한 성능 향상을 가져다 주는지 보자.     

```cpp
static void RowFirst() 
{
    int a[100][100];
    int result = 0;

    for(size_t i = 0 ; i < 100 ; i++)
    {
        for(size_t j = 0 ; j < 100 ; j++)
        {
            result += a[i][j];
        }
    }
}

static void ColumnFirst() 
{
    int a[100][100];
    int result = 0;

    for(size_t i = 0 ; i < 100 ; i++)
    {
        for(size_t j = 0 ; j < 100 ; j++)
        {
            result += a[j][i];
        }
    }
}

void Randomly() 
{
    int a[100][100];
    int result = 0;

    for(size_t i = 0 ; i < 100 ; i++)
    {
        for(size_t j = 0 ; j < 100 ; j++)
        {
            result += a[j][randIndex[i]]; // randIndex에는 미리 입력된 random number가 저장되어 있다.
        }
    }
}
```

위의 세 함수 RowFirst, ColumnFirst, Randomly는 각각 2D array에 접근한다.         
중요한건 RowFirst 방식은 한 행씩 우선적으로 접근하여서 Row major 기반의 CPP에서 한 캐시라인에 있는 데이터를 연속적으로 접근하여서 Cache miss를 최대한 줄였다.       
반면 ColumnFirst와 Randomly 함수는 Cache Line을 무시하고 매 iteration마다 서로 다른 캐시라인에 접근하였다.     
얼핏보면 ColumnFirst와 Randomly 두 함수의 성능상의 차이는 없을꺼 같아보인다.    

그럼 벤치마크를 보자. ( 벤치마크는 컴파일러 옵션의 Optimization option을 모두 끈 상태로 진행되었다 )

<img width="434" alt="20210514001711" src="https://user-images.githubusercontent.com/33873804/118149673-bb6e1500-b44c-11eb-8de5-b22486d77f77.png">     

Randomly 방식이 ColumnFirst 방식보다 훨씬 느리다.     
더 중요한 것은 ColumFirst 방식과 RowFirst 방식의 성능상 큰 차이가 없다는 점이다.           
이는 위에서 배운 Cache Prefetch 때문이다.       
ColumnFirst 함수에서 일정한 stride(간격)마다 데이터에 접근한다는 것을 CPU가 인지하고 다음 Column의 데이터를 미리 fetch하였기 때문에 Randomly 함수에 비해 훨씬 빠른 것이다.      
