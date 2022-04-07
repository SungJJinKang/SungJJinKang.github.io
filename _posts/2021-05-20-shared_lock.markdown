---
layout: post
title:  "One Writer, Multiple Reader에서 불필요한 lock을 막기 위해 shared_mutex 사용하기"
date:   2021-05-20
categories: C++
---

shared_mutex는 쓰레드간 어떠한 공유되는 변수가 있을 때 어떠한 쓰레드들은 그 변수에 write를 수행하고 어떠한 쓰레드들은 read만을 수행할 때 read 쓰레드들이 불필요하게 lock을 수행하는 것을 방지하기 위해 사용된다.      

shared_mutex에는 lock의 두 종류가 있는데 하나는 lock(), 다른 하나는 lock_shared()이다.    
이 둘의 차이를 알기 위해서는 exclusive 모드와 shared 모드의 lock의 차이를 알아야한다.       
**exclusive 모드**는 하나의 쓰레드만이 독점적으로 lock을 획득한 상태를 말한다. 다른 쓰레드가 lock을 얻으려하면 block상태에 들어간다.       
mutex가 shared 모드 상태로 lock되어 있는 경우 exclusive 모드 lock은 block에 들어간다 ( 이건 다시 확인해보자 ).              
반면 **shared 모드**는 여러 쓰레드가 lock을 획득할 수 있다. 단 해당 mutex가 exclusive 모드로 lock이 된 경우에는 shared 모드의 lock은 block 상태에 들어간다.       

**lock()은 일반적으로 사용하는 lock()과 똑같다. exclusive 모드로 lock을 획득한다.** 한 mutex만이 lock을 얻을 수 있고 이미 다른 쓰레드에 의해 lock이 된 경우 block을 한다.         
일반적으로 직접적으로 lock을 호출하지 않고 std::unique_lock, std::scoped_lock, and std::lock_guard을 통해 lock을 하는 것이 장려된다( 실수를 줄여준다 ).                   

반면 **shared_lock**은 shared 모드로 lock을 하여 **여러 쓰레드가 동시에 lock을 획득**할 수 있다.      
위의 이러한 특징 때문에 **exclusive 모드는 공유 자원에 write를 수행할 때** 사용되고 **shared 모드는 공유 자원에 해당 자원을 훼손하지(값을 바꾸지) 않는 read시에 사용된다.**       
**일반적인 mutex를 사용했을 때는 read시에도 exclusive lock을 해주어야하는 데 이 경우 공유 자원이 훼손될 가능성이 없음에도 불구하고 동시간에 하나의 쓰레드만이 read를 한다는 단점이 있다.**        
반면 **shared_mutex를 사용하면 여러 쓰레드가 동시에 read를 할 수 있다**는 장점이 있다.

사실 예시 경우는 그냥 atomic을 쓰는게 좋다. 그렇지만 shared_mutex를 어떻게 사용하는지를 보여주는 예시이니 그냥 참고만하기 바란다.  
```cpp
#include <iostream>
#include <mutex>
#include <shared_mutex>
#include <thread>

std::shared_mutex mutex_;
int value = 0;
void writer_thread()
{
    std::unique_lock lock(mutex_); // unique_lock을 사용했기 때문에 하나의 쓰레드만 mutex lock에 접근한다. 
    ++value; // atomic이 아니라 mutex로 보호해주어야 한다. 
}

void read_thread()
{
    std::shared_lock lock(mutex_); // 여러 쓰레드가 동시에 lock을 획득할 수 있다. 
    std::cout << value << std::endl;
}
```
