---
layout: post
title:  "팩토리 메소드 패턴"
date:   2021-07-02
tags: [DesignPattern, Recommend]
---

"추상 팩토리 패턴"에 대해 알아보기 전에 "팩토리는 무엇인가"??      
코딩 측면에서 쉽게 말하면 오브젝트를 생성하는 것을 "팩토리(공장)"이라 한다.       

그럼 추상 팩토리 패턴은 무엇인가??       

정말 쉽게 설명해보겠다.      
"삼성휴대폰"을 만들기 위해서는 "삼성에서 디자인한 SM011 메모리칩"과 "삼성에서 디자인한 S512 배터리"가 필요하다.  "삼성에서 디자인한 SM011 메모리칩"을 제작하자, 배터리 공장에는 "삼성에서 디자인한 S512 배터리"을 제작하자.        
그럼 "애플 휴대폰"을 만들기 위해서는 무엇이 필요한가? "애플에서 디자인한 AP13 메모리칩"과 "애플에서 디자인한 AB01 배터리"가 필요하다. 메모리칩 공장에 "애플에서 디자인한 AP13 메모리칩"을 제작하자, 배터리 공장에는 "애플에서 디자인한 AB01 배터리"을 제작하자.             
그리고 "모토로라 휴대폰"을 .......        
그리고 "노키아 휴대폰"을 .......          
그리고 ........                  
.......                   
               
자 여기서 애플 휴대폰을 만들려한다. 필요한 "애플에서 디자인한 AP13 메모리칩"과 "애플에서 디자인한 AB01 배터리"주문을 제작해야한다.               

이렇게 된다면 각각의 휴대폰을 만들기 위해서는 내가 모든 휴대폰에 들어가는 모든 부품 번호의 제품번호를 다 알아야한다....       
얼마나 힘든일인가? 그래서 이렇게 하는 것이다.              
공장(팩토리)에다가 미리 모든 휴대폰에 맞는 제품 번호를 미리 알려준다.        
그 후 삼성 휴대폰 만들꺼야라고만 메모리칩 공장, 배터리 공장에 전달을 해주면 공장에서 알아서 "삼성에서 디자인한 S512 배터리"를 알아서 생산해서 보내준다.         

이것이 팩토리 패턴이다.        
이것을 클래스 설계 측면에서 유용성을 말하자면 어떤 오브젝트로부터 그 오브젝트와 관련된 오브젝트를 얻으려고 할 때 얻으려는 오브젝트에 대한 구현, Dependency를 몰라도 해당 오브젝트를 얻을 수 있다.       

```cpp

class IBattery{}
class SamsungBatteryS512 : IBattery {}
class AppleBatteryAB01 : IBattery {}

class IMemoryChip{}
class SamsungMemoryChipSM011 : IMemoryChip {}
class AppleMemoryChipAP13 : IMemoryChip {}

class Phone
{
    public:
    IBattery* CreateBattery() const = 0;
    IMemoryChip* CreateMemoryChip() const = 0;
}
class SamsungPhone : Phone
{
    public:
    IBattery* CreateBattery() const final
    {
        return new SamsungBatteryS512();
    }
    IMemoryChip* CreateMemoryChip() const final
    {
        return new SamsungMemoryChipSM011();
    }
}

class ApplePhone : Phone
{
    public:
    IBattery* CreateBattery() const final
    {
        return new AppleBatteryAB01();
    }
    IMemoryChip* CreateMemoryChip() const final
    {
        return new AppleMemoryChipAP13();
    }
}

--------------------

IBattery* GetBattery(const Phone* phone)
{
    return phone->CreateBattery();
}
IMemoryChip* GetMemoryChip(const Phone* phone)
{
    return phone->CreateMemoryChip();
}
```

위의 코드를 보면 이해가 쉽게 되리라고 생각한다. 팩토리라는 단어만 안들어갔지 CreateBattery, CreateMemoryChip 가상 함수가 사실상 팩토리의 역할을 한다고 생각하면 된다.                  
GetBattery, GetMemoryChip을 통해 Phone 오브젝트만 있으면 해당 Phone에 실제로 어떤 종류의 배터리, 메모리칩이 쓰이는지에 대한 구현, Dependency 없이 그 Phone에 맞는 배터리, 메모리칩을 얻을 수 있다.           



reference : [https://refactoring.guru/design-patterns/factory-method](https://refactoring.guru/design-patterns/factory-method)  