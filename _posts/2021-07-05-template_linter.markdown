---
layout: post
title:  "왜 template을 사용할 때 translation unit이 그 template의 구현부를 모두 알아야할까? ( 그 이유에 대한 구체적인 원리 설명 )"
date:   2021-07-05
categories: C++
---

템플릿을 통해 코드들을 만드는 것은 컴파일 단계에서 한다. 그럼 어떤 템플릿 argument를 받아서 **해당 템플릿 argument에 대한 코드를 생성하려면 컴파일 단계에서 해당 템플릿 argument의 구현부를 볼 수 있어야한다.**           
그러니깐 각각의 translation unit(cpp 파일)이 해당 템플릿의 Definition을 볼 수 있어야한다.         
**혹은 다른 translation unit에서 해당 template argument를 받는 템플릿에 대한 Definition을 생성해주어서 링커단계에서 연결을 할 수도 있다.**                    


```c++
a.hpp

template <typename T>
int Do();

--------------------------
a.cpp

template <typename T>
int Do()
{

}

--------------------------
main.cpp

#include "a.hpp"

int main()
{
  Do<int>(); //ERROR 발생
}
--------------------------
```

위의 코드에서도 main.cpp translation unit 파일을 컴파일 단계에서 Do<int>에 대한 코드를 생성하려 해도 Definition을 알 수 없어서 템플릿 코드를 생성할 수 없는 것이다.                                   
**그렇다고 a.cpp 파일이 Do<int>가 쓰일 것이라는 것을 예상하고 Do<int>에 대한 코드를 생성할 수도 없는 노릇이니 Do<int> 함수에 대한 코드는 어디서도 생성되지 못하게 되고 결국 링커 단계에서 main.cpp은 Do<int>의 Definition을 찾으려고 해도 없어서 못찾게 되어 컴파일 에러가 발생하게 되는 것이다.**              

그럼 이렇게 해주면 문제가 해결된다.           
```c++
a.hpp

template <typename T>
int Do()
{

}

--------------------------
main.cpp

#include "a.hpp"

int main()
{
  Do<int>(); //컴파일 성공
}
--------------------------
```

그냥 main.cpp 파일이 컴파일 단계에서 Do의 Definition을 볼 수 있게 a.hpp에서 Definition을 그냥 헤더에 넣어버리는 것이다.         

다른 방법으로는 main.cpp 파일은 Definition을 못봐도 해당 template argument에 대한 템플릿 코드를 다른 곳에서 만들어주어서 링커 단계에서 찾을 수 있게 해주는 것이다.          
이를 C++에서는 Template explicit instantiation 이라 한다.          
```c++
a.hpp

template <typename T>
int Do();

--------------------------
a.cpp

template <typename T>
int Do()
{

}

template void Do<int> (); // Template explicit instantiation
--------------------------
main.cpp

#include "a.hpp"

int main()
{
  Do<int>(); //컴파일 성공
}
--------------------------
```

헤더에 Definition을 써주지 않아 원래 같았으면 Do<int>에 대한 코드는 main.cpp가 만들지 못해서 definition이 없는 에러가 링커 단계에서 발생해야하는데 위와 같이 **a.cpp에서 Do 템플릿 함수가 템플릿 argument로 int를 받을 때의 코드를 명시적으로 만들라고 하여 a.cpp파일은 Do<int>에 대한 코드를 생성하고 main.cpp는 링커 단계에서 a.cpp가 만든 Do<int>의 Definition을 연결해줄 수 있다.**                     