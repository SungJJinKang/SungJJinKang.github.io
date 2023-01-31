---
layout: post
title:  "reflection 시스템을 이용한 C++ 가비지 컬렉터"
date:   2023-12-01
tags: [ComputerScience, InHouseEngine, C++]
---

[지난번 만든 reflection 시스템](https://sungjjinkang.github.io/reflection)을 이용하여 C++ 가비지 컬렉터를 만들 것이다.             

이전에도 [메모리 누수를 방지하기 위한 구조](https://sungjjinkang.github.io/dangling_pointer)가 존재했지만 이 방법은 씬이 끝나거나 특수한 상황에서 임의로 모든 DObject들을 한꺼번에 회수해주는 기능이라 엄연히 참조가 되지 않은 오브젝트를 주기적으로 회수해주는 가비지 컬렉터와는 역할이 다르다.               

**언리얼 엔진의 가비지 컬렉터**를 생각하면 된다.       
가비지 컬렉터는 루트 오브젝트부터 프로퍼티들을 순회하면서 ( 리플렉션 데이터 이용 ) 최종적으로 **참조되지 않은 오브젝트를 회수**한다.          
또한 **회수된 오브젝트에 대한 주소 or 유효하지 않은 주소를 가지고 있는 포인터에는 null 값을 넣어준다.** 이를 통해서 매우 값싼 비용으로 게임 엔진의 안전성을 높여준다. ( **유효하지 않은 주소를 ( ex) null ) 참조하거나, 이미 해제한 오브젝트를 다시 한번 해제하는 것을 막아준다.** )                         

이를 위해서 게임 내 거의 모든 오브젝트는 DObject라는 하나의 클래스로부터 상속받아 뻗어나간다. 또한 리플랙션을 위한 매크로도 붙여주어야한다.              
그리고 DObject의 생성자 단계에서 해당 오브젝트의 주소를 따로 관리하는 해시테이블쪽에 저장을 해두면 Sweep 단계에서 이 리스트를 순회하면서 최종적으로 참조되지 않은 파괴할 오브젝트들을 알 수 있고 또한 가비지컬렉트시 어떤 포인터가 가지고 있는 주소가 유효한지 ( 할당된 DObject인지 )를 알 수 있어져 해당 포인터에 null 값을 셋팅해줄 수 있다.                    

필자는 이러한 가비지 컬렉터가 **개발 효율성의 측면에서 매우 도움이 된다**고 생각한다. C++에서 메모리 관리는 고생스러운 일이고 필자 또한 기존에는 스마트 포인터를 사용해서 관리해왔다. 이마저도 불편하다.       
가비지 컬렉터를 위해 몇가지 제약들이 생겼지만 이는 가비지컬렉터로부터 오는 개발 편의성과 비교해보면 납득할만한 제약이라고 생각한다.           
스마트포인터도 필요없다. 그냥 파괴할 오브젝트에 대해서는 reference를 끊어주면된다. 얼마나 편한가.               



가비지 컬렉터를 위한 보조 기능들은 다 구현이 되어 있어서 금방 만들 것 같다.             

------------------------------              

**동작 원리**           

- Unreachability 플래그 셋팅 단계 ( 1 단계 )                 
**프로그램 내의 모든 DObject ( 게임 내 거의 모든 클래스들의 조상 클래스 [참고](https://sungjjinkang.github.io/dangling_pointer) )들의 Unreachability 플래그를 1로 셋팅**한다. 이 플래그도 캐시 극대화를 위해 연속되게 배치하였다. ( SOA!! )                       


- Mark 단계 ( 2 단계 )              
**루트 오브젝트를 순회하고 그 오브젝트의 프로퍼티들을 순회**한다. ( 이전에 제작한 **리플랙션 시스템을 이용**한다. )        
이를 재귀적으로 하다보면 리플랙션 시스템이 관리하는 모든 오브젝트들을 순회할 수 있다. **순회를 하면서 들린 오브젝트들은 위에서 1로 셋팅한 Unreachability 플래그를 0으로 셋팅**해준다.           
이때 프로퍼티 중 **Raw 포인터 프로퍼티를 만나면 해당 포인터가 가지고 있는 주소가 유효한 주소인지를 확인**한다. 포인터가 가지고 있는 DObject가 **IsPendingKill** 플래그를 가지고 있거나 해당 주소가 게임 엔진에서 관리하는 DObject 리스트에 없다면 해당 **포인터가 가지고 있는 주소가 유효하지 않다고 판단하고 포인터에 nullptr 값을 넣어준다.** 이는 **이미 파괴된 오브젝트의 주소를 참조해서 사용하는 것을 방지하기 위해**서 오브젝트가 유효하지 않으면 바로 바로 nullptr을 셋팅해서 버그를 방지하는 것이다. 이 또한 언리얼 엔진에서 차용한 기능이다.     
여기서 IsPendingKill 플래그라는 것은 기본적으로 **게임 내의 오브젝트들은 해당 오브젝트를 파괴하는 함수를 호출 해도 즉시 메모리를 회수하지 않는다.** ( delete 대신 필자가 정의한 함수를 사용한다. ) 오브젝트를 **파괴하는 함수를 호출하면 일단은 IsPendingKill 플래그만 셋팅**을 해둔다.              
이는 이미 **파괴한 오브젝트를 참조하는 것을 방지**하기 위함이다. 그러니깐 만약 어떤 포인터가 들고 있는 오브젝트를 파괴했다고 가정해보자. 그럼 해당 포인터는 파괴된 오브젝트의 주소를 가지고 있을 것이다. 그리고 이를 참조하면 예외가 발생한다. 위에서 말했듯이 가비지 컬렉터가 유효하지 않은 주소를 가진 포인터에 null 값을 넣어주지만 **가비지 컬렉터는 주기적으로 호출되는 것이지 매 프레임 호출되지 않는다.** 그러니 그 사이 해당 유효하지 않은 주소를 참조해버릴 수 있다. 그렇기 때문에 일단은 해당 오브젝트의 IsPendingKill 플래그만 셋팅해두는 것이다. 그럼 프로그래머는 어떤 포인터가 가진 주소를 호출하는 것이 안전한지 안한지를 판단하려면 해당 주소가 NULL 값인지 아닌지, 아니라면 해당 주소의 오브젝트를 참조해 IsPendingKill 플래그가 0인지만 확인하면 된다. ( 이 플래그를 확인하는 연산도 매우 매우 싸다, 빠르다. )            
중요한 것이 **일단은 해당 주소가 NULL 값인지 아닌지만 판단하면 해당 주소의 참조 자체는 안전하다.** ( 물론 파괴 대기 중이기 때문에 의도와 다르게 동작할 수는 있지만 그것이 exception을 발생시키지는 않는다. ) 위에서 말했듯이 이 가비지 컬렉트 시스템이 있으면 D_PROPERTY ( 이 변수를 리플랙션 한다는 의미 )가 붙은 모든 포인터는 NULL 값을 가지거나 파괴되지 않은 오브젝트 주소를 가지고 있다. 둘 중 하나가 항상 보장된다.      
그럼 프로그래머는 간단히 해당 주소가 NULL 값인지, 아니라면 IsPendingKill 플래그를 가졌는지만 확인하는 매우 값싼, 빠른 코드만 포인터 참조 전 체크해주면 매우 안전하게 Raw 포인터를 다룰 수 있는 것이다.           
( 프로그래머가 고장 내려는 마음을 먹고 의도적으로 포인터에 Dummy 값을 넣어버리고 가비지 컬렉터가 돌기전에 참조해서 사용해버리면 exception이 뜨나, 이렇게 의도적으로 코드를 짜는 것도 쉽지 않다. )       
또한 std::vector<T*>나 std::array<T*>도 지원되게 구현을 해두었기 때문에 매우 편리하게 개발을 할 수 있다.         
솔직히 말하면 그냥 동적 할당한 오브젝트를 파괴안하고 그냥 해당 오브젝트를 가진 주소에 null 값만 셋팅해줘도 된다.         
귀찮은 일은 모두 가비지 컬렉터가 대신 해주기 때문이다.         

- Sweep 단계 ( Mark 단계와 연달아서 동작 ) ( 3 단계 )       
**Mark 단계가 끝나고도 여전히 Unreachability 플래그가 1인 오브젝트들은 해당 오브젝트에 대한 reference가 없다고 판단을 하고 파괴**시킨다. ( 또한 IsPendingKill 플래그가 1인 오브젝트도 파괴시킨다. )                          
이는 Mark 단계와 달리 싱글스레드에서 동작한다. 오브젝트 파괴 동작을 멀티스레드로 하려면 고려해야하는 변수가 너무 많아진다. 그래서 싱글스레드로 수행한다.        
다만 한 프레임에 모든 오브젝트를 Sweep해버리면 순간 프레임 저하가 발생할 수도 있을 것 같아서 매 프레임 조금씩 Sweep을 하는건 어떨까 고민 중에 있다.             
위의 단계를 주기적으로 ( ex) 10초에 한번 ) 수행하면서 엔진은 메모리 누수를 방지해주어 C++에서 가장 귀찮은 메모리 누수 관리에 대한 부담을 덜어준다. 그냥 프로그래머는 일반 포인터에 힙 할당을 하고 사용하지 않으면 그냥 해당 포인터를 NULL 값으로 셋팅만해주어도 엔진이 알아서 메모리를 회수해주는 개념이다.             
         
       
필요한 기능들이 이미 만들어져 있어서 이틀만에 금방 만들었다.       
성능은 느리지만 동작은 한다.       

[시연 영상](https://youtu.be/E4CNOIXYQnQ)               
[엔진 내 구현 소스코드](https://github.com/SungJJinKang/DoomsEngine/tree/main/Doom3/Source/Core/GarbageCollector)                

-------------------------------         

이제는 성능을 높여갈 차례이다.     

일단 필자의 계획은 **순회를 하면서 어떤 오브젝트를 파괴해주어야할지 결정하는 과정은 멀티스레드로 분배해서 수행**할 것이다. ( 다만 유효하지 않은 포인터에 null 값을 넣는 과정에서 데이터 레이스, 가비지 컬렉터를 위한 flag 셋팅 과정에서 캐시 동기화, false sharing ( 각 오브젝트들의 flag들이 연속되게 할당되어 있다 )으로 인한 성능 저하가 약간은 걱정되기도 한다. )           
멀티스레드로 파괴할 오브젝트들이 모두 정해지면 **파괴하는 작업 ( delete )은 싱글 스레드에서 수행**할 것이다. ( 파괴 동작을 멀티스레드로 하려면 고려해야할 변수가 너무 많아진다. )           

일단은 False Sharing과 같은 문제들을 차치해두고 단순히 **멀티스레드로 구현을 해보니 Mark 단계의 성능이 3배가 빨라졌다.**                 
매우 만족하는 결과이다.            

Marking 중인 오브젝트 개수와 Marking이 끝난 오브젝트 개수를 카운팅하기 std::atomic을 사용하는데 엔티티 개수가 많아질 수록 카운팅 연산 횟수가 늘어나면서 인접한 atomic 변수들간의 **[false sharing](https://sungjjinkang.github.io/cachecohrencyAndFalsesharing)**이 우려되어서 중간에 padding을 넣어주었다.                   


```cpp
namespace dooms::gc::garbageCollectorSolver
{
	struct GCMultithreadCounter
	{
	    std::atomic<size_t> workingOnRootObjectCount;
	    UINT8 padding[64]; // padding for preventing false sharing
	    std::atomic<size_t> completedOnRootObjectCount;
	};
}
```

여기서 조금 더 최적화를 할까 생각을 하였지만 프로파일링을 해보니 현재 엔진 규모에서 GC 동작 자체가 그렇게 느린 동작이 아니라서 그냥 두기로 하였다.      

-------------------------------       

2022-02-05           
         
이후 좋은 아이디어가 떠올라 최적화 작업을 한번 더 하였다.                     
기존에는 포인터가 유효하지 않은 주소를 가진 경우 null 값을 넣어줄 때 포인터가 가지고 있는 주소가 유효한지 판단하기 위해 게임 엔진 내의 모든 DObject를 들고 있는 해시테이블을 참조했었다. 해시테이블 물론 빠르다. 그렇지만 오브젝트 개수가 많아지면 많이질수록 이 또한 부담이 될 수 있다.                     
그래서 최적화를 수행하였다. "그냥 포인터가 가지고 있는 오브젝트가 유효한지 그렇지 않은지를 해시테이블로 확인하지 말고 그냥 참조해보는 것이다"          
그럼 유효하지 않은 주소인 경우 Exeption이 발생하지 않나? 그렇다. 그렇지만 Exception이 발생한 경우 Catch하면 된다. 그리고 그 주소를 유효하지 않은 주소라고 판단하면 되는 것이다.        
게임 내의 오브젝트들은 파괴를 하여도 즉시 파괴되지 않고 ( PendingKill ) 가비지 컬렉터에 의해 메모리가 회수되기 때문에 포인터 타입의 멤버변수가 유효하지 않은 주소를 들고 있을 가능성은 매우 매우 극히 드물다. ( 프로그래머가 의도적으로 주소에 이상한 값을 넣지 않는 이상말이다. ) 이러한 매우 매우 드문 상황을 대비해서 매번 해시 테이블을 탐색하는건 낭비이다.           
그냥 참조하고 Exception이 발생하는지 확인해보면 된다.          
실제 작성한 코드는 아래와 같다.          

```cpp
FORCE_INLINE static bool CheckDObjectIsValid(const dooms::DObject* const dObject)
{
	bool isDObjectValid = false;

#if _MSC_VER
	__try 
	{
#else
#error Unsupported Compiler
#endif
		isDObjectValid = IsValid(dObject);
#if _MSC_VER
	}
	__except (_exception_code() == EXCEPTION_ACCESS_VIOLATION ? EXCEPTION_EXECUTE_HANDLER : EXCEPTION_CONTINUE_SEARCH)
	{
		isDObjectValid = false;
		D_ASSERT(IsLowLevelValid(dObject) == false);
	}
#endif

	return isDObjectValid;
}
```
뭐 이러한 계획된 Exception의 경우 SEH ( Structured Exception Handling )라고 부른다.             

프로파일링을 해보니 GC의 Mark 단계에서 20% ~ 30% 정도의 성능 향상이 있었다.                



references : [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Objects/Optimizations/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Objects/Optimizations/)          