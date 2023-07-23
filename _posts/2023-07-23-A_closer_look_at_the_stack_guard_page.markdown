---
layout: post
title:  "A closer look at the stack guard page(번역)"
date:   2023-07-23
tags: [ComputerScience]
---

[MSDN - A closer look at the stack guard page](https://devblogs.microsoft.com/oldnewthing/20220203-00/?p=106215)을 번역한 글입니다.       
많은 의역이 들어가 있으니 참고해주세요.         
        
----------------------         

[Is­Bad­Xxx­Ptr이 Crash­Program­Randomly라고 불려야 하는 이유](https://devblogs.microsoft.com/oldnewthing/20220203-00/?p=106215)에, 나는 "stack guard page"에 대해 간략하게 소개하였다.          

```
스택의 능동적인 성장(스택이 런타임에 점진적으로 늘어나는거)은 guard pages를 통해 수행된다:             
스택의 마지막 유효한(valid) 페이지를 지나면 guard pages가 있습니다.            
스택이 guard page로 커지면, guard page exception이 발생하는데, 디폴트 exception handler가 새로운 스택 페이지를 commit하고 다음 페이지를 guard page로 설정함으로서 그 guard page exception을 다룹니다.
```

조금 더 자세히 알아보면요..        
        
아래 사진은 스레드가 한동안 동작을 한 후 스택의 상황입니다. 메모리 다이어그램에서 관례적으로 높은 주소가 위에 위치해 있는데, 이는 스택이 아래쪽(낮은 주소로)으로 커져간다는 것을 의미합니다.      
<img class="guard page1" src="/assets/images/guard page1.PNG">              
보통의 커밋(각주 : 피지컬 메모리가 할당된, [참고](https://sungjjinkang.github.io/Window_Memory))된 페이지는 프로그램이 지금껏 사용해온 모든 스택 메모리를 감쌉니다. 지금 당장 그 메모리 모두를 사용하지는 않을 수 있습니다. [red zone](https://devblogs.microsoft.com/oldnewthing/20190111-00/?p=100685) 너머의 메모리는 프로그램의 한계를 벗어난겁니다. 스택 포인터가 스택의 한계까지 갔다가 다시 돌아올 때(각주 : 스택 사이즈에도 제한이 있는데, 스택 포인터가 그 제한 사이즈까지 커졌다가 다시 쪼그라들때), 남겨진 페이지(각주 : 스택 공간이 커지면서 사용했었지만, 다시 스택 공간이 줄어들면서 사용되지 않을 페이지들을 의미함)들은 decommit되지 않는다.(각주 : 피지컬 메모리가 free되지 않는다)           
         
스택 포인터의 한계점 너머의 페이지는 "guard page"라고 알려진 특별한 타입의 커밋된 페이지이다. guard page는 처음 접근될 때 "STATUS_GUARD_PAGE_VIOLATION" exception을 발생시키는 페이지를 말한다.     
     
스택 포인터가 guard page로 이동했다는 것은, 그 스레드가 스택 공간을 한 페이지 더 요구한다는 것을 의미한다.           
         
스레드가 guard page에 속하는 주소에 접근하는 순간, 시스템은 ("PAGE_GUARD" 플래그를 제거함으로서) guard page를 보통의 커밋된 페이지로 바꾸고 "STATUS_GUARD_PAGE_VIOLATION" exception을 발생시킨다. 디폴트 exception 핸들러는 그 주소가 현재 스택의 guard page에 속하는지를 봄으로서 exception을 다루는데, 만약 그 주소가 현재 스택의 guard page에 속한 경우 그 다음 예약(각주 : Virtual address 페이지를 확보해둠, [참고](https://sungjjinkang.github.io/Window_Memory))된 페이지를 guard 페이지로 바꾸고 프로그램을 계속 실행한다.         
<img class="guard page2" src="/assets/images/guard page2.PNG">              
        
guard page에 접근할 때 "PAGE_GUARD" 플래그를 초기화하는 것은 "일단 너가 그 guard page에 접근하면, 그 페이지는 더 이상 guard page가 아니라는 것"을 의미한다. 이는 guard page가 오직 첫 번째 접근에 대해서만 guard page excepion을 발생시킨다는 것을 의미한다. 그래서 만약 너가 guard page exception에 대한 행동을 취하는 것을 실패한다면, 시스템은 그것을 무시할 것이고, 너는 guard page exception에 대해 무언가를 할 수 있는 단 한번은 기회를 날린 것이다      

이것이 [스택오버플로우를 탐지하는 우리 코드](https://devblogs.microsoft.com/oldnewthing/20200609-00/?p=103847)가 회복할 것을 결정(각주 : 스택오버플로우 상태에서 벗어나서 프로그램을 계속해서 진행시킬 것을 결정 -> 스택이 사이즈가 다시 줄어듬)하는 경우 "_resetstkoflw()" 함수를 호출하는 이유이다. 스택오버플로우 상태를 초기화하는 것은 guard page였던 페이지에 대해 "PAGE_GUARD" 플래그를 복구시킴으로서 스택 사이즈 증가를 탐지하는 guard page의 역할을 복구시키는 것을 수반한다.            
            
이것이 모든 것이 정상적으로 동작할 때 벌어지는 일들입니다. 그러나 항상 정상적으로 동작하지는 않습니다.               
               
만약 하나의 스레드가 또 다른 스레드의 guard page에 접근하는 경우(아마도 버퍼 오버플로우 때문에, 혹은 단지 초기화되지 않은 포인터에 접근하여)도 마찬가지로 guard page exception을 발생시킬 것이다. 해당 guard page가 속한 스택을 소유한 스레드가 아닌, 다른 스레드에 의해 exception이 발생한 것이다. 만약 guard page exception이 현재 스레드의 스택에 속하지 않은 guard page에 대한 접근으로 인해 발생한 것이라고 판단되면, 디폴트 exception 핸들러는 이 exception을 무시한다. (각주 : 위에서 말했듯이 guard page에 접근시 "PAGE_GUARD" 플래그를 제거함으로서 guard page를 보통의 커밋된 페이지로 "바꾸고 난 후" "STATUS_GUARD_PAGE_VIOLATION" exception을 발생시킨다. )                       
```
이론적으로, 디폴트 exception 핸들러는 프로세스의 모든 스레드를 훓고, 그 주소가 어떤 스레드의 guard page에 속했는지를 알 수 있지만, 그렇게 하지 않습니다.      
이유 중 하나로, 이러한 동작히 너가 갑작스럽게 접근한 guard page가 속한 스택의 스레드(+ 동시에 그 guard page에 접근할 가능성이 있는 다른 모든 스레드들과의)와의 스레드간 통신을 요구하기 때문입니다.             
그러나 더 큰 이유는 그러한 상황(다른 스레드의 guard page에 접근하는 상황) 자체가 버그이기 때문입니다.      
그러한 (프로그램이 하지 말아야 할) 비정상적인 상황을 다루기 위해 시스템을 느리게한다는 것이 의미 없는 행동이기 때문입니다.
```
                                 
축하합니다. guard page가 사라졌기 때문에. 여러분의 스택은 corrupt되었습니다.               
<img class="guard page3" src="/assets/images/guard page3.PNG">               
          
해당 guard page(였던?)가 속한 스택의 스레드가 그 guard page였던 페이지까지 스택을 키우기전까지는, 마치 아무 문제 없는 것처럼 동작을 할 것입니다.        

<img class="guard page5" src="/assets/images/guard page5.PNG">               
            
(각주 : 그리고 해당 guard page(였던?)가 속한 스택의 스레드가 그 guard page였던 페이지에 접근하는 순간) 일반적인 상황에서는 guard page exception이 발생하고, 시스템은 원래하던대로 다음 예약된 페이지를 새로운 guard page로 승격했을겁니다.         
그러나 그 페이지는 guard page가 아니기 때문에(다른 스레드가 해당 페이지에 접근하여 commit된 페이지로 만들어버려), 아무런 행동 없이(정상적인 상황에서는 발생하였어야할 guard page exception이 발생하지 않고) 그냥 프로그램을 진행시킬 것입니다.     
          
모든 것이 완벽히 평상시와 같았던 것처럼 동작은 하고 있지만, 그 스택 포인터가 두 번째 새 페이지(첫번째 예약된 페이지)에 다다랐을 떄 잘못된 행동의 결과가 마침내 당신에게 영향을 미칩니다.                            
                                                   
<img class="guard page4" src="/assets/images/guard page4.PNG">                  
                   
현재 스택 포인터가 가리키는 페이지는 guard page가 아니므로, 특별한 스택 확장(각주 : "특별한 스택 확장"은 "PAGE_GUARD" 플래그를 제거함으로서 guard page를 보통의 커밋된 페이지로 바꾸고 난 후 "STATUS_GUARD_PAGE_VIOLATION" exception을 발생시키는 동작을 말함)을 하지 않습니다. 그리고 결국 스택오버플로우 예외를 발생시키고 죽게됩니다.                
            
그것이 유효하지 않은 메모리 접근의 슬픈 삶입니다. 빨리 드러나지 않고, 나중에서야 드러나는 미묘한 방법으로 너의 프로세스를 corrupt시킬 수 있습니다.               
        
다음 시간에는, [스택오버플로우 문제를 조사해보고, 이 guard page corruption이 발생했는지 않했는지를 탐지하는 방법을 알아 볼 것](https://devblogs.microsoft.com/oldnewthing/20220204-00/?p=106219)입니다.          