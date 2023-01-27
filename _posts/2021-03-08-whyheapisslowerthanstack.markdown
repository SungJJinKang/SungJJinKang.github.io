---
layout: post
title:  "왜 heap영역은 stack보다 느릴까?"
date:   2021-03-07
tags: [ComputerScience]
---

stack영역은 해당 프로그램에 배당된 stack 사이즈가 얼만큼 배당되어 있다.(물론 이것도 나중에 늘릴 수 있다.)
그럼 어떠한 scope에 진입하는 순간 그 scope에서 사용되는 메모리 사이즈만큼 stack pointer을 미리 이동시켜둔다. ( stack의 사이즈는 변함이 없고 현재 총 사이즈 중 어디까지 할당했는 지를 stack pointer로 표시하는 것이다, scopre를 빠져나가면 다시 stack pointer가 원래대로 돌아감 )      
그리고 stack pointer를 기준으로 얼만큼의 offset(해당 scope 기준)으로 스택 변수에 접근하는 것이다.
stack pointer + offset으로 바로 데이터에 접근할 수 있는 것이다.
심지어 이 scope의 local 변수들의 총 사이즈와 각 변수의 offset 또한 컴파일 타임에 다 결정되어서 매우 매우 빠르게 현재 stack pointer를 옮기고 offset을 더해서 데이터에 접근하는 것이다.
그리고 스택영역은 큰 사이즈를 한꺼번에 배정받기 때문에 data들이 서로 붙어있다. 이는 cpu cache hit 가능성이 높다는 의미이기도 하다.

반면 heap은 어떤가??
heap은 allocation부터 느리다. heap의 allocation은 실시간으로 필요한 그 순간 allocate를 하는 데 이 heap allocate가 프로그램단에서 이루어 지는 게 아니라 os에 해당 사이즈만큼 heap영역을 할당해달라고 요청해야한다.
그럼 os는 요청된 사이즈만큼의 힙영역을 allocate하기 위해 빈 공간을 찾아다닌다 (물론 free page list가 존재한다 ).
free page list에 그 사이즈만큼 있으면 다행인데 없는 경우도 있다.
그럼 또 os는 free page list를 합치면서 해당 size를 확보해야한다. 이 또한 엄청 느리다 ( 흔히 memory fragmentation 문제라고 한다).
heap이 실시간으로 os로부터 memory를 할당받기 때문에 heap으로 할당받은 데이터들은 서로 떨어져 있을 가능성이 높다. 이는 cpu cache miss 가능성을 높인다.    


여기서 free block(page) list를 소개하자면, 프로그램이 heap을 쓰고 나서 free를 하면 바로 os에 돌려주지 않는다. 왜냐면 만약 해당 heap 영역을(page)를 바로 os에 돌려줘버리면 OS에는 수 많은 fragmentation(구멍)이 생길 것이다. 이건 결국 나중에 가면 큰 문제가 될 가능성이 높다. 그래서 이 반환될 블록을 바로 os에 반환하지 않고 free block list(circular list)에 보관해 둔다.       
이 free block list는 만약 인접한 block이 연결되면 그 블록을 합치기도 하고 프로그램이 heap영역을 다시 요청하면 이 free block list를 먼저 검색해보고 만약 맞는 사이즈의 block이 있다면 이 free block list에서 프로그램에 메모리를 준다.     
즉 힙 메모리 할당을 요청하면 이 free block list를 먼저 탐색해보고 그래도 못찾으면 os에 heap 메모리를 할당을 요청한다.        
이런 fragmentation을 줄이기 위한 전략은 os마다 다양하다.              



stack, heap에 대한 이해를 높여주는 글들 :     
[https://stackoverflow.com/questions/24057331/is-accessing-data-in-the-heap-faster-than-from-the-stack](https://stackoverflow.com/questions/24057331/is-accessing-data-in-the-heap-faster-than-from-the-stack)     
[https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap)       
[https://stackoverflow.com/questions/1119134/how-do-malloc-and-free-work?rq=1](https://stackoverflow.com/questions/1119134/how-do-malloc-and-free-work?rq=1)       
[https://stackoverflow.com/questions/2308751/what-is-a-memory-heap](https://stackoverflow.com/questions/2308751/what-is-a-memory-heap)       
[https://stackoverflow.com/questions/18431261/how-does-x86-paging-work](https://stackoverflow.com/questions/18431261/how-does-x86-paging-work)       