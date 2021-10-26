---
layout: post
title:  "C++ 리플랙션 ( 작성 중 )"
date:   2021-10-20
categories: ComputerScience C++
---

이 글은 C++ 리플랙션 시스템을 직접 구현하기 위한 여러 시도들을 포함하고 있습니다.          

그러나 매크로, 템플릿을 사용하는 방법의 한계를 깨닫고 [clReflect](https://github.com/Celtoys/clReflect)라는 clang의 컴파일러 내부 데이터 ( Compliatation Database )를 활용하는 방식으로 대체를 할 예정입니다. 이에 대한 글도 향후 쓸 예정이니 참고해주세요.


----------

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

또한 클래스의 멤버함수, 멤버변수에도 텍스트를 이용해서 접근할 수 있다.       


아래는 필자가 작성한 코드이다.        

[소스코드1](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObject.h)          
[소스코드2](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObjectGlobals.h)       
[소스코드3](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DObjectMacros.h)        
[소스코드4](https://github.com/SungJJinKang/DoomsEngine/blob/main/Doom3/Source/Core/DObject/DClass.h)