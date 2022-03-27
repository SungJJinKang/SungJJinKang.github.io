---
layout: post
title:  "자체 게임 엔진에서 게임 스레드, 렌더 스레드 분리하기."
date:   2022-03-27
categories: ComputerScience ComputerGraphics
---

[얼마 전 작성한 글](https://sungjjinkang.github.io/computerscience/computergraphics/ue4/unrealengine4/2022/03/27/ue4_rendering_thread.html)에서 어떻게 언리얼 엔진4가 게임 스레드에서 렌더스레드로 데이터를 안전하게 전송하는지에 대해 다루어보았다.               

필자는 이를 [내가 개발 중인 게임 엔진](https://github.com/SungJJinKang/DoomsEngine)에 접목해볼 예정이다.           

-------------------------------------------------           
             
사실 이렇게 게임 스레드와 렌더 스레드를 분리한다고 해도 얼마나 성능이 향상될지는 알 수 없다.                
언리얼 엔진4 정도의 수준까지 구현을 하는 것은 당연히 불가능하기 때문에 **오히려 싱글 스레드에서 모두 연산을 하는 것보다 더 느릴수도 있다.**                    
그렇지만 **언리얼 엔진4의 렌더링이 어떻게 동작하는지를 공부할 겸**해서 [내 게임 엔진](https://github.com/SungJJinKang/DoomsEngine)에서 구현을 해보면 언리얼 엔진을 이해하는데 큰 도움이 될 것 같아 이러한 결정을 하게 되었다.          
            
