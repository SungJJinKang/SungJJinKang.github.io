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
weak_ptr은 원래의 shared_ptr의 reference_counter에는 아무런 영향을 주지 않는다.          
즉 weak_ptr은 weak_ptr의 존재 여부와는 상관 없이 shared_ptr의 reference_counter가 0이 되면 가지고 있던 오브젝트는 무조건 파괴된다.            



reference : [https://github.com/google/libcxx/blob/master/src/memory.cpp](https://github.com/google/libcxx/blob/master/src/memory.cpp)  ,  [https://github.com/llvm-mirror/libcxx/blob/master/include/memory](https://github.com/llvm-mirror/libcxx/blob/master/include/memory)      