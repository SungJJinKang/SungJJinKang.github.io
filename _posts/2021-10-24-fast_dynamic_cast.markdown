---
layout: post
title:  "고속 dynamic_cast"
date:   2021-10-24
categories: ComputerScience C++
---

```
namespace __fast_runtime_type_casting_details
{
	//!!!!!!!!!!!!
	//Never change static to extern. static give hint to compiler that this definition is used only in source file(.cpp)
	//								 Then Compiler remove this functions definition from compiler if it is called only at compile time
	template <typename BASE_DOBJECT_TYPE_CLASS>
	static constexpr void BASE_CHAIN_HILLCLIMB_COUNT(size_t& base_chain_count)
	{
		base_chain_count++;
		if constexpr (std::is_same_v<FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS, BASE_DOBJECT_TYPE_CLASS> == false) {
			BASE_CHAIN_HILLCLIMB_COUNT<typename BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_BASE_TYPE>(base_chain_count);
		}
	}

	template <typename BASE_DOBJECT_TYPE_CLASS>
	static constexpr size_t BASE_CHAIN_HILLCLIMB_COUNT()
	{
		size_t base_chain_count = 1;
		if constexpr (std::is_same_v <FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS, BASE_DOBJECT_TYPE_CLASS > == false) {
			BASE_CHAIN_HILLCLIMB_COUNT<typename BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_BASE_TYPE>(base_chain_count);
		}
		return base_chain_count;
	}

	template <typename BASE_DOBJECT_TYPE_CLASS, size_t COUNT>
	static constexpr void BASE_CHAIN_HILLCLIMB_DATA(size_t& count, std::array<const char*, COUNT>& chain_data)
	{
		chain_data[count] = BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_CLASS_TYPE_ID;
		count++;
		if constexpr (std::is_same_v<FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS, BASE_DOBJECT_TYPE_CLASS> == false) {
			BASE_CHAIN_HILLCLIMB_DATA<typename BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_BASE_TYPE>(count, chain_data);
		}
	}

	template <typename BASE_DOBJECT_TYPE_CLASS, size_t COUNT>
	static constexpr std::array<const char*, COUNT> BASE_CHAIN_HILLCLIMB_DATA()
	{
		std::array<const char*, COUNT> chain_data{};
		chain_data[0] = BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_CLASS_TYPE_ID;
		if constexpr (std::is_same_v <FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS, BASE_DOBJECT_TYPE_CLASS > == false) {
			size_t count = 1;
			BASE_CHAIN_HILLCLIMB_DATA<typename BASE_DOBJECT_TYPE_CLASS::__FAST_RUNTIME_TYPE_CASTING_BASE_TYPE>(count, chain_data);
		}
		return chain_data;
	}
}




#define __FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BASE_CHAIN(BASE_DOBJECT_TYPE_CLASS)							\
	static_assert(std::is_same_v<__FAST_RUNTIME_TYPE_CASTING_CURRENT_TYPE, BASE_DOBJECT_TYPE_CLASS> == false);	\
	public:																										\
	using __FAST_RUNTIME_TYPE_CASTING_BASE_TYPE = BASE_DOBJECT_TYPE_CLASS; /* alias base FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS Type Class */	\
	private:																									\
	constexpr static size_t __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT = __fast_runtime_type_casting_details::BASE_CHAIN_HILLCLIMB_COUNT<__FAST_RUNTIME_TYPE_CASTING_CURRENT_TYPE>();		\
	constexpr static const std::array<const char*, __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT> __FAST_RUNTIME_TYPE_CASTING__BASE_CHAIN_DATA = __fast_runtime_type_casting_details::BASE_CHAIN_HILLCLIMB_DATA<__FAST_RUNTIME_TYPE_CASTING_CURRENT_TYPE, __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT>();			\
	public:																									\
	D_FORCE_INLINE constexpr static size_t __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT_STATIC()			\
	{																										\
		return __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT;												\
	}																										\
	D_FORCE_INLINE constexpr static const char* const * __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_DATA_STATIC()\
	{																										\
		return __FAST_RUNTIME_TYPE_CASTING__BASE_CHAIN_DATA.data();											\
	}																										\
	virtual size_t __FAST_RUNTIME_TYPE_CASTING_GET_BASE_CHAIN_COUNT() const { return __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT_STATIC(); }	\
	virtual const char* const * __FAST_RUNTIME_TYPE_CASTING_GET_BASE_CHAIN_DATA() const {					\
	static_assert(std::is_base_of_v<BASE_DOBJECT_TYPE_CLASS, std::decay<decltype(*this)>::type> == true, "Current Class Type is not derived from Passed Base ClassType is passed");	\
	return __FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_DATA_STATIC(); }

/////////////////////////////////

#ifndef FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BODY

#define FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BODY(CURRENT_CLASS_TYPE, BASE_CLASS_TYPE)	\
		public:																		\
		static_assert(std::is_base_of_v<FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS, BASE_CLASS_TYPE>, "Base ClassType is not derived from DObejct");	\
		using __FAST_RUNTIME_TYPE_CASTING_CURRENT_TYPE = CURRENT_CLASS_TYPE;				\
		__FAST_RUNTIME_TYPE_CASTING_TYPE_ID_IMP(CURRENT_CLASS_TYPE)							\
		__FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BASE_CHAIN(BASE_CLASS_TYPE)

#endif

```

복잡해보이지만 별거 없다.      
클래스마다 상속하는 클래스 타입을 적어주면 된다.      

```
class Collider3DComponent : public FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS
{
	FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BODY(Collider3DComponent, FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS) <- Pass Current Class Name, Base Class Name
}
class MeshCollider : public Collider3DComponent
{
	FAST_RUNTIME_TYPE_CASTING_DOBJECT_CLASS_BODY(MeshCollider, Collider3DComponent) <- Pass Current Class Name, Base Class Name
}

Collider3DComponent* object = new MeshCollider();

MeshCollider* meshCol = CastTo<MeshCollider*>(object);

if(object->IsChildOf<MeshCollider>() == true)
{
	~~
}
```

이 알고리즘은 언리얼 엔진에서 영향을 받은 것으로 실제 언리얼 엔진의 UObject 타입간 런타임 타입 캐스팅도 이와 같은 방법으로 구현되어 있다.     
다만 언리얼 엔진은 클래스가 Hierarchy 정보를 얻기 위해 파싱을 위한 다른 외부 툴을 사용하는데 반해 내 방법은 외부 툴이 필요없고 어느 컴파일러나 Portable하게 적용 가능하다.         

필자의 게임 엔진에서는 거의 모든 클래스들이 DObject라는 루트 클래스에서 뻗어나간다. ( Unreal Engine의 UObject와 비슷하다 )          
위의 _BASE_CHAIN에는 현재 클래스 타입이 상속 중인 부모 타입을 타고 올라가서 루트 클래스 타입인 DObject 클래스까지의 클래스 Type ID를 저장한다.       

**상속 관계에 관한 데이터가** 전역변수로 저장되어 **컴파일 타임**에 부모 클래스를 타고 올라가면서 각 부모 클래스의 유니크한 타입 ID를 **저장**한다.           
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
물론 부모 타입으로 캐스팅하는 경우에는 컴파일타임에 바로 타입 변환을 한다. 위의 알고리즘은 부모 타입의 포인터에서 자녀 타입으로 캐스팅을 할 경우에만 사용된다.        

<img width="437" alt="20211023013700" src="https://user-images.githubusercontent.com/33873804/138491569-e507bfb8-be3b-4d3e-989e-54abe565a927.png">          

벤치마크 결과 dynamic_cast에 비해 2배 이상 빠른 것으로 확인되었다.           

당연히 RTTI 생성 컴파일러 옵션도 완전히 껐다.         

이 코드는 해당 오브젝트가 특정 클래스의 자녀 타입의 오브젝트인지 확인하는 함수이다.         
```
template <typename BASE_TYPE>
D_FORCE_INLINE bool IsChildOf() const
{
	static_assert(IS_DERIVED_FROM_FAST_RUNTIME_TYPE_CASTING_ROOT_CLASS(BASE_TYPE));

	const bool isChild = (__FAST_RUNTIME_TYPE_CASTING_GET_BASE_CHAIN_COUNT() >= BASE_TYPE::__FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT_STATIC()) && (__FAST_RUNTIME_TYPE_CASTING_GET_BASE_CHAIN_DATA()[__FAST_RUNTIME_TYPE_CASTING_GET_BASE_CHAIN_COUNT() - BASE_TYPE::__FAST_RUNTIME_TYPE_CASTING_BASE_CHAIN_COUNT_STATIC()] == BASE_TYPE::__FAST_RUNTIME_TYPE_CASTING_CLASS_TYPE_ID_STATIC());

	return isChild;
}
```

또한 저 막대한 코드를 만들어낼 HILL_CLIMB 템플릿 함수는 컴파일 타임 연산에서만 호출되고 런타임에는 호출되지 않으니 실행파일에는 빠지게되니 템플릿 Bloat를 가져오지도 않는다. ( 정확하지 않으니 나중에 한번 더 확인을 해보아야겠다.)        

-> 후에 확인하였더니 HILL_CLIMB 함수에 extern이 아닌 static 옵션을 주어서 해당 함수가 컴파일하는 소스파일마다 각자 정의를 가지고 해당 소스파일 내에서만 ( internal linking ) 사용된다는 힌트를 주니 컴파일러가 HILL_CLIMB 함수를 .obj 파일에서 제외시켰다!!!!       
혹시나 인라이닝 때문에 이 함수들이 없어져 보이는 것이 아닌가 걱정되어서 인라이닝을 금지하는 컴파일러 옵션을 준 후에도 확인해보았다. 여전히 해당 함수들은 제거되어 있었다.           


일일이 매크로를 클래스마다 써주어야된다는 점, 다중 상속을 지원하지 않는다는 문제점이 있지만 프로젝트에 따라서는 이 방법이 적합한 방법일 수 있다고 생각한다. ( 실제로 내 게임 엔진 프로젝트에서는 이 방법이 충분히 적용 가능하고, 현재 사용중이다. )        

오브젝트 파일, exe 파일이 커질 수 있다는 단점이 존재하지만 이 고속 dynamic_cast를 사용한다면 컴파일러 RTTI 옵션을 끌 수 있으니 어느 정도 Trade-Off 라고 생각한다. ( 사실 RTTI의 데이터 사이즈를 알지 못한다. )      


[레딧에도 올려봤다.](https://www.reddit.com/r/cpp/comments/qenrtn/i_wrote_runtime_time_casting_code_faster_than/)          
      
[소스코드](https://github.com/SungJJinKang/Fast_Runtime_Type_Casting_cpp)          