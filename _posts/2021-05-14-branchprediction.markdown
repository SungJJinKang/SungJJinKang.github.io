---
layout: post
title:  "Branch Prediction에 대한 아주 구체적인 예시( virtual function call )"
date:   2021-05-14
tags: [ComputerScience, Recommend]
---

[CPPCON 영상](https://youtu.be/BP6NxVxDQIs)을 보다 Branch Prediction에 대해 알게되어 글을 적어보겠다.     

( 이 글의 모든 벤치마크는 x64 msvc 컴파일러에서 연산된 결과이다. )        

Branch Prediction은 간단히 설명하면 if 분기점에서 CPU가 하나의 결과를 예측하여 그 예측한 결과를 토대로 명령어를 실행하는 방식이다.   
이는 분기의 결과를 기다리는게 느리니 차라리 하나의 결과를 예측해서 명령어를 실행하다 예측이 틀렸을 경우 다시 롤백을 하는 것이 훨씬 빠르다 판단하여 생긴 기법이다.        
실제로는 CPU의 Branch Prediction은 대부분의 경우 예측한대로 결과가 나와 롤백하는 경우가 극히 드물다(10% 미만)     

자세한 내용은 [글 1](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array)과 [글 2](https://en.wikipedia.org/wiki/Branch_predictor)를 참고하기 바란다.      

그럼 Branch Prediction의 좋은 예를 보겠다. 

```cpp
static void Sorted()
{
    std::vector<int> a(1000000);
    std::generate(a.begin(), a.end(), [] {return (rand() % 2) ? 1 : -1; });

    std::sort(a.begin(), a.end()); // !!!!!

    int result = std::count_if(a.begin(), a.end(), [](int value) {return value > 0; });
}

static void NotSorted()
{
    std::vector<int> a(1000000);
    std::generate(a.begin(), a.end(), [] {return (rand() % 2) ? 1 : -1; });

    int result = std::count_if(a.begin(), a.end(), [](int value) {return value > 0; });
}
```

겉보기에 NotSorted 함수는 std::sort 연산을 수행하지 않기 때문에 Sorted 함수에 비해 훨씬 빠른 시간내에 std::count_if 함수의 실행을 끝낼 것 처럼 보인다.     

그럼 벤치마크 결과를 보자.     
```
-----------------------------------------------------
Benchmark           Time             CPU   Iterations
-----------------------------------------------------
Sorted        3396630 ns      3417969 ns          224
NotSorted     7787517 ns      7812500 ns           90
```

Sorted 함수 즉 vector를 정렬한 경우 그렇지 않은 NorSorted에 비해 훨씬 빠른 시간내에 연산을 완료한다.   
이는 바로 Branch Prediction 덕분이다. std::count_if는 결국 내부적으로는 if 테스트를 모든 a의 element에 대해 실행한다.    
정렬된 경우에는 1000000번의 if에서 높은 확률로 Branch Prediction의 예측이 실제 condition test와 일치한다.     
그래서 벤치마크 결과와 같이 a vector가 정렬된 경우 훨씬 빠른 것이다.       

( 이 부분에 대해서는 [이 글](https://en.wikipedia.org/wiki/Branch_predictor)을 꼭 읽어보기 바란다. Branch Predition은 CPU Pipeline과 연관되어 있다. )

Branch Prediction은 virtual 함수를 호출할 때도 사용한다.    
보통 virtual function 호출을 할 때 드는 cost가 virtual table에서 적합한 virtual function을 찾는 비용이 크다고 생각하는 데 그렇지 않다. virtual table에서 virtual function을 찾는 작업 후 virtual function을 실행하는데 CPU 파이프라인상 CPU는 어떤 virtual function이 호출될지를 미리 예측(Branch Prediction)하고 그 함수의 호출 명령어를 미리 CPU 파이프라인에 올려둔다. 그런데 여기서 Branch Prediction이 틀린 경우 CPU는 미리 예측하여 CPU 파이프라인에 올려둔 명령어를 flush한 후 실제 호출할 virtual function call을 기준으로 다시 파이프라인을 채워 넣어야 되고 이때 생각보다 큰 cost가 발생하는 것이다.                
( [참조 글](https://stackoverflow.com/questions/667634/what-is-the-performance-cost-of-having-a-virtual-method-in-a-c-class/667680) )        

그럼 벤치마크를 한번 보자.       


```cpp
class Animal
{
public:
    virtual int Bite() noexcept
    {
        return 1;
    }
};

class Dog : public Animal
{
public:
    virtual int Bite() noexcept override
    {
        return 2;
    }
};

class Cat : public Animal
{
public:
    virtual int Bite() noexcept override
    {
        return 3;
    }
};

class Tiger : public Animal
{
public:
    virtual int Bite() noexcept override
    {
        return 4;
    }
};

static void SortedVTable()
{
    std::vector<Animal*> animals;
    std::fill_n(std::back_inserter(animals), 10000000, new Dog);
    std::fill_n(std::back_inserter(animals), 10000000, new Cat);
    std::fill_n(std::back_inserter(animals), 10000000, new Tiger);

    int result = 0;
    for (auto animal : animals)
    {
        result += animal->Bite();
    }
}

static void ShuffledVTable()
{
    std::vector<Animal*> animals;
    std::fill_n(std::back_inserter(animals), 10000000, new Dog);
    std::fill_n(std::back_inserter(animals), 10000000, new Cat);
    std::fill_n(std::back_inserter(animals), 10000000, new Tiger);


    std::random_device rd;
    std::mt19937 g(rd());

    std::shuffle(animals.begin(), animals.end(), g);

    int result = 0;
    for (auto animal : animals)
    {
        result += animal->Bite();
    }
}
```

벤치마크 결과      
```
---------------------------------------------------------
Benchmark               Time             CPU   Iterations
---------------------------------------------------------
SortedVTable     53515845 ns     53977273 ns           11
ShuffledVTable  191010150 ns    191406250 ns            4
```

animals가 Shuffle되어 매 iteration마다 다른 Object에 접근하는 경우 결국 거의 매번 Branch Prediction에 실패하여 위와 같이 정렬된 경우 보다 훨씬 긴 실행 시간을 보여준다.       
