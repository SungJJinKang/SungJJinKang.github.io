---
layout: post
title:  "UE4 CDO 생성 코드 분석. ( UClass::CreateDefaultObject )"
date:   2022-04-10
categories: UE4 UnrealEngine4 ComputerScience
---

중요도가 떨어지는 코드는        
```

...
...
...

```
으로 처리해두었으니 확인하고 싶은 경우에는 UE4 소스코드를 확인해주세요.           

------------------------
          
CDO 생성 관련 코드.         
           
```cpp
UObject* UClass::GetDefaultObject(bool bCreateIfNeeded = true) const
{
    if (ClassDefaultObject == nullptr && bCreateIfNeeded)
    {
		// ⭐ 
		// CDO가 생성되지 않았다면 CDO를 생성한다. 
		// ⭐
        const_cast<UClass*>(this)->CreateDefaultObject(); 
    }

    return ClassDefaultObject;
}

UObject* UClass::CreateDefaultObject()
{
	if ( ClassDefaultObject == NULL )
	{
		ensureMsgf(!HasAnyClassFlags(CLASS_LayoutChanging), TEXT("Class named %s creating its CDO while changing its layout"), *GetName());

		UClass* ParentClass = GetSuperClass();
		UObject* ParentDefaultObject = NULL;
		if ( ParentClass != NULL )
		{
			UObjectForceRegistration(ParentClass);
			// ⭐ 
			// 생성하려는 CDO 오브젝트의 클래스의 부모 클래스의 CDO를 가져온다. 생성되어 있지 않으면 생성한다.
			// ⭐
			ParentDefaultObject = ParentClass->GetDefaultObject(); 
			check(GConfig);
			if (GEventDrivenLoaderEnabled && EVENT_DRIVEN_ASYNC_LOAD_ACTIVE_AT_RUNTIME)
			{ 
				check(ParentDefaultObject && !ParentDefaultObject->HasAnyFlags(RF_NeedLoad));
			}
		}

		if ( (ParentDefaultObject != NULL) || (this == UObject::StaticClass()) )
		{
			// If this is a class that can be regenerated, it is potentially not completely loaded.  Preload and Link here to ensure we properly zero memory and read in properties for the CDO
			if
			( 
				// ⭐ 
				// this 클래스가 BP 기반 클래스인 경우 
				// Indicates that the class was created from blueprint source material
				// ⭐
				HasAnyClassFlags(CLASS_CompiledFromBlueprint ) &&

				// ⭐ 
				// 이 클래스의 프로퍼티 리스트가 비어 있는 경우 
				// ⭐
				(PropertyLink /** In memory only: Linked list of properties from most-derived to base */  == NULL) &&            

				// Indicates that we're currently processing the first frame of intra-frame debugging
				!(GIsDuplicatingClassForReinstancing)
			)
			{
				// ⭐ 
				// 프로퍼티들을 초기화(0으로 초기화, UPROPERTY 옵션이 붙은 프로터피들은 기본적으로 0으로 초기화된 후 읽어온다.)하고, 읽어온다.
				// ⭐        
				auto ClassLinker = GetLinker();
				if (ClassLinker && !ClassLinker->bDynamicClassLinker)
				{
					if (!GEventDrivenLoaderEnabled)
					{
						UField* FieldIt = ( Children /** Pointer to start of linked list of child fields */ ) ;
						while (FieldIt && (FieldIt->GetOuter() == this))
						{
							// ⭐
							// 디스크의 언리얼 package 파일에서 this 클래스의 필드들의 데이터를 일일이 읽어온다.
							// 여기서는 아직 CDO가 생성되어 있지 않은 상태이다. 
							// CDO 오브젝트의 프로퍼티들에 초기화될 데이터를 디스크에서 읽어와서 UClass가 가지고 있는 프로퍼티 리스트에 저장하는 과정이다.
							// ⭐

							// If we've had cyclic dependencies between classes here, we might need to preload to ensure that we load the rest of the property chain
							if (FieldIt->HasAnyFlags(RF_NeedLoad))
							{
								ClassLinker->Preload(FieldIt); /*  */
								/**
								 * Serialize the object data for the specified object from the unreal package file.  Loads any
								 * additional resources required for the object to be in a valid state to receive the loaded
								 * data, such as the object's Outer, Class, or ObjectArchetype.
								 *
								 * When this function exits, Object is guaranteed to contain the data stored that was stored on disk.
								 *
								 * @param	Object	The object to load data for.  If the data for this object isn't stored in this
								 *					FLinkerLoad, routes the call to the appropriate linker.  Data serialization is 
								 *					skipped if the object has already been loaded (as indicated by the RF_NeedLoad flag
								 *					not set for the object), so safe to call on objects that have already been loaded.
								 *					Note that this function assumes that Object has already been initialized against
								 *					its template object.
								 *					If Object is a UClass and the class default object has already been created, calls
								 *					Preload for the class default object as well.
								 */
								 // void FLinkerLoad::Preload( UObject* Object )
							}
							FieldIt = FieldIt->Next;
						}
					}
					
					StaticLink(true); /** Static wrapper for Link, using a dummy archive */
				}
			}
			
			// ⭐
			// 만약 위의 코드에서 클래스의 프로퍼티들을 계속 로드하다 this 클래스가 this 클래스 타입의 프로퍼티를 참조 중이라면 ( 순환 참조 ), 위에서 Preload 중에 this 클래스에 대한 UClass::CreateDefaultObject 함수를 재귀적으로 호출하게 될 것이다. 
			// 그럼 그 재귀적으로 호출된 이 UClass::CreateDefaultObject 함수에서 ClassDefaultObject에 대한 생성, 초기화 과정을 맞쳤을 것이다.
			// 그럼 그 재귀적으로 호출된 함수가 종료되고 원래의 이 함수로 돌아 왔을 때는 this 클래스의 ClassDefaultObject에는 유효한 CDO가 들어 있을 것이다. ( 재귀적으로 호출된 이 함수에서 이미 셋팅을 했으니 말이다. )
			// 그럼 원래의 이 함수 호출에서 ClassDefaultObject를 다시 셋팅해줄 필요는 당연히 없다...
			// 그러니 맨 위에서 ClassDefaultObject가 NULL인지 한번 체크를 했더라고 여기서 한번 더하는 것이다. ( 순환 참조로 ClassDefaultObject가 이미 셋팅되어 있을 수 있으니 말이다.... )      
			// 여기서도 ClassDefaultObject이 NULL이라면 this 클래스의 프로퍼티 중 자기 자신에 대한 순환 참조 관계가 없다는 것을 의미한다.          
			// ⭐

			if (ClassDefaultObject == NULL)
			{
				FString PackageName;
				FString CDOName;
				bool bDoNotify = false;
				if (GIsInitialLoad && GetOutermost()->HasAnyPackageFlags(PKG_CompiledIn) && !GetOutermost()->HasAnyPackageFlags(PKG_RuntimeGenerated))
				{
					PackageName = GetOutermost()->GetFName().ToString();
					CDOName = GetDefaultObjectName().ToString();
					NotifyRegistrationEvent(*PackageName, *CDOName, ENotifyRegistrationType::NRT_ClassCDO, ENotifyRegistrationPhase::NRP_Started);
					bDoNotify = true;
				}

				// ⭐
				// RF_ArchetypeObject flag is often redundant to RF_ClassDefaultObject, but we need to tag
				// the CDO as RF_ArchetypeObject in order to propagate that flag to any default sub objects.
				// CDO를 생성한다. RF_ClassDefaultObject flag를 전달하는 것을 알 수 있다.
				// CDO를 위한 메모리를 할당하고 기본적인 셋팅들을 해준다. ( 여기서 CDO의 프로퍼티들이 로드되는 것은 아니다. )
				// CDO를 위한 메모리를 할당해주고, 할당된 공간에 대해 0으로 초기화를 수행하고, 여러 Flag 셋팅, 모든 UObject의 조상격인 UObjectBase에 대한 생성자 호출 등등의 작업을 이 StaticAllocateObject 함수에서 수행합니다.
				// 자세한건 아래를 참고하세요.
				// ⭐

				ClassDefaultObject = StaticAllocateObject(this, GetOuter(), NAME_None, EObjectFlags(RF_Public|RF_ClassDefaultObject|RF_ArchetypeObject))
				check(ClassDefaultObject);
				// Blueprint CDOs have their properties always initialized.
				const bool bShouldInitializeProperties = !HasAnyClassFlags(CLASS_Native | CLASS_Intrinsic);
				// Register the offsets of any sparse delegates this class introduces with the sparse delegate storage
				for (TFieldIterator<FMulticastSparseDelegateProperty> SparseDelegateIt(this, EFieldIteratorFlags::ExcludeSuper, EFieldIteratorFlags::ExcludeDeprecated); SparseDelegateIt; ++SparseDelegateIt)
				{
					const FSparseDelegate& SparseDelegate = SparseDelegateIt->GetPropertyValue_InContainer(ClassDefaultObject);
					USparseDelegateFunction* SparseDelegateFunction = CastChecked<USparseDelegateFunction>(SparseDelegateIt->SignatureFunction);
					FSparseDelegateStorage::RegisterDelegateOffset(ClassDefaultObject, SparseDelegateFunction->DelegateName, (size_t)&SparseDelegate - (size_t)ClassDefaultObject);
				}
				if (HasAnyClassFlags(CLASS_CompiledFromBlueprint))
				{
					if (UDynamicClass* DynamicClass = Cast<UDynamicClass>(this))
					{
						(*(DynamicClass->DynamicClassInitializer))(DynamicClass);
					}
				}

				// Internal class to finalize UObject creation (initialize properties) after the real C++ constructor is called. 
				// Sets the class to use for a subobject defined in a base class, the class must be a subclass of the class used by the base class.

				// ⭐
				// 이제 진짜로 CDO의 생성자를 호출한다. !!
				// 흔히 ADefaultPawn::ADefaultPawn(const FObjectInitializer& ObjectInitializer) 이러한 생성자를 본 적이 있을 것이다.
				// 이 생성자를 여기서 호출하는 것이다!!
				// 이 생성자에서 여러분이 정의한 프로퍼티들의 초기화나 SubObject 생성, 에셋 로드 등등의 코드들이 모두 여기 들어갑니다.
				// 여러분이 쓴 생성자 코드가 여기서 호출됩니다!!
				// 생성자의 매개변수로는 FObjectInitializer가 들어간다.
				// 이와 함께 BaseClass에서 정의 SubObject들에 대해서도 초기화를 한다.
				// ⭐
				(*ClassConstructor)(FObjectInitializer(ClassDefaultObject /* CDO */, ParentDefaultObject, false, bShouldInitializeProperties)); 
				
				if (bDoNotify)
				{
					NotifyRegistrationEvent(*PackageName, *CDOName, ENotifyRegistrationType::NRT_ClassCDO, ENotifyRegistrationPhase::NRP_Finished);
				}
				// ⭐ 
				// CDO 생성 후 호출되는 경우. 
				// 일반적으로는 아무 동작을 하지 않지만 일부 클래스들 ( ex. UMaterialInterface, UTexture .... )은 상속받아 필요한 동작을 수행합니다. 
				// ⭐
				ClassDefaultObject->PostCDOContruct(); 
			}
		}
	}
	return ClassDefaultObject;
}

UObject* StaticAllocateObject
(
	const UClass*	InClass,
	UObject*		InOuter,
	FName			InName,
	EObjectFlags	InFlags,
	EInternalObjectFlags InternalSetFlags,
	bool bCanRecycleSubobjects,
	bool* bOutRecycledSubobject,
	UPackage* ExternalPackage
)
{
	...
	...
	...

	// ⭐ 
	// 위의 UClass::CreateDefaultObject 함수에서 호출되었으니 true를 가짐. 
	// ⭐
	bool bCreatingCDO = (InFlags & RF_ClassDefaultObject) != 0; 

	...
	...
	...

	UObject* Obj = NULL;
	if(InName == NAME_None)
	{
		...
		...
		...
	}
	else
	{
		// See if object already exists. 
		// ⭐ 
		// 혹시 CDO가 이미 메모리에 로드되어 있는지 확인한다. 
		// ⭐
		Obj = StaticFindObjectFastInternal( /*Class=*/ NULL, InOuter, InName, true );

		// Temporary: If the object we found is of a different class, allow the object to be allocated.
		// This breaks new UObject assumptions and these need to be fixed.
		if (Obj && !Obj->GetClass()->IsChildOf(InClass))
		{
			// ⭐ 
			// 각종 예외 처리 코드 
			// ⭐
			...
			...
			...
		}
	}

	FLinkerLoad*	Linker						= NULL;
	int32			LinkerIndex					= INDEX_NONE;
	bool			bWasConstructedOnOldObject	= false;
	// True when the object to be allocated already exists and is a subobject.
	bool bSubObject = false;
	int32 TotalSize = InClass->GetPropertiesSize();
	checkSlow(TotalSize);

	// ⭐ 
	// 이 글에서는 일단 CDO가 이미 생성되어 있지 않은 경우를 가정하고 분석하겠다. \
	// ⭐  
	if( Obj == NULL )
	{	
		// ⭐ 
		// CDO의 메모리를 할당한다. 
		// 메모리 Alignment를 준수하여 CDO 사이즈 만큼의 메모리를 할당한다. 
		// ⭐
		int32 Alignment	= FMath::Max( 4, InClass->GetMinAlignment() );
		Obj = (UObject *)GUObjectAllocator.AllocateUObject(TotalSize,Alignment,GIsInitialLoad);
	}
	else
	{
		...
		...
		...
	}

	...
	...
	...

	if (!bSubObject)
	{
		// ⭐ 
		// 위에서 할당한 메모리 공간을 0으로 초기화한다. 
		// ⭐
		FMemory::Memzero((void *)Obj, TotalSize);

		// ⭐ 
		// 그 후 생성자를 호출한다. 
		// 특별한 동작이 있는 것은 아니고 CDO 오브젝트의 기본적인 셋팅들을 한다. 
		// 그리고 UObject Array에 이 CDO를 추가한다. 
		// 아래 참고. 
		// ⭐ 
		new ((void *)Obj) UObjectBase(const_cast<UClass*>(InClass), InFlags|RF_NeedInitialization, InternalSetFlags, InOuter, InName);
	}
	else
	{
		...
		...
		...
	}

	...
	...
	...

	return Obj;
}


/**
 * Constructor used by StaticAllocateObject
 * @param	InClass				non NULL, this gives the class of the new object, if known at this time
 * @param	InFlags				RF_Flags to assign
 * @param	InOuter				outer for this object
 * @param	InName				name of the new object
 * @param	InObjectArchetype	archetype to assign
 */
UObjectBase::UObjectBase(UClass* InClass, EObjectFlags InFlags, EInternalObjectFlags InInternalFlags, UObject *InOuter, FName InName)
:	ObjectFlags			(InFlags)
,	InternalIndex		(INDEX_NONE)
,	ClassPrivate		(InClass)
,	OuterPrivate		(InOuter)
{
	check(ClassPrivate);
	// Add to global table.
	// ⭐ 
	// 오브젝트의 이름을 FName 테이블에 추가하고, UObject들을 모두 모아둔 Array에 이 오브젝트를 추가한다. 
	// ⭐
	AddObject(InName, InInternalFlags); 
}

```