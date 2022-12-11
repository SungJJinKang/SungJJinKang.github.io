---
layout: post
title:  "Unreal Engine CreateDefaultSubObject 함수의 두 번째 매개 변수 bTransient의 의미 ( FEventLoadNodeArray::GetAddedNodes 크래시 )"
date:   2022-12-11
categories: ComputerScience ComputerGraphics UE UnrealEngine
---            
                     
회사 업무 중 크래시 리포트를 보다 발견한 이슈이다.                   
                
<img width="421" alt="20221211161302" src="https://user-images.githubusercontent.com/33873804/206891051-f0a08f91-4727-4d1f-a0b7-8776155aa17e.png">              
           
원인을 찾기 힘들어 UDN을 검색하던 중 해당 크래시의 원인을 알 수 있었다. ( 다른 원인이 존재할 가능성도 있다. )           
                   
원인은 Transient로 선언된 멤버 변수에 CreateDefaultSubObject로 인스턴스를 할당할 때 CreateDefaultSubObject의 두 번째 매개변수 bTransient를 true가 아닌 false를 사용한 것이 문제였다. ( 해당 매개변수의 기본 값이 false로 설정되어 있다. )           

```cpp
UCLASS()
class SampleClass : public UObject
{
    GENERATED_BODY()

    UPROPERTY(Transient)
    USampleComponent* Comp;
};

SampleClass::SampleClass()
{
    Comp = CreateDefaultSubobject<USampleComponent>(TEXT("SampleComponent")); // 두 번째 매개변수를 기본 값인 falsr를 사용!
}

```
       
"Transient"는 해당 멤버 변수가 시리얼라이즈되지 않는다는 것을 의미한다. 즉 저장 또는 로드되지 않는다는 것이다. 이런 식의 지정자가 붙은 프로퍼티는 로드 시간에 0 으로 채워집니다.          
            
CreateDefaultSubObject 함수의 주석을 보아도 반환되는 오브젝트가 transient 프로퍼티에 셋팅되는 경우 "bTransient" 매개 변수에 true를 전달하라고 적혀 있다.                       
그리고 **컴포넌트 그 자체를 transient하게 만들지는 않지만, 컴포넌트가 부모 Default의 값을 상속 받아 오는 것을 막는다고 되어 있다.**         
무슨 말인지 잘 이해가 안되어서 코드를 더 살펴보니 컴포넌트를 붙이는 오브젝트(위의 예제에서는 SampleClass의 인스턴스)의 아키타입(ex. BP 클래스)으로부터 SubObject "SampleComponent"의 데이터를 가져올 것이냐는 것이다. ( 컴포넌트를 붙이는 오브젝트의 아키타입의 해당 컴포넌트 SubObject를 템플릿으로 사용할 것이냐는 것이다. )         
**컴포넌트를 붙이는 오브젝트(위의 예제에서는 SampleClass의 인스턴스)의 BP 클래스에서 Comp의 여러 변수 값들을 셋팅하였더라도 이 값을 상속 받아오지 않는다는 의미**이다. 그냥 해당 컴포넌트의 CDO로부터 컴포넌트의 프로퍼티들을 초기화하겠다는 의미이다.               
코드를 보아도 "bTransient"가 true인 경우 오브젝트의 아키타입에서 DefaultSubObject를 가져와서 컴포넌트를 초기화하는(ComponentInits) 부분이 빠지는 것을 볼 수 있다.           
         
요약하자면 CreateDefaultSubobject의 매개변수 bTransient에 true를 전달한 경우 **해당 컴포넌트의 프로퍼티 전체가 transient로 처리, 즉 0으로 초기화되는건 아니고, 해당 컴포넌트의 프로퍼티들은 CDO에서의 값들로 초기화**된다.                               
**CreateDefaultSubobject의 매개변수 bTransient를 false로 전달했을 때와의 차이점**은 **false로 전달하는 경우에는 컴포넌트를 붙이는 오브젝트의 아키타입(BP 클래스)에서 셋팅한 해당 컴포넌트의 프로퍼티 값들을 상속** 받지만, **true로 전달하는 경우에는 해당 컴포넌트 클래스의 CDO를 상속 받는다.**                                                                                 
            
```cpp
/**
* Create a component or subobject.
* @param	TReturnType					Class of return type, all overrides must be of this type
* @param	SubobjectName				Name of the new component
* @param	bTransient					True if the component is being assigned to a transient property. This does not make the component itself transient, but does stop it from inheriting parent defaults
*/
template<class TReturnType>
TReturnType* UObject::CreateDefaultSubobject(FName SubobjectName, bool bTransient = false)
{
UClass* ReturnType = TReturnType::StaticClass();
return static_cast<TReturnType*>(CreateDefaultSubobject(SubobjectName, ReturnType, ReturnType, /*bIsRequired =*/ true, bTransient));
}
```

<img width="752" alt="20221211161407" src="https://user-images.githubusercontent.com/33873804/206891048-5c273738-715a-4e9f-8033-151ecf82578c.png">                             
<img width="819" alt="5" src="https://user-images.githubusercontent.com/33873804/206891671-dbd0da50-de11-497b-801e-9db0b644c775.png">        

