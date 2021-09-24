---
layout: post
title:  "std::atomic의 숨겨진 동작 원리 ( Atomicity, Volatile )"
date:   2021-09-24
categories: ComputerScience
---

우선 atomicity라는 개념에 대해 알아보자.        

**atomicity란 indivisible이란 뜻으로 해당 연산 ( 동작 ) 사이에 다른 동작이 끼어들 수 없다는 의미이다.** 그러니깐 atomicity한 동작을 하는 중간에는 다른 동작이 끼어들 수 없다.                          
이러한 atomicity를 위해서는 우선 접근하거나 쓰기를 수행하려는 **데이터가 레지스터 사이즈에 align되어 있어야하고 변수의 크기가 레지스터 사이즈와 같아야한다.** ( [메모리 Alignment](https://sungjjinkang.github.io/computerscience/2021/03/28/memoryalignment.html) )        
위의 조건을 충족하는 변수에 대해서는 **읽기와 쓰기 연산은 항상 atomic**하다. 잘 기억해두라. **읽기와 쓰기 연산에 대해서만**이다...      
여전히 i++와 같은 읽은 후 1을 더하고 다시 그 결과를 쓰는 동작은 atomic하지 않다. 읽기, 쓰기 그 각각은 atomic 하지만 이들을 여러개 합친 동작은 그 사이에 다른 동작이 낄 수 있으니 atomic하지 않은 것은 당연하다.      
( 참고로 아무리 위의 조건을 충족하여도 atomicity한 명령어를 사용하지 않으면 연산이 atomicity하지 않는데 MSVC 같은 경우에는 변수가 위의 조건을 충족하고 volatile한 경우에는 알아서 atomicity한 명령어를 사용해서 atomicity를 충족시켜준다. )                 

------------------------

여기서 잠깐 volatile이라는 개념에 대해 알아보고 가자.           
종종 volatile이라는 개념과 atomicity를 혼용하는 경우가 있다.       
**volatile이라는 개념은 상호 배제 ( mutual exclusion )과 Atomicity, Memory Ordering와는 아무런 관련이 없다!!!**              
volatile이라는 disk로부터의 메모리 매핑 I/O ( Memory Mapped IO )위해서 디자인된 개념인데 **volatile은 해당 변수가 다른 프로세스 ( 프로그램 )에 의해 접근, 변화될 수 있기 때문에 해당 변수에 대해서는 컴파일러에게 레지스터 할당과 같은 최적화를 수행하지 말라고 명시**하는 것이다.     
여기서 말하는 **"레지스터 할당 최적화"란 연산을 하는 과정에서 해당 변수에 대한 연산 결과를 레지스터에 임시로 저장하지 말고 반드시 메모리 ( 캐시든 DRAM )에 쓰라 ( Write )(!!!!!)는 것을 말한다.**                  

```
int result = 0;
for ( size_t i = 0 ; i < 10000 ; i++ )
{
    result++;
}
```
위의 코드를 보자. 메모리(캐시든 DRAM이든)에 result라는 변수가 있는데 10000번의 루프 동안 result++의 결과를 메모리에 쓸 것인가???       
그렇게 하지 말고 **그냥 레지스터에(!!) 각 루프의 연산 결과를 임시로 저장하면서 매 루프마다 연산을 한 후 마지막에 한번만 메모리에 쓰면 되지 않을까?**                    
메모리 계층상 레지스터는 캐시와 DRAM에 비해 월등히 빠르니 굳이 result++의 결과를 메모리(캐시나 DRAM)에 쓰기보다는 레지스터에 저장해두었다가 루프가 다 끝나면 메모리로 옮기면 된다.         

이러한 최적화는 다른 코어나 쓰레드, 프로그램이 result의 값을 알고자 하지 않는 경우에는 아무런 문제가 되지 않는다.         
**그런데 문제는 이 코어가 Loop를 도는 와중에 다른 코어나, 쓰레드 혹은 다른 프로그램 ( 메모리 매핑 I/O )이 result를 읽으려고 할 때 메모리에는 실제 result의 값이 아니라 0이 쓰여 있다는 것** ( 실제 result의 값은 레지스터에 임시로 저장하고 있다 )이다.               
실제 result 값은 루프가 어느 정도 진행된 0과 10000 사이의 값이지만 메모리에는 0이 쓰여 있어 0을 읽는 것이다.      
그래서 Volatile이라는 개념을 도입해서 위와 같은 **레지스터 할당 최적화를 하지 말고 연산의 결과를 항상 메모리에 쓰라고 컴파일러(!)에게 지시**할 수 있다.              

이 **Volatile, 즉 연산의 결과를 항상 메모리 ( 캐시든 DRAM이든 ) 저장한다는 특징과 Atomicity의 개념이 합쳐지면 우리는 Volatile 변수의 값의 변화가 항상 다른 쓰레드들에게 보인다는 효과를 얻을 수 있다.** Volatile 즉 연산의 결과를 항상 메모리에 쓰면 Cache Coherent나 Memory Snooping와 합쳐져 코어들이 항상 다른 코어에서 Volatile 변수의 값의 변화를 즉시 볼 수 있다는 것을 의미한다. ( [MESI 프로토콜](https://sungjjinkang.github.io/computerscience/2021/04/06/cachecoherency.html) )                     
그런데 만약 Volatile은 하지만 Atomicity하지 않으면 어떠한가? 그러면 Atomic 하지 않아 연산을 하는 와중에 다른 코어가 그 값을 읽어 갈 수 있고 그러면 연산이 완전하게 끝나지 않은 변수 값을 읽어갈 수 있기 때문에 어떤 연산의 결과가 항상 다른 코어에게 보인다는 효과를 잃어버린다.    

------------------------------

다시 std::atomic으로 돌아가자.          
**std::atomic은 Atomicity와 Memory Ordering이 합쳐진 개념**이다.                 
우선 atomicity 부분을 보면 위에서 말했듯이 변수의 사이즈가 레지스터 사이즈와 같고, 레지스터 사이즈에 Align되어 있으면 해당 변수에 대한 읽기, 쓰기 연산은 atomicity가 보장된다고 말했다.           
그럼 변수의 사이즈가 레지스터 사이즈와 다르거나 레지스터 사이즈에 Align되어 있지 않으면 어떻게 될까?      
그리고 Atomicity한 변수의 읽기, 쓰기 연산은 Atomicity하다고 했는데 i++와 같은 연산은 Atomicity하지 않다.          
그래서 std::atomic은 여러 메커니즘을 추가하여서 위와 같은 연산들에 대해서도 atomicity를 보장해준다.           
                
대표적으로 std::atomic::fetch_add 즉 Read-Modify-Store 연사이 있는 데 이는 CPU에 따라 다르지만 대채적으로 atomic하지 않다. ( 명령어 하나로 처리하지 못한다 )        
i++ 같은 것이 그 예이다. i의 값을 읽고(atomic), 1을 더하고(non atomic), 다시 1을 더한 값을 써야(atomic)한다. 이 동작은 atomicity하지 않다. **(읽기), (쓰기) 동작 (각각은 atomicity)하지만, (값을 읽은 후 1을 더하고 다시 쓰는 동작은 atomic하지 않다.)**            

그래서 C++ std::atmoic는 실제로는 atomic하지 않은 연산에 Locking과 같은 추가적인 매커니즘을 더해서 atomic하게 만든다.               
X84 초기에는 메모리 공유 버스에 Lock을 걸어서 Read-Modify-Store을 수행하였다. 그렇지만 이는 다른 모든 CPU가 메모리가 접근하는 것을 막아버려 좋은 방법은 아니다.            
그래서 현재의 X86, 64는 MESI와 같은 Cache Coherency Protocol들을 사용하는 데 이는 전체가 아닌 **특정 캐시 라인만을 Exclusive 상태로 만들어 다른 CPU의 해당 캐시 라인만으로의 접근을 막는 방법으로 매우 효율적이다.** ( [MESI 프로토콜](https://sungjjinkang.github.io/computerscience/2021/04/06/cachecoherency.html) )                 
atomic 연산을 하는 동안 해당 캐시 라인에 Modified 상태로 만든다. 이때 다른 코어가 해당 캐시 라인을 읽으려고하면 ( 캐시 라인의 Modified 상태를 가지고 있는 코어는 Snooping 중이다. ) 해당 캐시 라인의 Modified 상태를 가지고 있는 주인 코어는 atomic 연산이 다 끝날때까지 캐시 라인을 flush 해주지 않고 Modified 상태를 가지고 있는다. ( 그럼 당연히 해당 캐시 라인을 읽으려는 다른 코어들은 주인 코어가 캐시 라인 flush를 해줄때까지 기다려야한다. ) ( 추가적으로 X86은 강한 메모리 모델을 가지고 있어서 Cache Coherency를 해칠 시 가장 최근에 Cache Coherency를 준수 했던 상태로 Rollback한다 )
CPU에 따라 Locking을 할 수도 있고 MESI 프로토콜을 이용해서 Atomicity를 구현할 수 있는데 이를 확인하려면 [atomic_is_lock_free](https://en.cppreference.com/w/cpp/atomic/atomic_is_lock_free)를 사용하면 된다.         


-------------------------------------------------


std::atomic의 핵심은 위에서 설명한 Atomicity의 개념이고 아무리 Atomicty를 보장해주어도 메모리 Reordering이 발생하는 경우에는 해당 Atomic 연산의 결과를 다른 코어가 즉시 볼 수 없는 경우가 생긴다. ( 여기서 메모리 Reordering은 MESI 프로토콜에서 Cache Flush를 즉시 수행하지 않는 경우와 같은 것을 말한다. ) 그래서 std::atomic에는 Atomicity에 더해 Memory Reordering을 막는 Memory Order 변수를 연산시 추가로 전달하여 std::atmoic 변수에 대한 한 코어에서의 연산 결과가 반드시 다른 코어에서 즉시 보이게 만든다.      
딱히 memory_order을 지정해주지 않았다면 std::memory_order_seq_cst이 기본 값으로 들어간다.                        
메모리 Reordering에 대해서는 [이 글](https://sungjjinkang.github.io/computerscience/2021/05/13/MemoryReordering.html)을 읽어보기 바란다.        


-------------------------------



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
