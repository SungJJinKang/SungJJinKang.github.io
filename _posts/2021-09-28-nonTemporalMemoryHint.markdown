---
layout: post
title:  "Write Combined Optimization - _mm_stream ( Non Temporal momory hint ( NT ) )"
date:   2021-09-28
categories: ComputerScience
---

얼마전 인터뷰에서 SIMD 중에 "_mm_stream_pd"에 대한 질문을 받았다. 여러 SIMD 명령어를 사용해보았지만 이건 처음 들어본 명령어다. 그래서 공부해보았고 새롭게 알게 된 내용에 대해 글을 써보겠다.       

인텔에서 해당 명렁어를 검색해보면 아래와 같은 설명이 나온다.     
```
Store 128-bits (composed of 2 packed double-precision (64-bit) floating-point elements) from a into memory using a non-temporal memory hint. mem_addr must be aligned on a 16-byte boundary or a general-protection exception may be generated.

"a non-temporal memory hint"을 사용하여 메모리에 128비트를 저장한다 
```

"a non-temporal memory hint" ??            
이게 도대체 뭔 말인가. 생전 처음 들어본다.          
그래서 스택 오버 플로우에 검색해보았고 이것이 무엇인지 말해보겠다.          

이 말은 "시간적 지역성"을 무시하라는 것이다.          
일반적으로 우리가 메모리에서 데이터를 가져올 때(LOAD)는 그 데이터가 이후 근시간내에 다시 사용될 것(시간적 지역성)이라고 예상하고 해당 데이터를 CPU 캐시에 복사해둔다.(이에 대해 이해가 가지 않는다면 [이 글](https://sungjjinkang.github.io/computerscience/2021/04/01/cachefriendly.html)을 읽어보기 바란다.)      

그런데 만약 프로그래머가 해당 데이터가 근시간 내에 사용될 가능성이 없다면 어떻게 되는가? 그러면 **이 근시간 내에 사용되지 않을 데이터를 캐시에 저장하기 위해 현재 캐시에 있는 어떤 데이터가 해당 캐시에서 쫒겨나 다음 단계의 캐시나 메모리로 옮겨져야한다. 근시간 내에 사용되지 않은 데이터를 캐시에 저장하기 위해 기존의 데이터를 쫒아낸다는 것이 얼마나 비효율적인 일인가.** 또한 캐시 라인 전체를 한꺼번에 쓰지 않는다면 쓰려는 데이터가 속한 캐시라인을 캐시로 우선 읽어온 후 캐시에 써야한다. ( 왜냐면 캐시 라인 단위로 캐시에 쓰는데 쓰려는 주소가 속한 캐시라인의 다른 데이터를 모르기 때문에 )                                  
그래서 "a non-temporal memory hint"란 쉽게 말해 **해당 데이터를 캐시에 저장하지 말라**는 것이다. 그냥 바로 메인 메모리 ( DRAM )에 쓰라는 것이다.                

이렇게 "a non-temporal memory hint"란 **LOAD시에는 메모리에서 읽어온 데이터를 캐시에 저장하지 않고 WRITE시에는 캐시에 쓰지 않고 곧바로 메모리에 쓰라**는 것이다.      

이러한 non temporal memory hint는 **큰 사이즈의 데이터**에 접근해야할 일이 생길 때 사용될 수 있는데 큰 사이즈의 데이터에 처음부터 끝까지 접근한다고 생각해보자. 사이즈가 매우 크기 때문에 **어느 정도 데이터에 다다르면 처음에 접근했던, 즉 초기의 접근해서 캐시에 캐싱되었던 데이터들은 축출되어버릴 것이다. 그럼 굳이 CPU 연산만 낭비해 캐싱을 할 필요가 없는 것**이다. 어차피 큰 사이즈의 데이터를 읽다가 중간에 앞에서 캐싱해두었던 데이터들이 캐시에서 축출되어버리니 말이다. 그래서 이러한 경우 데이터를 읽을 때 캐싱을 한 후 읽지 않고 그냥 메모리에서 곧 바로 읽는 non temporal memory hint를 주기도 한다.         

다만 주의해야할 것들이 몇가지 있는데 non-temporal store를 할 때 만약 저장하려는 목표 주소의 캐시라인이 이미 캐시에 올라와 있다면 이 명령어는 캐시에 올라와 있는 캐시를 메모리로 축출하는 동작을 한다. 이는 경우에 따라 심각한 성능 저하를 낳을 수 있다. 그래서 대개 non temporal hint 연산은 volatile한 타입의 메모리에만 사용한다.         
또한 non-temporal store 중인 주소에 load를 하게 되는 경우 당연히 최신 데이터는 non-temporal store가 끝나야 얻을 수 있으니 non-temporal store가 끝날 때까지 load 명령어는 stall된다.     

----------------------

이 명령어는 특히 **Write - Combined 버퍼**를 활용하는데 도움을 준다.       
자자 우선 Write Combine 버퍼가 무엇인지부터 알아보자.       

Write Combined 버퍼는 Write Combined 최적화에 사용되는데 쉽게 말하면 **여러 Write 동작을 모아서 한꺼번에 수행한다는 것이다.** 쓰기 동작을 할 여러 데이터를 Write Combined 버퍼에 한꺼번에 모아두었다가 한번에 쓴다는 개념이다.        
이렇게 Write할 데이터를 모아서 **Burst모드에서 한꺼번에 쓰기 동작을 수행**할 있다. **Burst 모드 동안에는 평소에 한번에 보낼 수 있는 데이터 양의 몇배가 되는 데이터를 전송할 수 있게 된다.**                 

대표적인 예로는 **CPU가 캐시에 쓰기 동작을 하려고 할때 주소가 속한 캐시라인이 L1 캐시에 올라와 있지 않으면 해당 캐시라인을 L1 캐시까지 가지고 와야한다. ( Write - Allocate )** ( 여기서 헷갈릴 수 있는데 캐시에 데이터를 쓰려고할 때 캐시 라인 전체를 한꺼번에 쓰지 않는 경우, 즉 캐시 라인의 일부분만 캐시에 쓰려고 하는 경우 당연히 CPU는 쓰려는 위치의 데이터만 알고 캐시 라인내 다른 데이터는 알지 못하니 쓰기 전 우선 해당 캐시 라인을 캐시로 가져와야한다. 이를 Write - Allocate라고 한다. )           
( 이후에 설명하겠지만 캐시 라인의 일부만 쓰는게 아니라 캐시 라인 전체에 대해 쓰기 동작을 수행한다고 하면 굳이 L1 캐시로 캐시라인을 가져올 필요가 없을 것이다. )           
그럼 CPU는 캐시에 캐시 라인을 가져오는 동안 뭘해야하나?? 그냥 가만히 기다리고 있나???       
아니다, CPU는 **쓰려고 하는 데이터를 CPU 칩의 Write Combined 버퍼에 임시로 저장해두고 write - Allocate에 따라 쓰려는 위치가 속하는 캐시라인을 L1캐시로 가져올 때 까지 기다리지 않고 다음 명령어를 수행하고 있는다.** 그 후 캐시를 쓸 수 있는 상태가 되면 이 Write Combined 버퍼를 L1캐시로 복사한다.        

**당연한 얘기지만 쓰려는 주소의 캐시라인이 이미 L1캐시에 올라와있다면 이 Write Combined 최적화는 적용되지 않는다.**        

쓰려고 하는 주소의 캐시라인이 메모리 계층상 더 멀리 있을 수록 Write Combine 기법으로 얻어지는 성능 향상은 더 커진다.     
( 왜냐면 캐시 라인을 가져오는 것을 완료하면 Write Combined 버퍼는 바로 비워지기 때문에 캐시 라인을 더 늦게 가져올 수록 이 Write Combine 버퍼을 활용할 여지가 더 커지기 때문이다. )                             

만약에 이 Write Combined 버퍼에 저장을 한 명령어의 다음 명령어도 이전 명령어가 쓰려던 위치와 같은 캐시라인에 속한 위치라면 다음 명령어도 같은 Write Combined 버퍼에 쓰일 것이다. 참고로 Write Combined 버퍼의 사이즈는 캐시 라인 사이즈와 같다. ( 쓰기 동작을 수행할 때 쓰려는 데이터는 여러 캐시 라인에 걸치면 안된다. 하나의 캐시라인에만 속해야한다. )               

**만약에 같은 캐시라인에 쓰기 동작을 계속 수행하다가 해당 캐시라인에 대한 쓰기를 전부 수행해서 Write Combine 버퍼를 어떤 특정 캐시라인에 대한 쓰기로 64바이트 전부 채우면 어떻게 될까??? 나이스!! 캐시에 원본 캐시라인 데이터를 가져오지 않고도 그냥 L1 캐시에 바로 쓸 수 있다.** 위에서 말했듯이 캐시 라인의 일부분만 쓰려고 하는 경우에만 해당 캐시라인의 원본 데이터를 캐시로 가져온다.              

---------------------

근데 사실 위에서 말한 것과 같이 캐싱을 사용하는 메모리 연산에서 Write Combined 버퍼가 가져오는 성능 향상은 그렇게까지 크지는 않다.         
**Write Combined 버퍼가 진가를 발휘하는 곳**은 volatile 데이터인 경우이다. volatile에 대해 간단히 설명하면 **캐싱을 하지 않고 메모리 ( DRAM )에 바로 쓰라는** 것이다.           
**캐시의 경우 그래도 속도가 빠르니 Write Combined 버퍼의 효과가 크게 두드러지지 않는데 DRAM에 쓰기 동작을 수행할 때 Write Combined 버퍼는 엄청난 성능 향상을 불러온다.**          
DRAM에 데이터를 쓰려면 반드시 메모리 버스를 통해야 하는데 이**메모리 버스는 여러 코어가 공유하고 있고 DMA도 메모리 버스를 사용하기 때문에 메모리 버스를 자주 점유 ( 버스 마스터링 )하는 것은 성능상 매우 좋지 않다.** 그래서 데이터를 **모아두었다가** ( CPU의 Write Combined 버퍼에 ) **한번에 쓰는 것**이 **성능향상에 큰 도움**이 된다.            
이렇게 캐싱을 활용하지 않는 메모리 연산으로는 위에서 배운 것 처럼 **"non-temporal memory hint"** 연산이 경우가 대표적이다.         
또한 **메모리 맵 IO**가 또 다른 예인데 메모리 맵 IO의 경우 알다 싶이 CPU 입장에서는 일반적인 메모리 연산과 명령어 코드가 똑같고, 디바이스가 최신의 데이터를 보기 위해 캐싱을 하면 안된다는 특징을 가지고 있다. ( 바로 DRAM에 써야 디바이스가 최신의 데이터를 읽어갈 수 있다. )      


가장 중요한 것은 **완전히 채워지지 않는 Write Combined 버퍼를 flush 하는 것은 매우 매우 매우 최악의 행동**이다.  

그러니 CPU마다 가지고 있는 Write - Combine 버퍼의 개수에 맞추어서, 만약 Write Combine 버퍼의 개수가 4개라면 이 Write - Combine 기법의 이점을 최대한 활용하기 위해 4개보다 많은 캐시 라인을 연속적으로 건들면 안된다. 왜냐면 4개보다 많은 캐시라인을 건드는 순간부터 현재 flush 되지 않는 다른 캐시라인의 Write Combine 버퍼를 flush하게 되고 이는 매우 비효율적인 동작이기 때문이다.       
그러니 절대로 **보유중인 Write - Combine 버퍼의 개수보다 많은 캐시라인은 동시에 건드려서 기존 버퍼가 다 차기 전에 flush 해버리는 치명적인 성능 하락을 만들지마라.**                         

아래 사진은 Write Combined 버퍼를 완전히 채우지 않고 flush 했을 경우의 성능을 비교한 사진이다.       
64바이트의 경우가 Write Combined 버퍼를 완전히 채운 경우이다.        

![write_combine](https://user-images.githubusercontent.com/33873804/134978510-deaee18d-f7ab-4350-a8d0-13df12d3aa6c.png)         

이러한 Write Combined 버퍼는 L1캐시로 캐시라인을 가져오는데도, L1과 L2 캐시간 캐시라인 전송, DRAM 전송 등 여러군데서 활용되기 때문에 꽉 차지 않은 Write Combined 버퍼를 flush하는 것은 성능상 매우 매우 좋지 않다.      


자자 위의 내용들을 종합해서 한가지 더 알려주겠다.         
만약에 **Write Combined 타입의 메모리를 읽으려고하면 무슨일이 벌어질까?**               
우선 Write Combined 타입의 메모리를 읽는 것은 캐싱되지 않는다. 그리고 Write Combined 타입을 읽으려고 하면 **존재하는 write combined 버퍼를 모두 flush를 해야한다**. ( 당연히 write combined 버퍼를 flush해야 신선한(?), 최신의 데이터를 읽을 수 있다. ) 여기서 Write Combined 버퍼를 모두 flush 한다는 것은 높은 확률로 다 차지도 않은 write combined 버퍼를 flush 해버린다는 것이다. 이는 **위에서 말한대로 매우 매우 비효율적**이다. ( 물론 캐시되지 않은 데이터를 읽는 동작 자체가 느리기는 하지만 IO를 위한 데이터들은 캐싱을 하지 않고 메모리에 쓰니 캐싱을 하지 않는 상황을 가정하자. )              

그러니 특별한 이유가 없다면 **절대로 write-combining 메모리를 읽지마라.** 특히 렌더링 관점에서 작성 중인 constant buffers, vertex buffers, index buffers 는 절대 읽지마라. 이 버퍼들은 write combined 타입의 메모리 ( 이 버퍼들은 GPU에서 읽어가야하므로 캐싱을 하지 않고 바로 메모리에 쓰기 동작을 하는 volatile해야 하는 유형의 데이터들이다 )이기 때문에 읽으려는 것은 최악이다..         

렌더링에서 활용되는 write-combined 버퍼에 대해서는 [이 글](https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/)을 읽어보기 바란다.       

또한 write-combined 최적화의 경우 메모리 ordering을 보장하지 않는다. write-combined store -> read -> write-combined store에서 read가 앞의 store가 보인다는 것을 보장하지 않는다는 것이다. 그래서 write - combined 최적화는 대량의 데이터를 빠르게 보내고자 할 때 사용되고 대량의 데이터를 다 전송하는 중간에 read를 하는 것이 필요없는 상황에서 사용되어야한다.         


근데 [Write - Combined 기법이 항상 빠르지는 않다는 글](https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/)도 있다... 고려할 경우의 수가 너무 많다. 궁금하다면 한번 읽어보아라.                         

references : [https://stackoverflow.com/a/37092/7138899](https://stackoverflow.com/a/37092/7138899), [https://mechanical-sympathy.blogspot.com/2011/07/write-combining.html](https://mechanical-sympathy.blogspot.com/2011/07/write-combining.html), [https://sites.utexas.edu/jdm4372/2018/01/01/notes-on-non-temporal-aka-streaming-stores/](https://sites.utexas.edu/jdm4372/2018/01/01/notes-on-non-temporal-aka-streaming-stores/), [https://stackoverflow.com/questions/14106477/how-do-non-temporal-instructions-work](https://stackoverflow.com/questions/14106477/how-do-non-temporal-instructions-work), [https://vgatherps.github.io/2018-09-02-nontemporal/](https://vgatherps.github.io/2018-09-02-nontemporal/),  [http://www.nic.uoregon.edu/~khuck/ts/acumem-report/manual_html/ch05s03.html](http://www.nic.uoregon.edu/~khuck/ts/acumem-report/manual_html/ch05s03.html), [https://stackoverflow.com/questions/49959963/where-is-the-write-combining-buffer-located-x86](https://stackoverflow.com/questions/49959963/where-is-the-write-combining-buffer-located-x86)                               