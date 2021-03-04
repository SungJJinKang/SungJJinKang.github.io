---
layout: post
title:  "class의 member variable는 선언된 순서대로 intialized 된다"
date:   2021-03-05
categories: C++
---

```c++
class A
{
public:
    A()
    {
        std::cout << "A constructor" << std::endl;
    }
};

class B
{
public:
    B()
    {
        std::cout << "B constructor" << std::endl;
    }
};

class C
{
public:
    C()
    {
        std::cout << "C constructor" << std::endl;
    }
};

class D
{
private:
    B b;
    A a;
    C c;
    
    public:
    D() : a{}, b{}, c{}
    {
        
    }
    
};

int main()
{
    D d{};

    return 0;
}

output : 
B constructor       
A constructor
C constructor
```

output에서 보이듯이 D의 constructor에서 어느 순서로 initialize를 하든 상관 없이 member variable가 선언된 순서인 B, A, C 순서로 initialize 되었다.