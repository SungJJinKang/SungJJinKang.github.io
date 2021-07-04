---
layout: post
title:  "const std::string& VS std::string_view ( 번역 )"
date:   2021-07-04
categories: C++
---

```c++
void print(const std::string& str);
void print(std::string_view str);
```

둘 중 어느것이 빠를까?          

전달하는 과정에서 복사되는 data의 사이즈 자체만 놓고 보면 const std::string&가 더 빨라 보일 수 있다.     
( const std::string&는 문자열 포인터가 복사 복사되고, std::string_view의 경우에는 생성자가 호출이 되고 문자열 포인터와 문자열의 길이가 복사된다.)             

그렇지만 몇몇 경우에서는 std::string_view가 빠르다.               

**첫째**, const std::string&은 데이터가 std::string 안에 있는 것을 요구한다. ( 이해가 되지 않을 것이다. 밑의 예를 보자 ) Literal string(char*)을 전달하는 경우 그 문자열 데이터 자체가 std::string 안에 있는 것을 요구한다. 타입 변환을 하지 않으면 문자열 데이터 복사도 발생하지 않고 ( 문자열이 Small String Optimization 보다 긴 경우 ) 그렇게 메모리 할당도 발생하지 않는다.          

```c++
void foo1( std::string_view bob ) { // std::string_view는 단순히 literal string의 포인터만 가지고 있다.
  std::cout << bob << "\n";
}
void foo2( const std::string& bob ) { // std::string는 literal string의 모든 문자열을 bob으로 복사해야한다. ( std::string이 데이터를 가지고 있어야 한다. ) 
  std::cout << bob << "\n";
}
int main(int argc, char const*const* argv) {
  foo1( "This is a string long enough to avoid the std::string SBO" ); 
  foo2( "This is a string long enough to avoid the std::string SBO" ); 
}
```

위의 경우를 보면 std::string_view의 경우에는 메모리 할당이 발생하지 않는다. 그러나 만약 std::string_view 대신에 const std::string&을 사용하였다면 문자열 복사를 위해 메모리 할당이 필요하다. ( literal string의 포인터가 전달되면서 const std::string&에서 생성자가 호출되면서 literal string의 문자열을 전부 복사해야한다. )                        


std::string_view가 const std::string& 보다 빠른 **두번째 이유**는 std::string_view는 복사 없이 substring을 가지고 놀게 해준다. 너가 2기가 바이트 짜리 json 문자열을 파싱한다고 생각해보아라. 만역 너가 그것을 std::string으로 파싱한다면 각각의 그러한 파싱 코드들은 복사를 하여서 로컬 노드로 가져와야한다.          

대신 std::string_view를 사용하면 각 파싱 노드들은 원본 데이터를 참조만 하면 된다. ( 포인터 사이즈만큼만 복사가 발생한다. ) 이것은 엄청난 양의 메모리 할당, 복사를 줄여준다. 이를 통해 얻어지는 성능 향상은 엄청날 것이다.           

사실 이건 극단적인 케이스다. 그렇지만 어찌되었든 substring은 가지고 놀 때 std::string_view는 많은 성능 향상을 가져다 준다.             

둘 중 어느 것을 선택할지를 결정할 때는 std::string_view를 사용했을 때 단점을 알면 된다. 단점이 그렇게 많지는 않지만 몇가지 있다.         

일단 null terminatior('\0')를 잃는다. 만약 같은 문자열이 null terminator을 요구하는 3개의 함수들에 모두 전달하려 한다면 그냥 std::string으로 한번 변환하는 것이 현명할 것이다. 그러므로 만약 너의 코드가 null turminator가 필요하다고 알려지고 문자열들이 C 스타일로 코딩된 버퍼에서 오는 경우를 제외한다면, const std::string&으로 받아라. 위의 경우가 아니면 std::string_view로 받아도 된다. ( 쉽게 설명하면 null terminator가 필요하고 전달되는 타입이 C 스타일 문자열이 아니라면 ( C 스타일 const char*이 전달되는 경우에는 std::string을 생성하면서 문자열을 복사해야 하기 때문에 ) const std::string&으로 받아야한다는 것이다. 반면 null terminator가 필요 없다면 그냥 std::string_view로 받아도 된다. )            

만약 std::string_view가 문자열이 null terminator가 있는지 없는지를 알 수 있는 flag를 가지고 있다면 const std::string&을 쓰는 마지막 이유 조차 없어진다.           

std::string_view 보다 const &를 붙이지 않는 std::string를 쓰는 것이 더 나은 경우도 있다. 만약 너가 호출 후 문자열의 복사본을 영원히 가지고 있기를 원한다면 Call by value가 효율적이다. 이 경우를 보면 만약 문자열이 SSO ( Small size optimization )의 경우라면, 힙 영역 메모리 할당은 발생하지 않고 그냥 문자열을 복사할 것이다. 그래서 std::string&&와 std::string_view 두 가지 오버로드 버전의 함수를 가지면 약간 더 빠를 수도 있다. 그렇지만 그러한 코드는 code bloat를 이끌 수 있다. 이 code bloat 때문에 두 오버로드 버전의 함수를 가져서 오는 약간의 성능 향상이 다시 반감될 수도 있다.             

references : [https://stackoverflow.com/a/40129198](https://stackoverflow.com/a/40129198)     