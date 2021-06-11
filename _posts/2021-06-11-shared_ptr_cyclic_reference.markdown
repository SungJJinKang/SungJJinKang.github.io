---
layout: post
title:  "shared_ptr 순환 참조와 해결방법"
date:   2021-06-11
categories: C++
---

간단히 말하면 **weak_ptr**을 사용하면 된다.             

```c++
#include <memory>
#include <iostream>

struct B;
struct A {
  std::shared_ptr<B> b;  
  ~A() { std::cout << "~A()\n"; }
};

struct B {
  std::shared_ptr<A> a;
  ~B() { std::cout << "~B()\n"; }  
};

void useAnB() {
  auto a = std::make_shared<A>();
  auto b = std::make_shared<B>();
  a->b = b;
  b->a = a;
}

int main() {
   useAnB();
   std::cout << "Finished using A and B\n";
}

output : 
Finished using A and B
```

위의 코드를 보면 struct A가 struct B에 대한 소유권을 가지고 잇고 struct B가 struct A에 대한 소유권을 가지고 있다.        
이 경우 서로가 서로의 소유권을 가지고 있기 때문에 어느 하나의 shared_ptr이 파괴되어도 다른 하나의 shared_ptr이 소유권을 가지고 있기 때문에 **내부적으로 두 오브젝트는 각각 1의 reference count를 여전히 가지고 있는 것이다.**            
Memory leak이 발생한 것이다. 우리는 두 A, B 오브젝트에 접근은 할 수 없지만 두 오브젝트는 여전히 살아있다.               

그래서 소멸자가 호출이 되지 않는다.          

이를 해결하기 위해 weak_ptr을 사용한 코드를 보자.            

```c++
#include <memory>
#include <iostream>

struct B;
struct A {
  std::shared_ptr<B> b;  
  ~A() { std::cout << "~A()\n"; }
};

struct B {
  std::weak_ptr<A> a;
  ~B() { std::cout << "~B()\n"; }  
};

void useAnB() {
  auto a = std::make_shared<A>();
  auto b = std::make_shared<B>();
  a->b = b;
  b->a = a;
}

int main() {
   useAnB();
   std::cout << "Finished using A and B\n";
}

output : 
~A()
~B()
Finished using A and B
```

비로소 소멸자가 호출된다.       
weaked_ptr은 내부적으로 shared_ptr의 shared_counter와 별개로 weak_couter를 가지고 있다.      shared_ptr이 파괴될 때 우선 shared_ptr의 shared_counter가 0인지를 체크하고 0이면 그 다음으로 weak_counter가 0인지 확인한다. 0이면 shared_ptr이 보유중인 오브젝트를 파괴한다. weak_counter가 0이 아니면 파괴되지 않는다.           
**즉 shared_ptr이 모두 파괴되더라도 그 shared_ptr을 참조한 weak_ptr이 하나라도 살아있으면 오브젝트는 파괴되지 않는다.**              


```c++
libcxx

//shared_ptr의 release 함수
bool
__shared_count::__release_shared() _NOEXCEPT
{
    if (decrement(__shared_owners_) == -1)
    {
        __on_zero_shared();
        return true;
    }
    return false;
}

//shared_ptr, weak_ptr 모두 __shared_weak_count를 내부 reference_counter로 사용한다.    
//shared_ptr의 release 함수
void
__shared_weak_count::__release_shared() _NOEXCEPT
{
    if (__shared_count::__release_shared())
        __release_weak();
}

void
__shared_weak_count::__add_weak() _NOEXCEPT
{
    increment(__shared_weak_owners_);
}

//weak_counter이 0인지 체크하는 함수
//0이면 shared_ptr이 보유중인 
void
__shared_weak_count::__release_weak() _NOEXCEPT
{
    if (decrement(__shared_weak_owners_) == -1)
        __on_zero_shared_weak();
}
```

위의 libcxx 코드를 보면 모든 shared_ptr와 weak_ptr은 __shared_weak_count을 내부 reference_counter로 사용한다.      
그리고 __shared_owners_와 __shared_weak_owners_라는 count를 하는 long 타입 변수 두개를 가지고 있다.          
weak_ptr이 shared_ptr를 받아 생성이 되면 weak_ptr은 내부적으로 자기 매개변수로 받은 shared_ptr의 __shared_weak_count.__add_weak 함수를 호출하여 shared_ptr의 __shared_weak_owners_을 1 올린다.         


reference : [https://github.com/google/libcxx/blob/master/src/memory.cpp](https://github.com/google/libcxx/blob/master/src/memory.cpp)  ,  [https://github.com/llvm-mirror/libcxx/blob/master/include/memory](https://github.com/llvm-mirror/libcxx/blob/master/include/memory)      