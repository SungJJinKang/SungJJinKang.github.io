---
layout: post
title:  "매우 사소하지만 성능에 영향이 큰 최적화 ( std::vector )"
date:   2021-01-28
categories: C++
---

현재 Doom 프로젝트에서 기본 밑바탕을 차지하는 Entity나 Component System을 만들고 있는 데 사소하지만 게임의 규모가 커지면 성능에 많은 영향을 미치는 최적화 방법을 하나 소개하고자 한다.   

Entity나 Component System을 만들다보면 당연히 내부적으로 vector을 통해 현재 Scene의 스폰되어 있는 Entity나 그 Entity들의 Component들을 관리할 것이다.   

그러다 특정 Entity를 Scene에서 삭제하거나 특정 Component를 Entity에서 뺄 때 아무런 생각없이 erase를 사용하는 경우가 있을 꺼 같다.   
근데 erase가 내부적으로 어떻게 돌아가는지 안다면 이러한 코드는 게임에 따라 엄청난 오버헤드를 생성할 수 있다.   
그 이유는 무엇일까?   

이유를 알기 위해서는 vector::erase가 어떻게 작동하는지 알아햐 한다.   
erase는 vector의 element 중 특정 element를 제거하는 함수이다.      
근데 여기서 중요한건 그 특정 element를 제거하면 element들 중간에 삭제된 element는 빈공간으로 남게 되고 erase함수는 자동적으로 그 삭제된 element 뒤의 element들을 reallocation해서 앞으로 한칸씩 이동시켜야 되서 만약에 move 생성자가 잘 구현되어 있지 않은 경우에는 큰 오버헤드를 만들어버릴 수 있다.   

그럼 두가지 방법이 있다.   
vector을 vector<Entity*> 이런식으로 포인터로 해서 삭제할 element는 그냥 메모리를 해제하고 nullptr을 그 자리에 넣어버리는 것이다.   
이 방법 나쁘지 않다. Entity vector가 필요할 때 그냥 nullptr 체크를 하면 빈 element는 걸러낼 수 있다.   
그치만 성능이 생명인 게임에서 이러한 null 체크 또한 쌓이면 큰 오버헤드가 될 수 있다.   

그래서 생각해낸게 그냥 vector의 맨마지막 element를 삭제할 element 자리에 move 해주는 것이다.   
그럼 중간에 빈 element도 생기지 않고 깔끔하게 이 문제를 해결할 수 있다.   
물론 vector의 element들의 순서를 유지해야한다면 그냥 첫번째 방법을 사용해도 된다.   

아래의 깃허브 주소를 가보면 필자가 구현한 방법을 볼 수 있다.   
[Github Repo](https://github.com/SungJJinKang/Voxel_Doom3_From_Scratch/blob/main/Doom3/Source/Core/CoreComponent/World.cpp)

