---
layout: post
title:  "atomic변수에서 Read-Modify-Store Operation은 locking없이 atomic할까?"
date:   2021-04-10
categories: ComputerScience
---

선행지식 :      

std::atomic은 여러 기능을 가지고 있다.       
우선 이름대로 명령어를 atomic하게 만들어낸다. atomic이란 indivisible이란 뜻으로 여러 Cycle에 걸쳐 수행하는 게 아닌 하나의 Cycle 내로 작업을 처리한다는 것이다.          
이러한 atomicity를 위해서는 우선 접근하거나 쓰기를 수행하려는 데이터가 레지스터 사이즈에 align되어 있어야한다. ( [메모리 Alignment](https://sungjjinkang.github.io/computerscience/2021/03/28/memoryalignment.html) )            
하지만 하드웨어, OS에 따라 atomic 변수가 수행하는 작업이 무조건 atomic(하나의 Cycle만으로 연산을 수행하는)하지는 않다.       
물론 Load, Store 같은 하나 하나의 작업은 atomic하지만(한 Cycle에 수행되지만) 특정 작업을 수행할 때는 명령어가 여러개 필요하다는 뜻이다. ( C++에서는 유일하게 std::atomic_flag만이 모든 작업이 atomic하다(완전히 lock free하다) )              
이 말은 해당 동작 수행이 여러 명령어로 구성되었을 수 있고 그러면 중간에 다른 스레드가 끼어들어 이상한 값을 만들 수도 있다는 것이다.       
대표적으로 std::atomic::fetch_add 즉 Read-Modify-Store 작업이 있는 데 이는 CPU에 따라 다르지만 대채적으로 atomic하지 않다. ( 명령어 하나로 처리하지 못한다 )      
그래서 std::atomic 클래스는 이렇게 한 Cycle 내에 처리 못하는 작업을 수행할 때는 일종의 Locking 메커니즘을 사용한다.        
X84 초기에는 메모리 공유 버스에 Lock을 걸어서 Read-Modify-Store을 수행하였다. 그렇지만 이는 다른 모든 CPU가 메모리가 접근하는 것을 막아버려 좋은 방법은 아니다.            
그래서 현재의 X86, 64는 MESI와 같은 Cache Coherency Protocol들을 사용하는 데 이는 전체가 아닌 특정 캐시 라인만을 Exclusive 상태로 만들어 다른 CPU의 해당 캐시 라인만으로의 접근을 막는 방법으로 매우 효율적이다.        
atomic 연산을 하는 동안 해당 캐시 라인에 Modified 상태로 만든다. 이때 다른 코어가 해당 캐시 라인을 읽으려고하면 ( 캐시 라인의 Modified 상태를 가지고 있는 코어는 Snooping 중이다. ) 해당 캐시 라인의 Modified 상태를 가지고 있는 주인 코어는 atomic 연산이 다 끝날때까지 캐시 라인을 flush 해주지 않고 Modified 상태를 가지고 있는다. ( 그럼 당연히 해당 캐시 라인을 읽으려는 다른 코어들은 주인 코어가 캐시 라인 flush를 해줄때까지 기다려야한다. ) ( 추가적으로 X86은 강한 메모리 모델을 가지고 있어서 Cache Coherency를 해칠 시 가장 최근에 Cache Coherency를 준수 했던 상태로 Rollback한다 )

[https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering](https://en.cppreference.com/w/cpp/atomic/memory_order#Relaxed_ordering)       
[https://en.cppreference.com/w/cpp/atomic/atomic_flag](https://en.cppreference.com/w/cpp/atomic/atomic_flag)           


-----------------------------------------------------

[https://stackoverflow.com/questions/38447226/atomicity-on-x86/38465341#38465341](https://stackoverflow.com/questions/38447226/atomicity-on-x86/38465341#38465341)                 
[https://stackoverflow.com/questions/39393850/can-num-be-atomic-for-int-num](https://stackoverflow.com/questions/39393850/can-num-be-atomic-for-int-num)       
[https://stackoverflow.com/questions/67034400/atomic-variable-also-require-lock-on-read-modify-store-operation](https://stackoverflow.com/questions/67034400/atomic-variable-also-require-lock-on-read-modify-store-operation)        
[https://sungjjinkang.github.io/computerscience/2021/04/06/cachecoherency.html](https://sungjjinkang.github.io/computerscience/2021/04/06/cachecoherency.html)        

-----------------------------------------------------

```c++
std::atomic<int> a;
a++; --> a.fetch_add(1)
++a; --> a.fetch_add(1) + 1
```
