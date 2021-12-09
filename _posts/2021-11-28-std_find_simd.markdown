---
layout: post
title:  "SIMD 명령어를 활용한 std::find"
date:   2021-11-28
categories: ComputerScience GameEngine
---

구현 중인 게임 엔진에서는 많은 개수의 item을 가진 std::vector 중 중간에 어떤 아이템을 삭제해주기 위해 간단한 트릭을 사용한다.       
[std::vector_swap_popback](https://github.com/SungJJinKang/vector_swap_popback)이 그것인데 간단히 설명하면 std::vector에서 중간에 어떤 아이템을 삭제하고자 하면 해당 아이템을 삭제하고 그 뒤에 있는 아이템들을 모두 한칸씩 앞으로 옮기는 동작이 필요하다.          
std::vector의 아이템 개수가 많을 수록 성능 저하는 더 커진다.         
( 나중에 보니 언리얼 엔진에도 똑같은 기능을 하는 함수가 있어서 내가 고안한 것이 틀리지는 않았구나 생각했다. )           

근데 이게 아이템이 워낙 개수가 많아지다보니 삭제할 **아이템의 index를 찾는데도 많은 시간이 걸렸다.**                      
그렇다고 다른 해쉬테이블이나 tree 구조의 컨테이너를 쓰기에는 해당 리스트를 순회하는 동작이 있어서 다른 컨테이너를 사용할 시 순회시 성능 저하가 걱정되었다.                
그래서 최적화를 해야겠다고 생각했다.             

어떻게 해야할지 고민을 하다가 **SIMD 명령어**를 활용하기로 했다.          
std::vector에 들어간 아이템 타입이 숫자나 포인터인 경우 단순 비트 비교만으로 비교가 가능하니 SIMD를 활용하면 성능을 극대화할 수 있다고 생각했다.          
간단한 SIMD 함수를 통해서 구현을 했다.           
그리고 SIMD 레지스터 사이즈에 alignment 해야하는 문제도 다 해결해준다. 이 함수를 사용하는 사람이 alignment 문제를 신경쓰지 않아도 된다는 것이다.             

[std_find_simd Github](https://github.com/SungJJinKang/std_find_simd)          

 
성능을 비교해보니 놀라울 정도로 빨랐다.         
거의 2배에 가까운 성능 향상을 얻을 수 있었다.       
std::find가 simd 명령어를 사용하지 않고 단순 비교를 하니 std::find에 비해서는 놀랍도록 빨랐다.          

![20211128031529](https://user-images.githubusercontent.com/33873804/143701373-1c8aafbe-6131-4538-9d60-5432b84cd87c.png)        


사실 내가 구현한 코드는 잘 믿지를 않아서 항상 레딧이나 트위터에 올려서 피드백을 받아보는데 이번에도 피드백을 받아보았다.           
다행히 구현이 나쁘지 않았나 보나. 레딧에서 많은분들이 피드백을 주시고 좋아해주셨다.             
글을 올린지 거의 12시간만에 레딧 cpp 커뮤니티에서 75 좋아요를 받았다.             

![20211128192511](https://user-images.githubusercontent.com/33873804/143764105-2868a8ef-b7d9-4fa5-a9b8-c2b53de45dd2.png)


           
