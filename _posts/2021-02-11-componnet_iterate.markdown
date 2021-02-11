---
layout: post
title:  "매 프레임마다 iterate를 하는 컴포넌트를 빠르게 iterate하는 방법"
date:   2021-02-11
categories: Doom3
---

게임을 한번이라도 만들어 본 사람이라면 게임 내에는 매 프레임마다 loop함수를 실행하고 그 함수에는 Rendering이나 Physics 등등의 작업을 매 프레임 처리한다는 것을 알 것이다.   
특히 Rendering을 처리한다고 하면 모든 Entity에 붙어 있는 Rendering 컴포넌트를 모두 iterate 해야하는 데 잘못할 경우 이 iterating이 큰 오버헤드를 발생시킬 수 있다.   

단순하게 모든 Entity에 붙어 있는 Rendering 컴포넌트를 iterate 한다고 생각해보자.    
그럼 이렇게 할 수 있다.    

```c++
for(auto& entity : GetAllSpawnedEntity())
{
	RendererComponent* renderer = entity.GetComponent<RendererComponent>();
    if(renderer != nullptr)
    {
        renderer->Update();
    }
}
```	  

만약 이런식으로 Iterate를 돈다면 매우 매우 느릴 것이다. 
내가 고안한 방법으로 하면 그냥 vector를 iterate 하듯이 매우 빠르게 모든 Entity의 RenderComponent의 Update 함수를 호출할 수 있다.   

```c++   
auto iter_begin_end = ComponentStaticIterater<RendererComponent>::GetIter();
for(auto iter = iter_begin_end->begin() ; iter != iter_begin_end->end() ; ++iter)
{
    (*iter)->Update();
}

```	   

구현 방법은 아래와 같다.   
static 변수에 생성된 인스턴스들을 캐싱해두고 그 변수에 직접 접근하는 것이다.   

```c++   
template <typename T>
class ComponentStaticIterater //Never inherit Component
{
	using this_type = typename ComponentStaticIterater<T>;
	using container_type = typename std::vector<T*>;

private:

    inline container_type mComponents{};

    constexpr virtual void AddComponentToStaticContainer()
    {
        this_type::mComponents.push_back(reinterpret_cast<T*>(this));
    }

    virtual void RemoveComponentToStaticContainer()
    {
        auto iter = std::find(this_type::mComponents.begin(), this_type::mComponents.end(), reinterpret_cast<T*>(this));

        D_ASSERT(iter != this_type::mComponents.end());
        std::vector_swap_erase(this_type::mComponents, iter);
    }

protected:

    constexpr ComponentStaticIterater()
    {
        this->AddComponentToStaticContainer();
    }

    ~ComponentStaticIterater()
    {
        this->RemoveComponentToStaticContainer();
    }

public:

	[[nodiscard]] static constexpr std::pair<typename container_type::iterator, typename container_type::iterator> GetIter()
	{
			return std::make_pair(this_type::mComponents.begin(), this_type::mComponents.end());
	}
};
```

이 방식이 왜 빠를까?    
1. 우선 보다시피 GetComponent 함수를 호출하지 않기 떄문에 그 만큼 function call이 줄어든다.    
2. 컴포넌트들이 random access 컨테이너인 벡터에 저장되어 있기 떄문에 훨씬 접근이 빠르다. 그리고 iterator을 돌면서 cache hit가 날 가능성이 높기때문에 그로 인한 성능 향상도 기대할 수 있다.    