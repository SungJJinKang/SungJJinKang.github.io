---
layout: post
title:  "메타프로그래밍이란?"
date:   2021-06-11
categories: ComputerScience C++
---

메타 프로그래밍은 간단히 설명하면 **어떤 하나의 코드를 작성해서 다른 코드를 작성하는 것**이다.             
뭔가 애매하다. 쉽게 말하면 템플릿을 생각하면 된다.      
템플릿은 기본이 되는 템플릿 코드를 작성하고 난 후 그 템플릿 코드에 다른 템플릿 매개 변수를 넘기면 그 매개 변수로 하나의 새로운 코드가 내부적으로 생긴다.     
(템플릿이라는 것이 정말 별거 아니다. 그냥 템플릿 매개변수의 자리에 넘겨진 타입 혹은 변수를 대신 넣어서 코드를 똑같이 쓰는 것이다)       

C++에서는 이 템플릿 메타프로그래밍을 이용해 여러 Container가 작성되었다.        


```c++
template <typename T>
class vector
{
private:
    T* begin;
    T* end;
    T* capacityPtr;

public:
    vector(size_t n)
    {
        this->begin = new T[n]();
        this->end = this->begin + n;
        this->capacityPtr = this->end;
    }

    ~~~
};
```

위와 같이 T에 어떤 타입을 넣느냐에 따라 수 많은 다른 코드를 생성해낼 수 있다.        

만약 template이 없다면 어떻게 vector을 작성할 수 있을까????     
```c++
class vectorInt
{
private:
    int* begin;
    int* end;
    int* capacityPtr;

public:
    vector(size_t n)
    {
        this->begin = new int[n]();
        this->end = this->begin + n;
        this->capacityPtr = this->end;
    }

    ~~~
};

class vectorFloat
{
private:
    float* begin;
    float* end;
    float* capacityPtr;

public:
    vector(size_t n)
    {
        this->begin = new float[n]();
        this->end = this->begin + n;
        this->capacityPtr = this->end;
    }

    ~~~
};
```

위와 같이 모든 타입에 대해 vector을 일일이 만들어주어야 한다.          

