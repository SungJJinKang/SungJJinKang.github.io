---
layout: post
title:  "condition variable이 atomic임에도 불구하고 write할 때 mutex로 보호해줘야하는 이유"
date:   2021-03-28
categories: ComputerScience
---

선행 지식 :    
condition_variable은 무조건 mutex를 lock한 후 predicate를 체크함 (lock 못하면 lock할 때까지 wait한 후 predicate 체크).            
condition_variable은 다시 waiting에 들어가는 동시에 얻었던 mutex를 다시 release함.       

아래와 같은 경우를 가정해보자.     
std::atomic<bool> sharedVariable이 predicate의 조건으로 사용됨.     

1. Waiting 쓰레드가 condition variable waiting에서 wake up함, 그리고 다시 mutex를 획득하고 shared variable이 true인지 predicate를 체크함. 여전히 predicate는 false임. 이제 다시 쓰레드를 기다리게(wait) 해야함. ( **아직 waiting 상태에 못들어감, predicate를 체크 한 후 다시 wait를 들어가기 직전의 상태 -> 여전히 Waiting쓰레드가 mutex를 가지고 있는 상태** )     
2. **mutex lock 없이** Worker 쓰레드가 shared variable에 true를 씀. ( predicate 충족 )                 
3. Worker 쓰레드가 condtion variable에 notification을 보냄. 그런데 위의 Waiting 쓰레드는 아직 wait 상태에 들어가지 않아서 이 notification을 못 받음.           
4. Waiting 쓰레드가 다시 wait에 들어감. condition variable의 notification은 이미 보내졌기 때문에 Waiting 쓰레드는 다음 notification을 기다려야함. 이렇게 되면 Waiting 쓰레드가 영원히 wait하는 경우도 생길 수 있다.          

**2번에서 Worker 쓰레드가 shared variable에 true를 쓰게 전 mutex를 lock을 한다면 1번에서 Waiting 쓰레드가 condition variable이 mutex를 획득한 상태이기 때문에 Waiting 쓰레드가 mutex를 release하기 까지 기다려야함.**       
**Waiting 쓰레드는 predicate 체크 후 다시 Waiting 상태에 들어가는 동시에 mutex를 release 하기 때문에 Worker 쓰레드가 던지는 notification을 Waiting 쓰레드가 받지 못하는 일이 발생할 가능성이 사라짐.**         