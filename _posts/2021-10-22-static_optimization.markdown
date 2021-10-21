---
layout: post
title:  "constexpr 함수가 오직 컴파일 타임에만 호출된다면 이 constexpr 함수는 컴파일 단계에서 완전히 제거되는가??"
date:   2021-10-22
categories: ComputerScience C++
---

```
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

constexpr std::array<const char*, ~~> base_chain = BASE_CHAIN_HILLCLIMB_DATA();
```

위의 어마어마한 템플릿 함수들이 실행파일에 담길 경우 실행 파일 사이즈가 어마어마하게 커지지 않을까 걱정이 되었다.      
필자는 궁금했다. **constexpr 함수가 오직 컴파일 타임에만 호출 ( 런타임에는 호출되지 않는다면 ) 된다면 이 constexpr 함수는 컴파일 단계에서 완전히 제거되는가??**      

위 함수들에 **static이 아닌 extern을 붙이는 경우** 컴파일러는 **이 함수들이 컴파일 중인 소스파일(.cpp) 외의 어딘가에서 사용될 수 있다고 생각하기 때문에 ( external linking ) ( 다른 소스파일이 header를 include해서 사용할 가능성이 존재한다 생각 ) constexpr 함수들의 심볼을 제거하지 않는다.** ( optimization 옵션이 붙은 경우 제거된다. ) ( 물론 링커 단계에서 최적화로 제거를 할 수 있지만 확실한지는 모르겠다. )        

반면 **static을 붙이는 경우** 컴파일러는 이 **함수들의 definition이 현재 컴파일 중인 소스파일에서만 사용된다고 생각**하기 ( internal linking ) 때문에 현재 소스파일에서 런타임에 호출되지 않는다고 확인되면 이 함수들의 **definition을 .obj 파일에서 완전히 제거**해버린다.        

혹시나 인라이닝 때문에 이 함수들이 없어져 보이는 것이 아닌가 걱정되어서 인라이닝을 금지하는 컴파일러 옵션을 준 후에도 확인해보았다. 여전히 해당 함수들은 제거되어 있었다.    