---
layout: post
title:  "reflection 시스템을 이용한 C++ 가비지 컬렉터 ( +클러스터링 ) ( 작성 중 )"
date:   2021-12-01
categories: ComputerScience
---

[지난번 만든 reflection 시스템](https://sungjjinkang.github.io/computerscience/gameengine/2021/11/12/reflection.html)을 이용하여 C++ 가비지 컬렉터를 만들 것이다.            

이전에도 [메모리 누수를 방지하기 위한 구조](https://sungjjinkang.github.io/computerscience/gameengine/2021/09/25/dangling_pointer.html)가 존재했지만 이 방법은 씬이 끝나거나 특수한 상황에서 임의로 모든 DObject들을 한꺼번에 회수해주는 기능이라 엄연히 참조가 되지 않은 오브젝트를 주기적으로 회수해주는 가비지 컬렉터와는 역할이 다르다.              

**언리얼 엔진의 가비지 컬렉터**를 생각하면 된다.     
가비지 컬렉터는 루트 오브젝트부터 프로퍼티들을 순회하면서 ( 리플렉션 데이터 이용 ) 최종적으로 참조되지 않은 오브젝트를 회수한다.          
또한 회수된 오브젝트에 대한 주소를 가지고 있는 포인터에는 null 값을 넣어준다. 이를 통해서 게임 내의 IsValid 함수에 대한 안전성을 높인다.        


가비지 컬렉터를 위한 보조 기능들은 다 구현이 되어 있어서 금방 만들 것 같다.             

------------------------------              

동작 원리

1. Unreachability 플래그 셋팅 단계        

프로그램 내의 모든 DObject ( 게임 내 거의 모든 클래스들의 조상 클래스 [참고](https://sungjjinkang.github.io/computerscience/gameengine/2021/09/25/dangling_pointer.html) )들의 Unreachability 플래그를 1로 셋팅한다. 이 플래그도 캐시 극대화를 위해 연속되게 배치하였다. ( SOA!! )                       


2. Mark 단계

**멀티스레드로 각각의 스레드들은 자신에게 할당된 루트 오브젝트들을 순회하면서 해당 루트 오브젝트에서 뻗는 모든 reference를 돌아다니면서 Unreachability 플래그를 0으로 셋팅**한다.           
그 후 최종적으로 Unreachability가 여전히 1인 플래그는 해당 오브젝트에 대한 reference가 없는 것으로 판단하고 파괴한다.        
여기서도 [False sharing](https://sungjjinkang.github.io/computerscience/2021/05/14/cachecohrencyAndFalsesharing.html)을 고려하였다. 각각의 스레드들이 순회를 하다보면 같은 오브젝트에 flag에 셋팅을 할 일이 생길 수 있다. 이때 캐시 동기화가 발생하면서 성능 저하가 발생한다.        
필자는 이를 막기 위해서 각각의 스레드들은 우선 작업 시작시 로컬 변수로 각 오브젝트들에 대한 flag를 저장할 변수를 따로 각자 만든다. 스레드들이 순회를 하는 동안에는 이 로컬 변수에 flag를 적어두었다가 마지막에 모든 스레드가 동작이 끝나면 한번에 오브젝트 flag를 관리하는 전역 변수로 옮길 것이다.         

3. Sweep 단계         

Mark 단계가 끝나고도 여전히 Unreachability 플래그가 1인 오브젝트들은 해당 오브젝트에 대한 reference가 없다고 판단을 하고 파괴시킨다.         
이는 Mark 단계와 달리 싱글스레드에서 동작한다. 오브젝트 파괴 동작을 멀티스레드로 하려면 고려해야하는 변수가 너무 많아진다. 그래서 싱글스레드로 수행한다.        
다만 한 프레임에 모든 오브젝트를 Sweep해버리면 순간 프레임 저하가 발생할 수도 있을 것 같아서 매 프레임 조금씩 Sweep을 하는건 어떨까 고민 중에 있다.             


-------------------------------         

처음에는 우선 기능 구현에 집중을 하고 기능이 구현되면 성능을 높여갈 예정이다.        

**기본적인 동작 원리**는 이렇다.           
일정 주기마다 루트 오브젝트를 순회한다.         
루트 오브젝트에서 D_PROPERTY가 선언된 모든 프로퍼티 ( 멤버 변수 )들을 순회한다.          
프로퍼티 타입이 DObject 클래스의 자녀 타입인 경우 해당 오브젝트는 이번 루틴에서 다른 오브젝트에 의해 참조가 되어 있다고 표시한다.              
최종적으로 모든 루트 오브젝트와 그 프로퍼티들을 순회하고 난 후 여전히 다른 오브젝트에 의해 참조가 되지 않은 오브젝트들을 회수 ( 파괴 )해준다.         

일단 필자의 계획은 **순회를 하면서 어떤 오브젝트를 파괴해주어야할지 결정하는 과정은 멀티스레드로 분배해서 수행**할 것이다.       
멀티스레드로 파괴할 오브젝트들이 모두 정해지면 **파괴하는 작업 ( delete )은 싱글 스레드에서 수행**할 것이다. ( 파괴 동작을 멀티스레드로 하려면 고려해야할 변수가 너무 많아진다. )               


-------------------------------     

[엔진 내 구현 소스코드](https://github.com/SungJJinKang/DoomsEngine/tree/main/Doom3/Source/Core/GarbageCollector)               


references : [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Objects/Optimizations/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/ProgrammingWithCPP/UnrealArchitecture/Objects/Optimizations/)          