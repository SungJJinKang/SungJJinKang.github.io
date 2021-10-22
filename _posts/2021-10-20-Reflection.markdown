---
layout: post
title:  "리플랙션 시스템 구현, 고속 dynamic_cast ( 작성 중 )"
date:   2021-10-20
categories: ComputerScience
---

흔히 언리얼 엔진을 사용해보면 아주 편리함 시스템이 StaticClass, IsA와 같이 컴파일 타임이 아닌 런타임에 클래스간의 상속 관계를 판단하는 등의 여러 리플렉션 시스템이다.         
언리얼 엔진은 이를 위해 게임 내 모든 헤더파일을 분석, 파싱하여 "XXX.generated.h"라는 헤더 파일에 해당 타입에 대한 정보를 담는다.          
아마 이 글을 읽는 당신의 언리얼 헤더 파일들이 모두 "XXX.generated.h" 파일을 #include 하고 있는 것을 알 수 있을 것이다.        

이를 위해서는 타입에 대한 정보 ( 타입명, 타입의 유니크한 ID, 타입의 부모 클래스들의 정보, 멤버 함수 종류, 멤버 변수 종류, 멤버 함수의 주소 등등 )가 필요한데 C++에서는 이러한 정보를 제공해주지 않기 때문에 흔히 **리플랙션**이라고 부르는 시스템을 만들어서 이러한 타입 정보를 런타임에 접근할 수 있게 만든다.        
필자 또한 게임 엔진을 만들면서 런타임에 타입 정보를 가져와야할 필요가 있었다.      

또한 dynamic_cast가 너무 느리기 때문에 이를 대체할 것이 필요했다. dynamic_cast가 느린 이유는 dynamic_cast는 부모, 자식뿐만아니라 형제들까지 온갖 상속 관계를 뒤져서 캐스팅을 한다.       
그래서 이를 대체할 시간 복잡도 O(1)의 런타임 캐스팅을 만들기로 결심하였다.            


언리얼엔진의 리플랙션 시스템의 경우 자체적으로 에디터 단계에서 각각의 소스파일을 파싱해서 클래스들의 모든 정보를 분석한 후 빌드 단계에서 컴파일해서 빌드 파일에 담는다.         
유비소프트 몬테리올에서도 언리얼엔진과 같이 소스파일을 자동으로 파싱해서 해당 데이터를 dll 파일에 저장해둔 후 런타임에 해당 dll 파일을 로드해서 타입 정보를 읽어온다. ( 실제 유비소프트 몬테리올에서 사용하는 라이브러리 : [clReflect](https://github.com/Celtoys/clReflect) )                     

사실 나도 이 오픈소스를 그냥 사용하면된다.           

하지만 그냥 직접 만들어보고 싶어 직접 만들기로 결정하였다.      
그냥 만들어보고 싶었다.       

------------------

구현할 리플랙션 시스템의 목표를 몇가지 정했다.          
```
1. 컴파일 타임(!!!)에 리플랙션 데이터가 결정되어서 실행파일에 데이터로 담길 것. ( 리플랙션 시스템이 런타임 성능에 영향을 주지 않음 )
2. 느려터진 dynamic_cast를 개선할 것
```


언리얼엔진이나 clReflect와 같이 소스파일을 분석해서 자동으로 타입 정보를 완전히 자동화되지는 않았지만 클래스 정의 맨 앞에 어떤 **매크로를 추가하고 타입명을 넘기면** 언리얼과 같은 리플렉션, 타입 정보를 런타임에 확인할 수 있게 구현하였다.          
언리얼엔진, clReflect에서 컴퓨터가 해주는 일을 내가 만든 시스템에서는 프로그래머가 직접 해야하기 때문에 약간은 귀찮아보이지만, **각 클래스마다 딱 2줄의 코드만을 추가**하면 되기 때문에 이 정도면 납득 가능하다고 생각한다.          

아래는 리플랙션 시스템을 지원하기 위해 클래스에 추가해야할 매크로 코드이다.     
딱 2줄이다!!           
클래스들간의 상속관계 같은 경우 각 클래스마다 매크로로 해당 클래스의 부모 클래스 타입명을 넘겨주어야한다.      

```
class DOOM_API MeshCollider : public Collider3DComponent
{
	DOBJECT_CLASS_BODY(MeshCollider)
	DOBJECT_CLASS_BASE_CHAIN(Collider3DComponent)
}
```

이 두줄의 코드를 통해 엔진내 거의 모든 클래스들의 타입 정보를 DClass라는 클래스로 객체화하였다. 클래스명, 클래스들간의 상속 관계 등의 여러 클래스 타입 정보를 런타임에 얻을 수 있다.         

다만 클래스 내부의 변수나 함수들의 정보를 런타임에 가져오지는 못한다.           
구현하려면 할 수는 있지만 현재 필자의 프로젝트에서는 아직까지 필요하지 않아 차후에 필요시 추가할 예정이다.     

--------------------

우선 클래스 타입들을 구분해주기 위해 각 클래스들을 구분해줄 유니크한 ID가 필요했다.       
어떻게 이를 얻을 수 있을까?    
```
const std::type_info& ti1 = typeid(A);
const std::type_info& ti2 = typeid(A);
 
assert(&ti1 == &ti2); // not guaranteed
assert(ti1.hash_code() == ti2.hash_code()); // guaranteed
assert(std::type_index(ti1) == std::type_index(ti2)); // guaranteed
```
C++에서는 std::type_info의 hash_code()와 std::type_index를 통해 클래스 타입들의 유니크한 ID를 얻을 수 있다.       
그러나 **이것들은 모두 런타임에 결정 ( 프로그램 시작 직후 )되는 데이터**다.        
**필자가 구현하려는 리플랙션 시스템은 컴파일 타임에 모든 것들이 결정되어야 하기 때문에 다른 방법이 필요**했다.          

좋은 방법이 떠올랐다.       
위의 리플랙션 매크로에서 넘겼던 **클래스명을 문자열화해서 해당 Literal 문자열 ( 상수 문자열 )의 시작 주소를 유니크 ID로 사용**하면 될 것 같다.    
같은 클래스명을 가진 클래스는 존재할 수 없으니 당연히도 유니크함이 보장된다.         
```
#define TYPE_ID_IMP(CLASS_TYPE)																							\
		public:																											\
		FORCE_INLINE static constexpr const char* CLASS_TYPE_ID_STATIC() {												\
			return #CLASS_TYPE;	            																			\
		}																												\
        virtual const char* GetClassTypeID() const { return CLASS_TYPE::CLASS_TYPE_ID_STATIC(); }		
```

성공적으로 **클래스 타입들의 각각의 유니크 ID를 컴파일타임에 가져올 수 있었다.**                      
당연히 두 클래스 타입의 유니크ID를 가지고 컴파일 타임에 비교하는 것도 가능하다.          

GodBolt를 통해 어셈블리어를 확인해보면 유니크 ID를 얻을 시 아래와 같이 프로그램내의 문자열의 위치를 데이터 심볼로 가져오는 것을 알 수 있다.       
```
class A
{
    TYPE_ID_IMP(A)
};

class B
{
    TYPE_ID_IMP(B)
};

int main()
{
    int c = 0;
    constexpr const char* a = A::CLASS_TYPE_ID_STATIC();
    constexpr const char* b = B::CLASS_TYPE_ID_STATIC();
    if constexpr(a != b)
    {
        c = 1;
    }
    else
    {
        c = 2;
    }
}
```
-->
```
$SG23072 DB     'A', 00H  <--- !!!!!!!!!!
        ORG $+2
$SG23073 DB     'B', 00H  <--- !!!!!!!!!!

c$ = 0
a$ = 8
b$ = 16
main    PROC
$LN3:
        sub     rsp, 40                             ; 00000028H
        mov     DWORD PTR c$[rsp], 0
        lea     rax, OFFSET FLAT:$SG23072 <--- !!!!!!!!!!
        mov     QWORD PTR a$[rsp], rax
        lea     rax, OFFSET FLAT:$SG23073 <--- !!!!!!!!!!
        mov     QWORD PTR b$[rsp], rax
        mov     DWORD PTR c$[rsp], 1
        xor     eax, eax
        add     rsp, 40                             ; 00000028H
        ret     0
main    ENDP
```


----------------------------          
     

또한 구현한 Reflection 시스템을 통해 **매우 느려터진 dynamic_cast도 대체할 수 있다.**                 
```
namespace doom
{
	namespace details
	{
		template <typename BASE_DOBJECT_TYPE_CLASS>
		static constexpr void BASE_CHAIN_HILLCLIMB_COUNT(SIZE_T& base_chain_count)
		{
			base_chain_count++;
			if constexpr (std::is_same_v<doom::DObject, BASE_DOBJECT_TYPE_CLASS> == false) {
				BASE_CHAIN_HILLCLIMB_COUNT<typename BASE_DOBJECT_TYPE_CLASS::Base>(base_chain_count);
			}
		}

		template <typename BASE_DOBJECT_TYPE_CLASS>
		static constexpr SIZE_T BASE_CHAIN_HILLCLIMB_COUNT()
		{
			SIZE_T base_chain_count = 1;
			if constexpr (std::is_same_v <doom::DObject, BASE_DOBJECT_TYPE_CLASS > == false) {
				BASE_CHAIN_HILLCLIMB_COUNT<typename BASE_DOBJECT_TYPE_CLASS::Base>(base_chain_count);
			}
			return base_chain_count;
		}

		template <typename BASE_DOBJECT_TYPE_CLASS, SIZE_T COUNT>
		static constexpr void BASE_CHAIN_HILLCLIMB_DATA(SIZE_T& count, std::array<const char*, COUNT>& chain_data)
		{
			chain_data[count] = BASE_DOBJECT_TYPE_CLASS::__CLASS_TYPE_ID;
			count++;
			if constexpr (std::is_same_v<doom::DObject, BASE_DOBJECT_TYPE_CLASS> == false) {
				BASE_CHAIN_HILLCLIMB_DATA<typename BASE_DOBJECT_TYPE_CLASS::Base>(count, chain_data);
			}
		}

		template <typename BASE_DOBJECT_TYPE_CLASS, SIZE_T COUNT>
		static constexpr std::array<const char*, COUNT> BASE_CHAIN_HILLCLIMB_DATA()
		{
			std::array<const char*, COUNT> chain_data{};
			chain_data[0] = BASE_DOBJECT_TYPE_CLASS::__CLASS_TYPE_ID;
			if constexpr (std::is_same_v <doom::DObject, BASE_DOBJECT_TYPE_CLASS > == false) {
				SIZE_T count = 1;
				BASE_CHAIN_HILLCLIMB_DATA<typename BASE_DOBJECT_TYPE_CLASS::Base>(count, chain_data);
			}
			return chain_data;
		}
	}
}




#define DOBJECT_CLASS_BASE_CHAIN(BASE_DOBJECT_TYPE_CLASS)													\
	public:																									\
	using Base = BASE_DOBJECT_TYPE_CLASS; /* alias Base DObject Type Class */								\
	private:																								\
	constexpr static SIZE_T _BASE_CHAIN_COUNT = doom::details::BASE_CHAIN_HILLCLIMB_COUNT<Current>();		\
	constexpr static std::array<const char*, _BASE_CHAIN_COUNT> _BASE_CHAIN_DATA = doom::details::BASE_CHAIN_HILLCLIMB_DATA<Current, _BASE_CHAIN_COUNT>();			\
	public:																									\
	FORCE_INLINE constexpr static SIZE_T BASE_CHAIN_COUNT_STATIC()											\
	{																										\
		return _BASE_CHAIN_COUNT;																			\
	}																										\
	FORCE_INLINE constexpr static const char* const * const BASE_CHAIN_DATA_STATIC()						\
	{																										\
		return _BASE_CHAIN_DATA.data();																		\
	}																										\
	virtual SIZE_T GetBaseChainCount() const { return BASE_CHAIN_COUNT_STATIC(); }							\
	virtual const char* const * const GetBaseChainData() const { return BASE_CHAIN_DATA_STATIC(); }

```

복잡해보이지만 별거 없다.      
클래스마다 상속하는 클래스 타입을 적어주면 된다.      

```
    class DOOM_API MeshCollider : public Collider3DComponent, StaticContainer
    {
        DOBJECT_CLASS_BODY(MeshCollider)
        DOBJECT_CLASS_BASE_CHAIN(Collider3DComponent) <- !!!!!
    }
```

필자의 게임 엔진에서는 거의 모든 클래스들이 DObject라는 루트 클래스에서 뻗어나간다. ( Unreal Engine의 UObject와 비슷하다 )          
위의 _BASE_CHAIN에는 현재 클래스 타입이 상속 중인 부모 타입을 타고 올라가서 루트 클래스 타입인 DObject 클래스까지의 클래스 Type ID를 저장한다.       

상속 관계에 관한 데이터가 전역변수로 저장되어 컴파일 타임에 부모 클래스를 타고 올라가면서 각 부모 클래스의 유니크한 타입 ID를 저장한다.           
그럼 해당 컨테이너는 이러한 데이터 형태를 가질 것이다.        
```
( 현재 속해 있는 클래스 유니크 타입 ID ) ( 부모 클래스 유니크 타입 ID ) ( 조부모 클래스 유니크 타입 ID ) ( 증조부모 클래스 유니크 타입 ID ) ( 고조부모 클래스 유니크 타입 ID )
```

**중요한 것은 위의 클래스 Hierarchy 데이터가 컴파일 타임에 다 결정**된다는 것이다!!

그럼 어떤 오브젝트가 포인터로 넘어왔을 때 그 오브젝트가 어떤 다른 클래스의 자식인지를 어떻게 알 수 있을까?      
방법은 간단하다.      

```
부모 리스트 컨테이너 [ From 캐스팅 오브젝트의 부모들의 개수 ( 깊이 ) - To 캐스팅 클래스의 부모들의 개수 ( 깊이 )  ] == To 캐스팅 클래스의 타입 ID
```

이를 통해 **모든 부모, 조상들의 클래스 Hierarchy 를 탐색 ( 순회 )하지 않고 O(1)만에 비교하려는 클래스가 현재 오브젝트의 부모인지 아닌지를 확인**할 수 있다.     
참고로 이 방법은 언리얼 엔진에서 차용한 방법으로 매우 빠르게 수직 관계의 클래스들간의 런타임 캐스팅을 구현하게 해준다.         
물론 부모 타입으로 캐스팅하는 경우에는 컴파일타임에 바로 타입 변환을 한다. 위의 알고리즘은 부모 타입의 포인터에서 자녀 타입으로 캐스팅을 할 경우에만 사용된다.        

당연히 RTTI 생성 컴파일러 옵션도 완전히 껐다.         

실제 코드이다.     
```
template <typename BASE_TYPE>
FORCE_INLINE bool IsChildOf() const
{
    static_assert(IS_DOBJECT_TYPE(BASE_TYPE));
    
    const bool isChild = ( GetBaseChainCount() >= BASE_TYPE::BASE_CHAIN_COUNT_STATIC() ) && ( GetBaseChainData()[GetBaseChainCount() - BASE_TYPE::BASE_CHAIN_COUNT_STATIC()] == BASE_TYPE::CLASS_TYPE_ID_STATIC() );

    return isChild;
}
```

또한 저 막대한 코드를 만들어낼 HILL_CLIMB 템플릿 함수는 컴파일 타임 연산에서만 호출되고 런타임에는 호출되지 않으니 실행파일에는 빠지게되니 템플릿 Bloat를 가져오지도 않는다. ( 정확하지 않으니 나중에 한번 더 확인을 해보아야겠다.)        

-> 후에 확인하였더니 HILL_CLIMB 함수에 extern이 아닌 static 옵션을 주어서 해당 함수가 컴파일하는 소스파일마다 각자 정의를 가지고 해당 소스파일 내에서만 사용된다는 힌트를 주니 컴파일러가 HILL_CLIMB 함수를 .obj 파일에서 제외시켰다!!!!       
혹시나 인라이닝 때문에 이 함수들이 없어져 보이는 것이 아닌가 걱정되어서 인라이닝을 금지하는 컴파일러 옵션을 준 후에도 확인해보았다. 여전히 해당 함수들은 제거되어 있었다.        

또한 언리얼 엔진과 같이 부모 클래스 타입을 임의로 적어줄 필요없이 "Base" type alias로 부모 클래스의 멤버에 접근할 수 있다.      
```
using Base = BASE_DOBJECT_TYPE_CLASS; /* alias Base DObject Type Class */							
```       


결과적으로 **모든 목표를 달성**했다.         
언리얼엔진과 같이 모든 클래스 타입 정보를 컴파일 타임에 저장하고 런타임에 가져올 수 있다.        
또한 dynamic_cast 없이 안전하고 매우 빠른 타입 캐스팅을 구현하였다.      
필자가 제시하는 방법이 절대 최선의 방법은 아니고 직접 고안한 방식이다보니 허점이 많을 수도 있다.       

아래는 필자가 작성한 코드이다.        

[소스코드1](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObject.h)          
[소스코드2](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObjectGlobals.h)       
[소스코드3](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObjectMacros.h)        
[소스코드4](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DClass.h)