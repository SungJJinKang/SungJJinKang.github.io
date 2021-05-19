---
layout: post
title:  "using namespace 사소한 디테일"
date:   2021-05-19
categories: C++
---

```c++
namespace B {
    void f(int){}
    void f(double){}
}
 
void h() {
    using B::f;  // 꼭 이렇게 써야한다. "using B;" 이건 안된다
    f('h');     
}

int main()
{
    h();
}
```
-
```c++
namespace Q {
    namespace V {   // V is a member of Q, and is fully defined within Q
// namespace Q::V { // C++17부터는 이렇게 써도된다!!!!!!!!!!!!!!!!
        class C { void m(); }; // C is a member of V and is fully defined within V
                               // C::m is only declared
        void f(); // f is a member of V, but is only declared here
    }
 
    void V::f() // definition of V's member f outside of V
                // f's enclosing namespaces are still the global namespace, Q, and Q::V
    {
        extern void h(); // This declares ::Q::V::h
    }
 
    void V::C::m() // definition of V::C::m outside of the namespace (and the class body)
                   // enclosing namespaces are the global namespace, Q, and Q::V
    {}
}
```
