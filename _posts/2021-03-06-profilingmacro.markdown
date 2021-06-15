---
layout: post
title:  "프로파일링 매크로 최적화하기(literal string overlap)"
date:   2021-03-05
categories: C++
---

현재 진행 중인 프로젝트에는 프로파일링을 도와주는 코드가 있다. 
```c++
#define D_START_PROFILING(name, layer) doom::profiler::StartProfiling(name, layer)
#define D_END_PROFILING(name) doom::profiler::EndProfiling(name)

D_START_PROFILING("Update", eProfileLayers::CPU);
this->Update();
D_END_PROFILING("Update");

output : 
Update  -  6581 microseconds
```
이 방식은 레인보우 식스를 개발한 개발자가 소개한 방법을 유튜브에서 보고 따라한 것인다.      
사실 프로파일링이라는 것이 리플렉션을 이용해서 구현을 해야하나 생각하였는 데 이렇게 어쩌면 원시적인 방법으로 큰 오버헤드 없이 프로파일링 기능을 구현할 수 있다.       

두 함수의 내부 코드를 살펴보면 이렇다. ( data race를 막기 위한 구현 부분은 뺐다. )     
```c++
static inline std::unordered_map<std::string, std::chrono::time_point<std::chrono::high_resolution_clock>> _ProfilingData{};

void Profiler::_StartProfiling(const char* name, eProfileLayers layer) noexcept
{
    Profiler::_ProfilingData[name] = std::chrono::high_resolution_clock::now();
}

void Profiler::_EndProfiling(const char* name) noexcept
{
    std::thread::id currentThread = std::this_thread::get_id();
    auto& currentThreadData = Profiler::_ProfilingData;
    long long consumedTimeInMS = std::chrono::duration_cast<std::chrono::microseconds>(std::chrono::high_resolution_clock::now() - currentThreadData[name]).count();
	
    // used time in microseconds
    if (consumedTimeInMS > 1000000)
    {
        std::cout << '\n' << "Profiler ( " << currentThread << " ) : " << name << "  -  " << 0.000001 * consumedTimeInMS << " seconds  =  " << consumedTimeInMS << " microseconds" << std::endl;
    }
    else
    {
        std::cout << '\n' << "Profiler ( " << currentThread << " ) : " << name << "  -  " << consumedTimeInMS << " microseconds" << std::endl;
    }
}
```

이 코드들을 보면 큰 성능저하를 일으키는 요소가 있다.     
literal string으로 프로파일링 내용을 넘기는 데 사람이 여러 프로파일링들을 구분하기 위한 프로파일링 제목과 시간을 저장하기 위해  
```c++
std::unordered_map<std::string, std::chrono::time_point<std::chrono::high_resolution_clock>>
```
에 데이터를 저장한다.    

근데 여기서 잘보면 unordered_map의 키 값을 std::string으로 하였다. 이게 왜 문제가 될까??    
생각을 해보자. D_START_PROFILING("Update", eProfileLayers::CPU);과 D_END_PROFILING("Update"); 를 호출하면 두 string literal "Update"는 unordered_map에서 일치하는 value를 찾기 위해 key값으로 사용된다. 그런데 여기서 unorderd_map의 키 값은 std::string이라서 내부적으로 자동으로 std::string 오브젝트가 생성되어 이 키값을 기존의 unordered_map의 데이터와 비교를 하는 것이다.      
그렇게 되면 매 D_START_PROFILING, D_END_PROFILING마다 std::string 임시 객체를 생성하고 그 안에 array에 literal string을 복사하는 과정을 매번 해주어야 하는 것이다. 적지않은 오버헤드이다.     

그렇다고 std::string_view 또한 임시 객체를 만들지 않는 것은 아니다.     

또한 std::string을 키 값으로 사용할 시 매번 std::string의 hash value를 구하기 위하 O(n)의 시간복잡도를 들여 hash value를 구한다. 이 또한 불필요한 연산이다. ( 자세한건 [이 글](https://sungjjinkang.github.io/c++/2021/05/08/unordered_map_vs_map.html)을 참고하자. )         

그래서 해결책으로 그냥 literal string의 address(char pointer)를 key값으로 사용하는 것이다. ( 참고로 std::hash는 c string 즉 char pointer에 대한 hash specialization을 제공하지 않아 키 값을 char pointer로 사용하는 경우 그냥 4byte 포인터 주소를 비교하는 작업을 한다. )      
생각해보니 key값으로 항상 literal string을 사용하기 때문에 굳이 문자열을 비교할 필요가 없다.              
```c++
std::unordered_map<const char*, std::chrono::time_point<std::chrono::high_resolution_clock>> _ProfilingData{};
```     
컴파일러는 매우 똑똑하여 알아서 똑같은 literal string은 중복해서 메모리를 allocate하지 않고 기존의 것과 같은 address를 사용한다.     
단 여기도 문제점이 있는 데 이것이 강제적인 사항이 아니라는 것이다.      
C++ 표준에서는 똑같은 literal string들을 한번만 allocate하여 두개 다 한 주소를 가지게 하는 것을 강제(!!!)하지 않기 때문에 컴파일러에 따라 const char*을 이용한 이 방법이 unordered_map의 키값으로 사용할 때 다른 value를 return할 수도 있다는 것이다.      

그래서 필자는 간단히 똑같은 literal string이 같은 address에 저장된 경우 key 타입으로 char pointer을 사용하는 코드를 템플릿 메타프로그래밍을 이용하여 작성하였다.    
```c++
using key_type = typename std::conditional_t<"TEST LITEAL STRING OVERLAP" == "TEST LITEAL STRING OVERLAP", const char*, std::string>;
```
위의 코드는 컴파일 타임에 결정되는 데 두 literal string이 같은 address인 경우 const char*을 key 타입으로 사용한다. ( 사실 똑같은 string literal이 다른 translation unit에 속하는 경우 등 다양한 경우를 테스트 해봐야하는 데 그냥 현시점에서는 msbuild가 literal string overlap을 지원하니 이렇게만 하고 넘어가겠다. )
