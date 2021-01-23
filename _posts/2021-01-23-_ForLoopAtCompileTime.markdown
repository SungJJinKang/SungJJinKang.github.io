---
layout: post
title:  "For Loop At CompileTime"
date:   2021-01-23
categories: C++
---

순조롭게 Doom From Scratch를 개발하던 중 귀찮은일이 생겼다.

Enum Type value를 template argument로 받는 템플릿 function을 Enum의 모든 Element에 대해 한번씩 호출해줘야하는 일이 생겼다.
물론 아래의 방법과 같이 수동으로 내가 직접해주어도 된다.

```c++
enum AssetType : unsigned int
{
	AUDIO = 0,
	FONT,
	TEXT,
	TEXTURE,
	THREE_D_MODEL,
	SHADER,
};

ImportAssetAndAddToContainer<Asset::AssetType::AUDIO>(AssetPaths[Asset::AssetType::AUDIO]);
ImportAssetAndAddToContainer<Asset::AssetType::FONT>(AssetPaths[Asset::AssetType::FONT]);
ImportAssetAndAddToContainer<Asset::AssetType::TEXT>(AssetPaths[Asset::AssetType::TEXT]);
ImportAssetAndAddToContainer<Asset::AssetType::TEXTURE>(AssetPaths[Asset::AssetType::TEXTURE]);
ImportAssetAndAddToContainer<Asset::AssetType::THREE_D_MODEL>(AssetPaths[Asset::AssetType::THREE_D_MODEL]);
ImportAssetAndAddToContainer<Asset::AssetType::SHADER>(AssetPaths[Asset::AssetType::SHADER]);
```
누가봐도 지저분하다.

그래서 결국 나는 ForLoop문을 Compile Time에 돌아가게 하여 그 Loop 변수를 템플릿 argument로 넘겨줄 생각이다.

```c++
for(int i = 0 ; i < 6 ; i++)
{
    ImportAssetAndAddToContainer<i>(AssetPaths[i]); // Loop variable ( for(int i = 0 i < 10 ; i++) 에서 i를 의미하는 명칭 ) 인 i 또한 constant value여야한다!!
}
```

이렇게 말이다.

위의 코드는 당연히 컴파일 오류가 뜬다.

template argument는 컴파일 타임에 결정되야하는데 C++에서 for constexpr 같은 기능을 지원하는 것도 아니기 때문에 이러한 작업을 도와주는 라이브러리 찾아보았다.

이걸 도와주는 라이브러리는 몇개 있지만 나는 Loop variable ( for(int i = 0 i < 10 ; i++) 에서 i를 의미하는 명칭 )을 Loop Job 내에서 Constant value로 사용해야하는데 이것 까지 지원되는 라이브러리를 찾지 못하여 직접 만들기로 하였다.

for loop의 초기값과 조건값, 증감값을 모두 non type template argumnet로 전달해 컴파일 타입에 결정되게 하고 내부적으로는 recursive function 형태로 Loop를 실행하는 것이다.

거기다가 Loop variable의 type도 integer부터 enum까지 다양한 타입을 지원할 예정이다.

아쉽게도 floating 타입은 값이 컴파일 타입에 결정되지 않아 template argumnet로 전달할 수 없기 때문에 floating loop variable은 지원하지 않는다. (C++20에서는 floating point의 value가 컴파일 타임  )

몇시간 동안 작업한 끝에 아래와 같은 함수를 만들 수 있었다.

```c++
template <typename LoopVariableType>
struct ForLoop_CompileTime<typename LoopVariableType, std::enable_if_t<std::is_enum_v<LoopVariableType>> >
{
	using enum_underlying_type = std::underlying_type_t<LoopVariableType>;

	template <LoopVariableType start, LoopVariableType end, typename enum_underlying_type increment, template<LoopVariableType> typename Functor, typename... Args, std::enable_if_t<start <= end, bool> = true >
	static void Loop(Args&&... args)
	{
		Functor<start>()(std::forward<Args>(args)...);

		if constexpr (static_cast<enum_underlying_type>(start) + increment <= static_cast<enum_underlying_type>(end))
		{
			Loop<static_cast<LoopVariableType>(static_cast<enum_underlying_type>(start) + increment), end, increment, Functor>(std::forward<Args>(args)...);
		}
	}
};

enum EnumTest
{
	A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P
}

template<EnumTest enumValue>
struct LoopJobFunctor
{
	constexpr void operator()()
	{
		Function<enumValue>();
	}
};

int main()
{
	ForLoop_CompileTime<EnumTest>::Loop<EnumTest::A, EnumTest::P, 1, LoopJobFunctor>();
}
```

상당히 복잡해 보이지만 기능은 간단하다.

함수 Loop의 non type template argumnet로 전달된 start, end, increment, Functor, argument를 통해 컴파일 타입의 Loop를 재귀적함수로 구현하는 것이다.

Non type template argument를 가지는 Functor을 이용하면 looped value를 컴파일 타임에 가져갈 수 있다.

매번 Functor을 만드는 것이 약간은 귀찮게 보일 수 있지만 모든 Enum element를 일일이 써주는 것보다는 훨씬 편하다고 생각한다.

그럼 이제 내 Doom 코드에서 사용해보자.

```c++
enum AssetType : unsigned int
{
	AUDIO = 0,
	FONT,
	TEXT,
	TEXTURE,
	THREE_D_MODEL,
	SHADER,
};

ImportAssetAndAddToContainer<Asset::AssetType::AUDIO>(AssetPaths[Asset::AssetType::AUDIO]);
ImportAssetAndAddToContainer<Asset::AssetType::FONT>(AssetPaths[Asset::AssetType::FONT]);
ImportAssetAndAddToContainer<Asset::AssetType::TEXT>(AssetPaths[Asset::AssetType::TEXT]);
ImportAssetAndAddToContainer<Asset::AssetType::TEXTURE>(AssetPaths[Asset::AssetType::TEXTURE]);
ImportAssetAndAddToContainer<Asset::AssetType::THREE_D_MODEL>(AssetPaths[Asset::AssetType::THREE_D_MODEL]);
ImportAssetAndAddToContainer<Asset::AssetType::SHADER>(AssetPaths[Asset::AssetType::SHADER]);
```

-->


```c++

enum AssetType : unsigned int
{
	AUDIO = 0,
	FONT,
	TEXT,
	TEXTURE,
	THREE_D_MODEL,
	SHADER,
};

template<AssetType loopVariable>
struct LoopJobFunctor
{
	constexpr void operator()()
	{
		ImportAssetAndAddToContainer<loopVariable>(AssetPaths[loopVariable]);
        ~~~~~~~
	}
};

static constexpr inline AssetType FirstElementOfAssetType = AssetType::AUDIO;
static constexpr inline AssetType LastElementOfAssetType = AssetType::SHADER; // enum의 element가 추가되면 이것 일일이 바꾸어 줘야한다 ( 아쉽게도 현재로서는 enum의 마지막 element를 자동으로 얻을 수 있는 방법이 없어보인다.)

ForLoop_CompileTime<AssetType>::Loop<FirstElementOfAssetType, LastElementOfAssetType, 1, LoopJobFunctor>();

```

위의 코드를 아래의 코드 같이 짦게 만들 수 있었다.

코드 줄 수는 큰 차이가 없어보이지만 enum의 element가 50개라고 생각해보자. Functor의 작업에 대해 일일히 50번을 코드로 복사 붙여넣기 한다고 생각하면 위의 코드는 정말 지저분해질 것이다

아래의 코드가 매번 Functor을 만들어줘야하는 것이 귀찮을 수도 있지만 코드가 훨씬 깔끔해지고 가독성이 높아진다.

라이브러리는 깃허브에 올려두었으니 필요하면 가져다 쓰기를 바랍니다...

[Github Repo](https://github.com/SungJJinKang/ForLoop_Compile_Time)


------------------

이 라이브러리를 레딧에 올려보니 어떤 분이 이런 코드를 올려주었다.

솔직히 나는 std::integer_sequence 이라는게 있는지도 몰랐다...

정말 C++을 파면 팔 수록 C++은 너무 방대하다는 느낌이 든다.


솔직히 내 코드가 가독성이 더 좋고 쉽게 읽힌다고 생각한다.

```c++
// Generic, reusable utility

template<auto Offset, typename IntSeq>
struct offset_integer_sequence {};

template<auto Offset, typename IntT, IntT ... Ints>
struct offset_integer_sequence<Offset, std::integer_sequence<IntT, Ints...>> {
    using type = std::integer_sequence<Int, (Ints + Offset) ...>;
};

template<auto Offset, typename IntSeq>
using offset_integer_sequence_t = typename offset_integer_sequence<Offset, IntSeq>::type;


// As in the library
// No super-linear template instantiations (unless the implementation's make_integer_sequence does).

template<typename Functor, typename IntT, IntT ... Indices>
void apply_functor_helper(std::index_sequence<IntT, Indices ...>) {
    (Functor<Indices>(), ...);
}

template<typename Functor, auto Start, auto End>
void apply_functor() {
    using int_type = std::common_type_t<Start, End>;
    apply_functor_helper<Functor>(offset_integer_sequence_t<Start, std::make_integer_sequence<int_type, End - Start>>{});
}
```