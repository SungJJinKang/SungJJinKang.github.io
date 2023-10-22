---
layout: post
title:  "C++ 파일시스템에서 파일 전체를 긁어올 때 실수하는 점 ( vector::reserve()의 중요성 )"
date:   2021-01-18
tags: [C++]
---

C++ filsesystem을 사용하다 보면 파일 전체를 한꺼번에 긁어와야하는 일이 생긴다.
이때 흔히하는 실수가 있다. 우선 그 사례를 보여주겠다.

```cpp
1.
std::ifstream inputFileStream{path};
if (inputFileStream.is_open())
{
    std::string str{std::istreambuf_iterator{ inputFileStream }, {}};
    inputFileStream.close();

    return str;
}


2.
std::ifstream inputFileStream{path};
if (inputFileStream.is_open())
{
     std::stringstream strStream;
     while(stream.peek() != EOF)
     {
        strStream << (char) stream.get();
     }
     return strStream.str();
}
```

위의 방법들은 모두 최악의 방법들이다.
간혹 stringstream을 이용하면 빠르다고 생각하는 사람들이 있지만 디버깅 결과 stringstream도 결국은 내부적으로 string 같이 작동한다.

이러한 방법들이 최악인 이유는 무엇인가?
바로 string 내부 버퍼들의 reallocation 때문이다.
두 방법 계속 string을 추가하는 방식으로 디버깅을 해보면 일정 길이 이상으로 늘어나면 내부 버퍼의 주소가 계속 바뀌는 것을 볼 수 있다.
std::string에는 small buffer optimization을 위한 버퍼 크기가 초과하게 되면 그 이후 부터는 dynamic size buffer에 저장이 되는 데 string을 계속 추가하다보면 이 buffer가 다 차게 되고 그럼 std::string은 또 다시 new char[~~~]와 같은 방법으로 string을 위한 새로운 버퍼를 allocation 하고 기존 버퍼의 data들을 이 새로운 버퍼로 복사한다.
이 reallocation의 비용이 적지않아 하여 성능저하가 발생하는 것이다.

위의 두 방법은 모두 string을 점진적으로 추가해나가기 떄문에 reallocation이 계속해서 발생하여 느렸던 것이다.

이를 해결하기 위해서는 std::string::reserve(size_t)를 이용하여 string을 넣기 전에 미리 내부 버퍼 사이즈를 size_t에 맞춰 allocation하는 것이다.
그럼 이 size_t까지는 string을 계속 추가해도 reallocation이 발생하지 않는다.

그럼 이 size_t를 file stream에서 알아내야한다.
밑의 코드에 그 방법이 나와있다.

```cpp
std::ifstream inputFileStream(path, std::ios::in | std::ios::binary | std::ios::ate); // std::ios::ate로 stream의 read position을 맨 마지막으로 설정해준다. ( 이 동작은 immediatly하게 작동한다 )

if (inputFileStream.is_open())
{
	std::string str{};
	str.reserve(inputFileStream.tellg()); // 현재 read pointer의 위치를 return한다 = file의 총 수
    inputFileStream.seekg(0, std::ios::beg); // read position을 다시 맨 처음으로 옮김
    str.assign(std::istreambuf_iterator{ inputFileStream }, {});
    inputFileStream.close();

    return str;
}
```
