---
layout: post
title:  "condition_variable와 Hurry Up And Wait, Spurious wakeups(가짜 Wake up)"
date:   2021-05-20
categories: C++
---

이 글에서는 condition_variable의 기본적인 작동 원리와 그와 관련된 두 문제인 Hurry Up And Wait, Spurious wakeups에 대해 알아보겠다.       

condition_variable은 다른 쓰레드가 공유되는 변수(조건 변수)를 수정하고 condition_variable에게 알릴 때(notify) 까지, 한 쓰레드 혹은 여러 쓰레드를 동시에 블락하기 위해 사용된다.      

condition_variable의 기본적인 동작 원리를 설명해보자면 wait 함수와 notify 함수를 알면된다.        
우선 wait함수는 **현재 lock 중인 mutex를 unlock한 후 block상태에 들어가 다른 쓰레드에 의해 notify될 때 까지 현재 쓰레드를 block상태에 있게 만드는 함수이다.**      
wait함수에 매개변수에 따라 다른 쓰레드에 의해 notify된 후 어떤 동작으로 작동하는지가 결정이된다.     
우선 매개변수로 unique_lock만을 받는 경우는 다른 쓰레드가 notify하여 wake up된 즉시 **다시 lock을 획득하고 wait함수를 바로 빠져나간다.**     
반면 매개변수로 unique_lock과 Predicate 즉 조건 함수를 받는 경우에는 다른 쓰레드가 notify하여 wake up된 후 **다시 lock을 획득하고 Predicate 조건 함수를 실행하여 true가 return된 경우 그대로 wait함수를 빠져나가고 그렇지 않은 경우에는 unlock 후 다시 blocking 상태에 들어간다**           
Predicate에 성공하면 획득했던 **lock을 유지한채로 wait문을 빠져나가 이후 명령어들을 실행한다.**         
**중요한 것은 wake up된 후 lock을 다시 획득한 후에 Predicate를 검사한다는 것이다.**      

wait문의 내부 구현은 아주 간단하다.     
```c++
template< class Predicate >
void wait( std::unique_lock<std::mutex>& lock, Predicate pred )
{
    while (!pred()) 
    {
        wait(lock);
    }
}
```

notify함수는 notify_one과 notify_all이 있는 데 notify_one은 위에서 wait함수로 wait 중인 쓰레드들 중 한 쓰레드를 wake up하는 함수이고 notify_all은 wait 중인 모든 쓰레드를 wake up하는 함수이다.   

notify_one, notify_all, wait 함수는 모두 atomic하고 순차적 일관성이 보장된다.      
예를 들어보면 A 쓰레드에 의해 notify_one이 호출된 **이후** B 쓰레드에서 wait를 시작한 경우 이 **B 쓰레드가 A 쓰레드의 notify에 의해 wake up될 가능성은 전혀 없다는 것이다.**     

condition_variable을 사용하면서 주의해야할 것이 있다.                 
조건 변수(Predicate 함수 내에서 조건을 만족하는지 확인하기 위해 사용되는 변수)를 수정하는 쓰레드는 반드시 mutex를 락한 후 락을 한 상태에서 조건 변수를 수정하고 condition_variable에게 알려야(notify)한다. 

간단히 설명하자면 notify에 의해 wake up된 쓰레드는 lock을 재획득하고 어떤 조건 변수에 대한 Predicate를 검사한 후 실패시 다시 unlock 후 blocking 상태에 들어간다.        
그런데 만약 wake up된 쓰레드 A가 Predicate를 검사한 후 실패하여 다시 blocking 상태에 들어가기 직전 다른 쓰레드 B에서 조건변수를 Predicate를 만족하게 수정하고 notify까지 한다면 어떻게 될까??   
이 경우 wake up된 후 Predicate에 실패하여 다시 blocking 상태에 들어가기 직전인 함수는 notify를 받지 못하게 되어 영원히 blocking 상태에 빠지게 될 수 있다.        
이는 notify를 받는 쓰레드는 waiting 상태에 들어간 쓰레드여야 하는 데 위의 상황에서 쓰레드 A는 Predicate문을 실행 후 실패하여 다시 wait를 들어가기 직전인 상태이다. 즉 쓰레드 A는 쓰레드 B의 notify를 받지 못한다.       
이렇게 되면 쓰레드 A는 또 다른 쓰레드가 notify 해주지 않으면 영원히 waiting 상태에 빠진다.        

그래서 이를 해결하기 위해서는 쓰레드 B에서 조건 변수를 수정하고 notify하기 전 반드시 쓰레드 A가 condition_variable에서 lock하는 mutex를 lock한 후 수정, notify하여야 한다.     
이렇게 하면 조건 변수를 수정하고 notify하는 코드 블록과 Predicate를 검사하고 다시 waiting하는 코드 블록이 동시에 진행될 수 없기 때문에 위와 같은 문제를 막을 수 있다.       
자세한건 [이 글](https://sungjjinkang.github.io/computerscience/2021/03/28/condtionvariable_atomic.html)을 참고하라.              
  
condition_variable의 사용 방법은 아래와 같다.     

```c++
#include <iostream>
#include <string>
#include <thread>
#include <mutex>
#include <condition_variable>
 
std::mutex m;
std::condition_variable cv;
std::string data;
bool ready = false;
bool processed = false;
 
void worker_thread()
{
    // Wait until main() sends data
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, []{return ready;});
 
    // after the wait, we own the lock.
    std::cout << "Worker thread is processing data\n";
    data += " after processing";
 
    // Send data back to main()
    processed = true;
    std::cout << "Worker thread signals data processing completed\n";
 
    // Hurry Up And Wait 문제를 방지하기 위해 notify 전 mutex를 unlock한다. 밑에서 설명할 것이다.       
    lk.unlock();
    cv.notify_one();
}
 
int main()
{
    std::thread worker(worker_thread);
 
    data = "Example data";
    // send data to the worker thread
    {
        std::lock_guard<std::mutex> lk(m);
        ready = true;
        std::cout << "main() signals data ready for processing\n";
    }
    cv.notify_one();
 
    // wait for the worker
    {
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, []{return processed;});
    }
    std::cout << "Back in main(), data = " << data << '\n';
 
    worker.join();
}
```
  
그럼 이제 "Hurry up and Wait"와 "Spurious Wake up" 문제에 대해 알아보겠다. 

"Hurry up ans Wait" 문제는 위의 코드에서 바로 확인할 수 있다.  

```c++
void worker_thread()
{
    // Wait until main() sends data
    std::unique_lock<std::mutex> lk(m);
    cv.wait(lk, []{return ready;});
 
    // after the wait, we own the lock.
    std::cout << "Worker thread is processing data\n";
    data += " after processing";
 
    // Send data back to main()
    processed = true;
    std::cout << "Worker thread signals data processing completed\n";
 
    // Hurry Up And Wait 문제를 방지하기 위해 notify 전 mutex를 unlock한다. 밑에서 설명할 것이다.       
    lk.unlock();
    cv.notify_one();
}
```

주목해서 보아야할 부분은 마지막 lk.unlock과 cv.notify_one() 부분이다.     
만약 lk.unlock이 없다면 위의 코드는 어떻게 작동할까 생각해보자.     
우선 메인쓰레드는 processed 조건변수에 대한 Predicate로 wait 중에 있다.

```c++
{
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, []{return processed;});
}
```

1) worker_thread가 lk.unlock 없을 시 notify를 한다    
2) **메인 쓰레드는 Wake up된 후 mutex를 다시 획득하려 한다, 그렇지만 여전히 worker_thread가 lock을 가지고 있다(worker_thread에서 lock은 unique_lock가 파괴될 때 release(unlock) 된다)**            
3) **메인 쓰레드는 다시 blocking 상태에 들어간다**       
4) worker_thread가 종료되며 unique_lock도 파괴되고 mutex가 release(unlock) 된다       
5) blocking 중이던 메인쓰레드가 lock을 획득 후 predicate를 체크하고 wait를 빠져나간다.      

**이 경우 메인 쓰레드가 불필요하게 한번 blocking 되었다가 worker_thread가 lock을 release(unlock) 한 후 다시 lock을 획득하여** Predicate문을 실행하게 된다.     
반면 worker_thread가 notify_one 전 mutex를 unlock한 경우에는 Wake up된 메인 쓰레드는 곧 바로 mutex lock을 획득할 수 있고 바로 Predicate문을 실행할 수 있다.          
디테일하게 설명하자면 Predicate 실패 후 waiting 상태가 될 때는 condition_variable 내부의 별도의 대기 queue에서 쓰레드가 대기하게 되고, 한번 Wake up된 후 다시 lock을 얻지 못해 blocking 될 때는 mutex의 대기 큐에 들어간다.     

이렇게 불필요하게 blocking을 하는 문제를 "Hurry up and Wait"라 한다.   
대부분의 pthread ( POSIX Thread )의 구현들은 이 문제를 방지하기 위해 notify하여 Wake up될 쓰레드를 실제로는 Wake up하지 않고 대신 mutex의 대기 큐 맨 앞에 Wake up될 쓰레드를 넣어 불필요한 waiting - wake up - blocking - unblock을 방지한다.     
이렇게 하면 wake up 쓰레드는 mutex가 unlock된 후에야 비로소 Lock을 획득하게 된다. 불필요하게 중간에 다시 blocking되는 일이 발생할 가능성을 없앤 것이다.         


다음으로 "Spurious Wake up" 문제에 대해 설명해보겠다.      
"Spurious Wake up" 즉 가짜 Wake up 문제는 위의 "Hurry up and Wait" 문제에서 worker_thread의 **notify 전 mutex를 unlock한 경우 발생할 수 있는 문제이다.** ( 참 아이러니다..... )     
이 가짜 Wake up 문제는 간단한 예시를 들어 설명하겠다. 
우선 이번에는 두개의 worker_thread가 어떤 queue integer값을 랜덤으로 넣는 함수라고 생각하자.
그리고 print_thread는 wait함수에서 queue에 interger 값이 1개 이상인지를 검사하고 그럴 경우 wait문을 빠져나가 lock을 획득한 채로 deque 후 출력을 한다. 이 과정을 무한으로 반복한다.      
대강 아래와 같은 함수일 것이다. 

```c++
queue<int> printed_queue;
void worker_thread1()
{
    while(true)
    {
        std::unique_lock<std::mutex> lk(m);

        printed_queue.push(rand());

        lk.unlock();
        cv.notify_one();
    }
}
void worker_thread2()
{
    while(true)
    {
        std::unique_lock<std::mutex> lk(m);

        printed_queue.push(rand());

        lk.unlock();
        cv.notify_one();
    }
}

void print_thread()
{
    while(true)
    {
        // 큐에 1개 이상의 값이 들어 올때까지 기다린다.
        std::unique_lock<std::mutex> lk(m);
        cv.wait(lk, []{return printed_queue.empty() == false;});

        while(printed_queue.empty() == false)
        {
            std::cout << printed_queue.front();
            printed_queue.pop();
        }
        
        //자동으로 mutex는 release(unlock)됨
    }
}
```

그리고 다른 쓰레드에서 printed_queue에 새로운 interger 값을 삽입하는 부분들도 모두 mutex lock으로 보호된다.       
그럼 아래와 같은 상황을 가정해보자.     

1) worker_thread1에서 lock을 획득한 후 queue에 값들을 삽입하고 다시 lock을 release(unlock)하였다.그리고 condition_variable에 notify를 하려한다. **근데 갑자기 context switch가 발생하여 다른 쓰레드고 context가 넘어간 것이다. 아직 notify를 하지 못하였다.**                  
2) 그리고 worker_thread2가 실행되어 lock을 획득한 후 queue에 값들을 삽입하고 다시 lock을 release(unlock)하였다. 그 후 condition_variable에 notify한다.       
3) 그럼 print_thread는 wake up되고 당연히 queue에는 worker_thread1과 worker_thread2가 삽입한 두 값이 ( 1), 2) ) 있으니 wait를 빠져나가 queue에 있는 모든 interger 값을 dequeue 후 출력한다. 그리고 다시 while문을 돌아 wait에 들어간다.          
4) 이제 다시 worker_thread1가 context를 획득하고 이전에 실행 못했던 notify를 한다.          
5) print_thread는 wake up되고 Predicate를 확인하는 데 이미 queue는 3)에서 모두 dequeue 되었기 때문에 Predicate문을 만족하지 못하고 다시 wait에 들어간다.     

결국 5)에서 print_thread는 Predicate문을 만족하지도 못하는 데 wake up되어 Predicate를 검사한 것이다. 이를 "가짜(Spurious) Wake up"이라 하는 것이다.         

여기서 중요한 것은 worker_thread1가 notify전 mutex를 unlock하지 않았더라면 2)에서 worker_thread2가 lock을 획득하지 못하게 되고 worker_thread2가 notify를 하는 일은 없었을 것이다.    
"Hurry up and Wait" 문제를 막고자 notify 전 mutex unlock을 한 것이 가짜 Wake up 문제를 유발한 것이다.         
