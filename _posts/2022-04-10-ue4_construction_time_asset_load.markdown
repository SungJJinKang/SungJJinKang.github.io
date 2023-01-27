---
layout: post
title:  "언리얼 엔진4 Construction Time Asset Load ( ConstructorHelpers )"
date:   2022-04-10
tags: [UE]
---


중요도가 떨어지는 코드는        
```

...
...
...

```
으로 처리해두었으니 확인하고 싶은 경우에는 UE4 소스코드를 확인해주세요.           

------------------------              

아래 코드는 UE4 ADefaultPawn 클래스의 생성자 코드이다.     

```cpp
ADefaultPawn::ADefaultPawn(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{
	...

	// Structure to hold one-time initialization
	struct FConstructorStatics
	{
		ConstructorHelpers::FObjectFinder<UStaticMesh> SphereMesh;
		FConstructorStatics()
			: SphereMesh(TEXT("/Engine/EngineMeshes/Sphere")) {}
	};
	
	// ⭐ 
	// CDO의 생성자에서 딱 한번 이 클래스가 Initialized 된다. 
	// ⭐
	static FConstructorStatics ConstructorStatics; 

    ...
    ...
    ...
}
```

주목할 것은 FConstructorStatics인데 이는 해당 클래스의 인스턴스들에서 사용할 에셋을 로드하는 작업을 매번 수행하지 않고, CDO(Class Default Object) 오브젝트의 생성자에서 딱 한번만 수행하기 위함이다. ( 함수 내의 전역 변수의 Initialization은 해당 코드에 닿는 처음 한번만 수행된다. )               
                  
UE4에서는 어떤 클래스에 대한 인스턴스를 생성할 때 미리 생성된 CDO에서 프로퍼티 값들을 가져온다.     
그러니 당연히 어떤 클래스에 대한 인스턴스들을 생성하기 전 해당 클래스의 CDO(Class Default Object)가 미리 생성되어 있어야한다. 그러니 위 ConstructorStatics 전역 변수는 CDO 인스턴스 생성자에서 딱 한번 Initialize 된다.                

C++에서의 전역 변수의 Initialization 속성을 이용한 코드라고 볼 수 있다.      

FConstructorStatics라는 구조체를 임시로 만들어서 전역 변수를 사용한 것은 UE4 엔진 코드단에서 흔히 사용하는 일종의 Idiom이라고 생각하면 된다.       

아래 처럼 코드를 작성해도 무관하다.           

```cpp
static ConstructorHelpers::FObjectFinder<UStaticMesh> SphereMesh { TEXT("/Engine/EngineMeshes/Sphere") };
```


잠시 ConstructorHelpers 클래스 코드를 살펴보겠다.        

```cpp
struct COREUOBJECT_API ConstructorHelpers
{
public:
	template<class T>
	struct FObjectFinder : public FGCObject 
    // ⭐ 
	// FObjectFinder 구조체는 UObject의 리플랙션 시스템의 지원을 받지 못하다 보니 로드한 오브젝트가 GC에 의해 회수될 수 있다. 
	// 그렇기 때문에 FGCObject 상속받아서 아래 AddReferencedObjects 함수에서 로드한 오브젝트를 GC 레퍼런스 오브젝트 목록에 추가해준다. 
	// ( 참고 자료 : https://ikrima.dev/ue4guide/engine-programming/memory/tracking-references/ ) 
	// ⭐
	</pre>
	{
		T* Object; // !!
		FObjectFinder(const TCHAR* ObjectToFind, uint32 InLoadFlags = LOAD_None)
		{
			CheckIfIsInConstructor(ObjectToFind); // ⭐ ConstructorHelpers의 Initialization은 반드시 어떤 클래스의 생성자에서 호출되어야한다. ⭐          
			FString PathName(ObjectToFind);
			StripObjectClass(PathName,true);

			// ⭐ 
			// 아래 참조 
			// ⭐
			Object = ConstructorHelpersInternal::FindOrLoadObject<T>(PathName, InLoadFlags); 
			ValidateObject( Object, PathName, ObjectToFind ); 
		}

		...
		...
		...

		virtual void AddReferencedObjects( FReferenceCollector& Collector ) override
		{
			// ⭐ 
			// GC가 로드한 오브젝트를 회수하지 못하도록 레퍼런스된 오브젝트 목록에 로드한 오브젝트를 추가한다. 
			// ⭐
			Collector.AddReferencedObject(Object); 
		}

		...
		...
		...

	};

    ...
    ...
    ...

 	// ⭐ 
	// FObjectFinder의 클래스 버전 ( TSubClassOf ) 
	// ⭐
	template<class T>
	struct FClassFinder : public FGCObject
	{
		TSubclassOf<T> Class; // !!
		FClassFinder(const TCHAR* ClassToFind)
		{
			CheckIfIsInConstructor(ClassToFind);
			FString PathName(ClassToFind);
			StripObjectClass(PathName, true);
			Class = ConstructorHelpersInternal::FindOrLoadClass(PathName, T::StaticClass());
			ValidateObject(*Class, PathName, *PathName);
		}
		
		...
		...
		...

		virtual void AddReferencedObjects( FReferenceCollector& Collector ) override
		{
			UClass* ReferencedClass = Class.Get();
			Collector.AddReferencedObject(ReferencedClass);
			Class = ReferencedClass;
		}

		...
		...
		...

	};

    ...
    ...
    ...
};

namespace ConstructorHelpersInternal
{
	template<typename T>
	inline T* FindOrLoadObject( FString& PathName, uint32 LoadFlags ) // ⭐ 오브젝트를 로드하는 함수.⭐ 
	{
		... ( 올바른 경로 찾기 코드들... )
		...
		...

		UClass* Class = T::StaticClass();

		// ⭐ 
		// 오브젝트를 로드하기 전 해당 클래스의 CDO가 생성되었는지 확인하고 생성되어 있지 않다면 CDO를 생성한다. 
		// CDO가 생성되어 있어야지만, 
		// CDO로부터 필요한 프로퍼티들에 대한 데이터를 복사해 올 수 있다. 
		// 아래 참고. 
		// ⭐
		Class->GetDefaultObject(); 
		T* ObjectPtr = LoadObject<T>(NULL, *PathName, nullptr, LoadFlags);
		if (ObjectPtr)
		{
			ObjectPtr->AddToRoot();
		}
		return ObjectPtr;
	}
}
```

CDO를 생성하는 코드는 [이 글](https://sungjjinkang.github.io/ue4_cdo_construction_load)을 참고하라.