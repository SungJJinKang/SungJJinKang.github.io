---
layout: post
title:  "싱글톤 만들 때 주의해야 할 점"
date:   2021-07-03
categories: DesignPattern
---

기본 생성자, 소멸자는 외부로부터 숨기고(private 혹은 protected)             
복사 생성자, 이동 생성자, 복사 대입 연산자, 이동 대입 연산자들을 모두 삭제해야한다.         

```cpp
class SingleTonClass
{

private:

    inline static SingleTonClass* mSingleTonClassInstance = nullptr;

    SingleTon()  = default;
    ~SingleTon() = default;
    SingleTon(const SingleTon&) = delete;
    SingleTon(SingleTon&&) noexcept = delete;
    SingleTon& operator=(const SingleTon&) = delete;
    SingleTon& operator=(SingleTon&&) noexcept = delete;

public:

    static SingleTonClass* GetSingletonInstance()
    {
        if(SingleTonClass::mSingleTonClassInstance == nullptr)
        {
            SingleTonClass::mSingleTonClassInstance = new SingleTonClass();
        }
        return SingleTonClass::mSingleTonClassInstance;
    }

};
```