---
layout: post
title:  "템플릿 특수화 되지 않은 템플릿 함수/타입 사용(인스턴스화)시 컴파일 에러 띄우기"
date:   2023-02-17
tags: [C++]
---          

아래 코드의 경우 특수화 되지 않은 함수의 static_assert의 condtion 코드("!std::is_same<T, T>::value") 평가가 템플릿 인스턴스화 이전에는 가능하지 않기 때문에(condition 코드의 평가가 템플릿 매개변수 T에 의존적이다) 특수화 되지 않은 버전의 함수를 호출(인스턴스화)하기 이전에는 컴파일 에러가 뜨지 않는다. ( 어떤 타입 T에 대한 "std::is_same"를 특수화하여 value 값이 "false"가 나오게 만들 수 있다. )        
```cpp
template <typename T>
T GetValue(const std::string& SectionName, const std::string& KeyName) const
{
    static_assert(!std::is_same<T, T>::value, "Unsupported Type for config value");
    return T{};
}

template<>
bool GetValue<bool>(const std::string& SectionName, const std::string& KeyName) const;

template<>
INT64 GetValue<INT64>(const std::string& SectionName, const std::string& KeyName) const;

template<>
FLOAT64 GetValue<FLOAT64>(const std::string& SectionName, const std::string& KeyName) const;

template<>
std::string GetValue<std::string>(const std::string& SectionName, const std::string& KeyName) const;
```
                
![20230218032525](https://user-images.githubusercontent.com/33873804/219750546-ffbdfbef-b642-4933-a55f-b65f8d3233e4.png)                  
![20230218035607](https://user-images.githubusercontent.com/33873804/219760212-66ffdd3c-3c0b-40d5-b060-f6470b0a5d2c.png)                  
                 
아래 사진에서도 보이듯이 어떤 타입 T(사진에서는 "float")에 대한 "std::is_same"를 특수화하여 "std::is_same<T, T>::value"에 대한 "value" 값이 "false"가 나오게 만들 수 있기 때문에 컴파일러가 템플릿 인스턴스화 이전에 "!std::is_same<T, T>::value"에 대한 평가를 할 수 없는 것이다.          
![20230218040809](https://user-images.githubusercontent.com/33873804/219764672-422e0759-5aa7-4ee8-921e-04a3a98170c9.png)        
            
            
          
반면 아래의 코드는 템플릿 인스턴스화 이전에 static_assert의 condition 코드("false")의 평가가 가능하기 때문에 특수화 되지 않은 함수를 호출(인스턴스화)하지 않는 경우에도 컴파일 에러가 뜬다.    
```cpp
template <typename T>
T GetValue(const std::string& SectionName, const std::string& KeyName) const
{
    static_assert(false, "Unsupported Type for config value");
    return T{};
}
```
            
