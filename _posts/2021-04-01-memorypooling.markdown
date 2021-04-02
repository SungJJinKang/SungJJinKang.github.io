---
layout: post
title:  "메모리 풀링(작성 중)"
date:   2021-04-01
categories: ComputerScience
---

게임을 계속 개발하다 모니 메모리 풀링이 필요하다는 것을 많이 느꼈다.    
게임이다 보니 당연히 매 프레임마다 모든 Entity, Component들을 업데이트 해주어야하는 데 이 과정에서 그렇게 큰 부하가 필요하지 않는 데도 단지 모든 Entity, Component들을 iterate하기 때문에 속도가 매우 느렸다.   
원인을 생각해보니 Entity, Compnent들이 파편화되어 있는 것이 문제라고 생각했다.      
새로운 Entity, Component가 필요할 때 마다 new로 매번 동적할당을 해주는 것이 문제였다. 이렇게 매번 동적할당을 해주다 보니 각 Entity, Component들이 파편화 되었고 이 클래스들의 Object들을 iterate할 때 cache hitting이 떨어질 수 밖에 없었다.    

Cache line이 64바이트이니
```c++
class Entity
{

};

Entity* CreateNewEntity()
{
   Entity* newEntity = new Entity();
   return newEntity;
}
```


게임과 같이 메모리 사용이 많은 메모리 파편화 관리는 필수적이다.   
대다수의 상용엔진들도 메모리 풀링 기법을 활용하고 특히 EA에서 개발한 EASTL은 메모리 할당, 해제도 자신들이 직접 만든 라이브러리를 통해 한다.    
그리고 메모리 풀링 기법을 잘만 활용한다면 interate를 하는 Entity나 Component같은 클래스들의 object에 접근할 때 cache hitting도 높일 수 있다.      
그래서 필자도 게임 개발 프로젝트를 하며 직접 메모리 풀링을 만들기로 결정하였다.     

한 페이지내에 블록이 들어갈 수 있게 블록의 사이즈는 4KB(4096Byte) 내로 제한하였다.  