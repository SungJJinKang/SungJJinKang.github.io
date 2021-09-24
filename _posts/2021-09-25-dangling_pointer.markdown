---
layout: post
title:  "게임 엔진에서 메모리 누수를 줄이기 위한 전략 ( 게임 엔진의 오브젝트 관리 )"
date:   2021-09-25
categories: ComputerScience GameEngine
---

new 할당한 오브젝트의 포인터를 잃어버리게 되면 영원히 해재할 수 없게된다.      
각 모듈, 클래스에서 new 할당한 오브젝트를 직접 관리하게 되면 dangling 포인터를 만들 수 있고 오브젝트를 파괴하는 것을 까먹을 수도 있기 때문에 게임 엔진의 메모리 관리 측면에서 매우 좋지 않다.           

이를 막기 위해서는 **게임 엔진 내에서 사용하는 여러 오브젝트들을 관리하는 어떤 매니저 주체가 필요하다.**               
**오브젝트가 생성되면 생성자 단계에서 해당 오브젝터의 포인터를 별도로 한 곳에 모아서 관리하면 적어도 생성한 오브젝트를 놓치는 일은 발생하지 않는다.**                    
오브젝트가 파괴될 때는 매니저에서도 삭제하면 된다.      

이건 사실 인턴을 하면서 실무자분들이 실무에서 이렇게 오브젝트를 관리한다고 알려주신 방법이다. ( 내가 있던 팀이 MMORPG 팀이었기 때문에 이러한 오브젝트 관리가 더 중요했을 것 같다. )                    
사실 이전에는 그냥 내가 실수하지 않으면 되지하고 안일하게 생각했는데 이 방법을 알고 나니 훨씬 부담을 덜었다.     
그리고 해쉬테이블로 오브젝트 포인터들을 관리하면 오브젝트ID를 가지고 원하는 오브젝트를 빠르게 찾을 수도 있다.       
또한 씬이 바뀌거나 오브젝트들을 한꺼번에 삭제해버리고 싶으면 모아두었던 오브젝트 포인터들을 순회하면서 파괴해주면 된다.      
매우 간편하다.          

실제로 언리얼 엔진을 보면 엔진내의 거의 모든 클래스는 최상단에 UObject 클래스를 조상 클래스로 두고 있다.          

다만 이 방법도 현재 문제가 있어 보인다.       
일단 new 할당을 하지 않은 오브젝트를 delete로 파괴해버릴 가능성이 존재한다.         

```
class A
{
    B b;
};
A* a = new A();
```

위의 경우 오브젝트 a와 b 모두 매니저에서 관리되고 있는데 만약 a를 파괴하기 전 b를 파괴하면 오브젝트 b는 new로 할당된 오브젝트가 아니기 때문에 delete로 파괴하려고 하면 exception이 발생한다.    
내 생각에는 이를 막기 위해서는 new 할당한 오브젝트와 그렇지 않은 오브젝트를 구분해주어야한다.        
가장 쉬운 방법은 new를 사용하는 대신 new 할당을 wrapping한 함수를 따로 만들어주면 될 것 같다.         
Variadic Template을 사용하면 생성자의 매개 변수 전달도 쉽게 구현할 수 있다.        
이 new 할당을 wrapping한 함수를 사용할 때는 오브젝트 생성 후 어떤 Flag를 셋팅해주어서 new 할당한 오브젝트와 그렇지 않은 오브젝트를 구분해주면 될 것 같다.           

실제 언리얼에서도 UObject는 new 할당을 금지하고 NewObject라는 함수를 사용한다. ( 사실 위의 이유 때문인지는 모른다. )        

아래는 내가 작성한 코드이다.        

[소스코드](https://github.com/SungJJinKang/ModernDoom2/tree/main/Doom3/Source/Core/DObject)         

```
class DObjectManager
{	
    friend class DObject;

private:

    inline static size_t mDObjectCounter = 0;
    static std::unordered_map<size_t, DObject*> mDObjectsHashMap;

    static size_t GenerateNewDObejctID();
    static bool AddDObject(DObject* const dObject);
    static bool ReplaceDObject(DObject& originalDObject, DObject* const newDObject);
    static bool RemoveDObject(DObject* const dObject);

public:

    static DObject* GetDObject(const size_t dObjectID);
    static void DestroyAllDObjects();
};

size_t doom::DObjectManager::GenerateNewDObejctID()
{
    mDObjectCounter++;
    return mDObjectCounter;
}

bool doom::DObjectManager::AddDObject(DObject* const dObject)
{
    bool isSuccess = false;

    D_ASSERT(dObject->GetDObjectID() != INVALID_DOBJECT_ID);

    auto dObjectIter = mDObjectsHashMap.find(dObject->GetDObjectID());
    if (dObjectIter == mDObjectsHashMap.end())
    {
        mDObjectsHashMap.emplace(dObject->GetDObjectID(), dObject);
        isSuccess = true;
    }

    D_ASSERT(isSuccess == true);

    return isSuccess;
}

bool doom::DObjectManager::ReplaceDObject(DObject& originalDObject, DObject* const newDObject)
{
    bool isSuccess = false;

    D_ASSERT(originalDObject.GetDObjectID() != INVALID_DOBJECT_ID);

    auto dObjectIter = mDObjectsHashMap.find(originalDObject.GetDObjectID());
    if (dObjectIter != mDObjectsHashMap.end())
    {
        dObjectIter->second = newDObject;
        newDObject->mDObjectID = originalDObject.GetDObjectID();
        originalDObject.mDObjectID = INVALID_DOBJECT_ID;

        isSuccess = true;
    }

    D_ASSERT(isSuccess == true);

    return isSuccess;
}

bool doom::DObjectManager::RemoveDObject(DObject* const dObject)
{
    bool isSuccess = false;

    D_ASSERT(dObject->GetDObjectID() != INVALID_DOBJECT_ID);

    auto dObjectIter = mDObjectsHashMap.find(dObject->GetDObjectID());
    if (dObjectIter != mDObjectsHashMap.end())
    {
        dObjectIter->second = nullptr;
        isSuccess = true;
    }

    D_ASSERT(isSuccess == true);

    return isSuccess;
}

doom::DObject* doom::DObjectManager::GetDObject(const size_t dObjectID)
{
    doom::DObject* dObject = nullptr;
    auto dObjectIter = mDObjectsHashMap.find(dObjectID);
    if (dObjectIter != mDObjectsHashMap.end())
    {
        dObject = dObjectIter->second;
    }
    return dObject;
}

void doom::DObjectManager::DestroyAllDObjects()
{
	for (auto dObject : mDObjectsHashMap)
	{
		if (dObject.second != nullptr)
		{
			delete dObject.second;
		}
	}
}
```

.            

```
class DObject
{
    friend class DObjectManager;

private:

    size_t mDObjectID;

    void InitializeDObject();

protected:

    DObject();



    DObject(const DObject& dObject);
    DObject(DObject&& dObject) noexcept;
    DObject& operator=(const DObject& dObject);
    DObject& operator=(DObject&& dObject) noexcept;

    virtual ~DObject();

public:

    inline size_t GetDObjectID() const
    {
        return mDObjectID;
    }
};

doom::DObject::DObject()
	:mDObjectID(INVALID_DOBJECT_ID)
{
	InitializeDObject();
}

void doom::DObject::InitializeDObject()
{
	mDObjectID = DObjectManager::GenerateNewDObejctID();
	DObjectManager::AddDObject(this);
}

doom::DObject::DObject(const DObject& dObject)
{
	InitializeDObject();
}

doom::DObject::DObject(DObject&& dObject) noexcept
{
	DObjectManager::ReplaceDObject(dObject, this);
}

doom::DObject& doom::DObject::operator=(const DObject& dObject)
{
	InitializeDObject();
	return *this;
}

doom::DObject& doom::DObject::operator=(DObject&& dObject) noexcept
{
	DObjectManager::ReplaceDObject(dObject, this);
	return *this;
}

doom::DObject::~DObject()
{
	if (mDObjectID != INVALID_DOBJECT_ID)
	{
		DObjectManager::RemoveDObject(this);
	}
}
```

.          

```
template <typename DObjectType, typename... Args>
DObjectType* CreateDObject(Args... args)
{
    static_assert(std::is_base_of_v<DObject, DObjectType> == true);

    DObjectType* const newDObject = new DObjectType(std::forward<Args>(args)...);
    newDObject->mDObjectID = doom::DObject::eDObjectFlag::NewAllocated;

    return newDObject;
}
```