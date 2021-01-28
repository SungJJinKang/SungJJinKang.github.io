---
layout: post
title:  "private destructor을 가진 클래스랑 unique_ptr 사용하기"
date:   2021-01-29
categories: C++
---

private destructor을 가진 클래스를 unique_ptr에 사용하려 하면 에러가 뜬다.   
이유는 간단하다 unique_ptr은 unique_ptr의 오브젝트가 파괴될때 소유중인 pointer의 오브젝트도 함께 파괴를 하는데 unique_ptr이 소유중인 오브젝트의 destructor에 접근이 불가능해서 생기는 문제이다.   

해결 방법은 간단하다 unique_ptr의 템플릿 argument 중에 deleter가 있는 데 여기 클래스 내부에 정의한 deleter functor을 전달해 주면 된다.   
이 custom delter은 당연히 감싸고 있는 클래스의 private member function destructor에 접근이 가능하다.   
unique_ptr을 만든 사람들도 나와 같은 문제를 겪었나 보다...   

```c++
class A
{
    struct Deleter
    {
        void operator()(A* ptr) const
        {
            delete ptr;
        }
    };

    private:
    ~A()
    {
        ~~~
    }
}

std::unique_ptr<A>, A::Deleter> aUnique_ptr;
```