---
layout: post
title:  "typename을 붙여야 하는 경우"
date:   2021-05-19
categories: C++
---

짧게 말하자면 : 다른 타입에 의존적 관계에 있는 중첩 이름에 대한 참조를 할 때 마다. 예) 알려지지 않은 템플릿 매개변수를 가진 템플릿 인스턴스에 속한 이름     

길게 말하자면 : C++에는 엔티티들의 3가지 등급(종류이 존재한다. 값, 타입, 템플릿이 그것이다. 이 세개 모두 이름을 가지고 있고 그 이름 자체는 너에게 그것이 어떤 티어에 속한 것인지를 말해주지 않는다.      
그 이름을 가진 엔티티의 속성에 대한 정보(등급, 엔티티 종류)는 문맥에서 추론된다.        

이 추론이 불가능할 때 마다 티어를 명확히 해주어야한다.        

```c++
template <typename> struct Magic; // 어딘가 정의 되어 있다.

template <typename T> struct A
{
  static const int value = Magic<T>::gnarl; // 값이라 생각하라

  typedef typename Magic<T>::brugh my_type; // 타입이라 선언
  //      ^^^^^^^^

  void foo() {
    Magic<T>::template kwpq<T>(1, 'a', .5); // 타입이라 선언
    //        ^^^^^^^^
  }
};
```
여기 이름들 Magic<T>::gnarl, Magic<T>::brugh and Magic<T>::kwpq는 티어를 명시해야한다.             
Magic이 템플릿이기 때문에, Magic<T>는 T 타입에 의존한다. -- 예를 들어보면 템플릿의 주 정의와 완전히 다른 템플릿 특수화들이 존재할 수 있다.               

Magic<T>::gnarl를 의존적 이름으로 만드는 것은 우리가 알려지지 않은 type T안에 있기 때문이다.             
  우리가 Magic<int>를 사용했다면 컴파일러는 Magic<int>의 정의를 모두 알기 떄문에 의존적이지 않은 이름이 된다.     

```c++
template <typename T> struct Magic
{
  static const T                    gnarl;
  typedef T &                       brugh;
  template <typename S> static void kwpq(int, char, double) { T x; }
};
template <> struct Magic<signed char>
{
  // Magic<signed char>에서는 gnarl가 존재하지 않는다!!!
  static constexpr long double brugh = 0.25;  // `brugh`는 이제 값이다!!
  template <typename S> static int kwpq(int a, int b) { return a + b; }
};
```
Usage:
```c++
int main()
{
  A<int> a;
  a.foo();

  return Magic<signed char>::kwpq<float>(2, 3);  // 모호함이 없어졌다!!
}
```
reference : [https://stackoverflow.com/questions/7923369/when-is-the-typename-keyword-necessary](https://stackoverflow.com/questions/7923369/when-is-the-typename-keyword-necessary)
  
  
