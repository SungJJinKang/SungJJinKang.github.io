---
layout: post
title:  "이미 Lock된 Critical Section을 Acquire 시도할시 항상 커널 모드로 전환(Sleep)되나?"
date:   2023-04-09
tags: [ComputerScience, UE, Recommend]
---          
             
이미 Lock된 Critical Section을 Acquire 시도할시 항상 커널 모드로 전환(Sleep)되나?          
UE4 코드 기준 "Windows 플랫폼은 그렇지 않다", "다른 플랫폼은 그렇다"가 답이다.         

일단 UE4의 "WindowsCriticalSection.h"쪽 코드를 보면 [SetCriticalSectionSpinCount](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-setcriticalsectionspincount)을 통해 이미 Lock된 경우라도 4000번 Spin Loop으로 Acquire을 시도한다. 그래도 실패하면 Sleep한다.             
![WindowCriticalSection](https://user-images.githubusercontent.com/33873804/230764377-ee96f9a6-a336-40f1-aa0e-16df02f2028d.png)                      
이는 Critical Section을 매우 짧은 시간 Lock하는 코드가 많은 경우에 유리하다.(값 비싼 커널 모드로의 전환을 하지 않아도 되니...)              
             
다른 플랫폼의 경우 Lock된 Critical Section을 Acquire 시도할시 항상 커널 모드로 전환된다.          
다른 플랫폼은 Windows의 "SetCriticalSectionSpinCount"의 equivalent에 해당하는 함수가 존재하지 않는다.            
           
               
참고로 Lock되지 않은 Critical Section을 Lock하는 비용은 매우 저렴하다.(커널 모드로 전환되지 않고 유저 모드에서 처리한다.)                      
                  
참고 : [Critical Section vs Mutex](https://stackoverflow.com/a/800422), [Overhead of pthread mutexes?](https://stackoverflow.com/a/1278965)       