---
layout: post
title:  "Cache Coherency와 False sharing에 대한 구체적 예시"
date:   2021-05-14
tags: [ComputerScience]
---

[CPPCON 영상](https://youtu.be/BP6NxVxDQIs)을 보다 얻은 Cache Coherency, False sharing에 대한 구체적인 예시를 들어보겠다.      

( 이 글의 모든 벤치마크는 x64 msvc 컴파일러에서 연산된 결과이다. )        
 
- Cache Coherency -            

```cpp
static void atomicVersion()
{
    std::atomic<int> a;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread2([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread3([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread4([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });

    thread1.join();
    thread2.join();
    thread3.join();
    thread4.join();
}

static void lockingVersion()
{
    std::mutex mutex;
    std::atomic<int> a;
    std::thread thread1([&]
        {
            mutex.lock();
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
            mutex.unlock();
        });
    std::thread thread2([&]
        {
            mutex.lock();
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
            mutex.unlock();
        });
    std::thread thread3([&]
        {
            mutex.lock();
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
            mutex.unlock();
        });
    std::thread thread4([&]
        {
            mutex.lock();
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
            mutex.unlock();
        });

    thread1.join();
    thread2.join();
    thread3.join();
    thread4.join();
    

}
```
두 함수 atomicVersion와 lockingVersion의 성능을 비교해보면 어떤 것이 더 빠를까?      
atomicVersion은 멀티스레드로 각 스레드가 a++를 각자 concurrent하게 하고 있고 반면 lockingVersion는 loop에 들어가기 전 mutex lock을 하므로 결국 그냥 각 스레드가 차례대로 loop에 들어가는 것과 똑같다.        
lockingVersion은 각 thread에 작업을 할당했지만 결국 하나의 thread에서 4번의 loop를 하는 것과 차이가 없다.       

얼핏보면 atomicVersion이 multi thread하게 동작하니 훨씬 빠를 것 처럼 보인다.      
벤치 마크를 보자. ( 벤치마크는 컴파일러 옵션의 Optimization option을 모두 끈 상태로 진행되었다 )        

```
---------------------------------------------------------
Benchmark               Time             CPU   Iterations
---------------------------------------------------------
atomicVersion    11870546 ns       218750 ns         1000
lockingVersion    5229011 ns       250000 ns         1000
```

놀랍게도 lockingVersion이 훨씬 더 빠르다. 이 이유는 Cache coherency 때문이다.     
MESI 프로토콜 ( 뭔지 모르면 [이 글](https://sungjjinkang.github.io/computerscience/2021/04/06/cachecoherency.html)을 참고하라 )로 인해 atomicVersion는 직전에 a++을 한 thread와 현재 a++를 할 thread가 다른 코어(물리적 코어)에 속해 있으면 직전의 a++을 한 thread가 Cache를 flush하기를 기다려야한다.(MESI 상태에서는 E상태를 놓아줘야한다)      
그렇다면 atomicVersion는 4개의 thread가 동시에 a++를 수 없이 하는 데 이 경우 엄청난 횟수의 Cache flush가 발생한다. 대걔 L1캐쉬는 코어 전용으로 있으니 L2캐시에 현재 L1캐시의 데이터를 옮기는 작업을 수 없이 수행해야한다. 이렇게 Cache를 flush하는 동작은 느릴 수 밖에 없다.         

반면 lockingVersion은 Cache flush를 3번만 수행하면 되니 multi threading의 이점을 전혀 활용하지 못함에도 불구하고 atomicVersion 보다 훨씬 빠른 것이다.      


------------------------------------------------


- False Sharing -       

자 이번에는 False sharing에 대한 예를 들어보겠다.       

```cpp
static void ThreadCount1(benchmark::State& state)
{
    std::atomic<int> a;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });

    thread1.join();
}

static void ThreadCount2(benchmark::State& state)
{
    std::atomic<int> a;
    std::atomic<int> b;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread2([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                b++;
            }
        });

    thread1.join();
    thread2.join();
}

static void ThreadCount3(benchmark::State& state)
{
    std::atomic<int> a;
    std::atomic<int> b;
    std::atomic<int> c;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread2([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                b++;
            }
        });
    std::thread thread3([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                c++;
            }
        });

    thread1.join();
    thread2.join();
    thread3.join();
}

static void ThreadCount4(benchmark::State& state)
{
    std::atomic<int> a;
    std::atomic<int> b;
    std::atomic<int> c;
    std::atomic<int> d;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a++;
            }
        });
    std::thread thread2([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                b++;
            }
        });
    std::thread thread3([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                c++;
            }
        });
    std::thread thread4([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                d++;
            }
        });

    thread1.join();
    thread2.join();
    thread3.join();
    thread4.join();
}
```
위의 각각 ThreadCount1, 2, 3, 4는 각자 다른 atomic변수를 증가하는 일정한 개수의 thread를 가진다. TrheadCount3는 3개의 서로 다른 atomic변수를 증가시키는 3개의 thread를 가지고 있다.     
얼핏보면 모든 thread는 서로 다른 atomic변수를 증가시키는 위에서 본 Cache Coherency도 없을테니 4개의 ThreadCount1, 2, 3, 4의 실행 시간의 차이가 없을 것 같아보인다.    

그럼 벤치마크를 보자.    

```
-------------------------------------------------------
Benchmark             Time             CPU   Iterations
-------------------------------------------------------
ThreadCount1    1439046 ns        32812 ns        10000
ThreadCount2    5483737 ns       125000 ns         1000
ThreadCount3    8620710 ns       140625 ns         1000
ThreadCount4   11841399 ns       312500 ns         1000
```

ThreadCount에 따라 엄청난 실행 시간 차이를 보인다.    
왜 이런 결과가 나왔을까??    
바로 False sharing 때문이다.    
간단히 설명하면 같은 Cache line에 속해 있는 데이터를 write 했을 때는 서로 다른 영역의 데이터 임에도 불구하고 Cache Coherency(Cache flush)가 작동한다는 것이다.      
False sharing에 대해 자세히 알고 싶다면 [이 글](https://sungjjinkang.github.io/computerscience/2021/04/01/falsesharing.html)을 보기 바란다.      

그럼 이 코드들을 개선해보자.        

```cpp
struct alignas(64) aligned_atomic_int
{
    std::atomic<int> atomic_value;
};

static void ThreadCount2_Aligned(benchmark::State& state)
{
    aligned_atomic_int a;
    aligned_atomic_int b;
    std::thread thread1([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                a.atomic_value++;
            }
        });
    std::thread thread2([&]
        {
            for (size_t i = 0; i < 100000; i++)
            {
                b.atomic_value++;
            }
        });

    thread1.join();
    thread2.join();
}
```

aligned_atomic_int라는 새로운 struct를 만들었는 데 이 struct는 std::atomic<int>를 멤버변수로 가지고 alignas(64)를 통해 변수를 캐시라인(64 byte)에 align되게 만들었다. ( memory alignment에 대해 모른다면 [이 글](https://sungjjinkang.github.io/computerscience/2021/03/28/memoryalignment.html)을 읽어보기 바란다. )      
이렇게 하면 thread가 증감하는 변수 a, b, c, d는 서로 다른 캐시 라인에 속해있으므로 서로 다른 코어에 속한 thread가 증감을 하여도 Cache flush가 필요없다.      

벤치마크 결과를 보자.      
```
--------------------------------------------------------------------
Benchmark               Time             CPU              Iterations
--------------------------------------------------------------------
ThreadCount1_Aligned    1447722 ns        41851 ns         7467
ThreadCount2_Aligned    2150025 ns        43248 ns        11200
ThreadCount3_Aligned    2986332 ns       171875 ns         1000
ThreadCount4_Aligned    3772771 ns       187500 ns         1000
```

이번에도 ThreadCount에 따라 약간의 실행 시간 차이를 보이지만 이전과 비교해서는 엄청나게 개선이 되었고 각 ThreadCount간 실행 시간 차이가 그렇게 크지 않다.       
아마 Thread 개수가 많아짐에 따라 CPU Scheduling에 의해 각 Thread가 ThreadCount가 적을 때에 비해 CPU를 덜 자주 할당받아 실행 시간이 늘어난 것 같다.       
