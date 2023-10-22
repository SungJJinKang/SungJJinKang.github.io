---
layout: post
title:  "Write Combine(Write Combined 버퍼) - _mm_stream ( Non Temporal momory hint ( NT ) )"
date:   2021-09-28
tags: [ComputerScience, Recommend]
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
일반적으로 우리가 메모리에서 데이터를 가져올 때(LOAD)는 그 데이터가 이후 근시간내에 다시 사용될 것(시간적 지역성)이라고 예상하고 해당 데이터를 CPU 캐시에 복사해둔다.(이에 대해 이해가 가지 않는다면 [이 글](https://sungjjinkang.github.io/cachecoherency)을 읽어보기 바란다.)      

그런데 만약 프로그래머가 해당 데이터가 근시간 내에 사용될 가능성이 없다면 어떻게 되는가? 그러면 **이 근시간 내에 사용되지 않을 데이터를 캐시에 저장하기 위해 현재 캐시에 있는 어떤 데이터가 해당 캐시에서 쫒겨나 다음 단계의 캐시나 메모리로 옮겨져야한다. 근시간 내에 사용되지 않은 데이터를 캐시에 저장하기 위해 기존의 데이터를 쫒아낸다는 것이 얼마나 비효율적인 일인가.** 또한 캐시 라인 전체를 한꺼번에 쓰지 않는다면 쓰려는 데이터가 속한 캐시라인을 캐시로 우선 읽어온 후 캐시에 써야한다. ( 왜냐면 캐시 라인 단위로 캐시에 쓰는데 쓰려는 주소가 속한 캐시라인의 다른 데이터를 모르기 때문에 )                                  
그래서 "a non-temporal memory hint"란 쉽게 말해 **해당 데이터를 캐시에 저장하지 말라**는 것이다. 그냥 바로 메인 메모리 ( DRAM )에 쓰라는 것이다.                

이렇게 "a non-temporal memory hint"란 **LOAD시에는 메모리에서 읽어온 데이터를 캐시에 저장하지 않고 WRITE시에는 캐시에 쓰지 않고 곧바로 메모리에 쓰라**는 것이다.      

이러한 non temporal memory hint는 **큰 사이즈의 데이터**에 접근해야할 일이 생길 때 사용될 수 있는데 큰 사이즈의 데이터에 처음부터 끝까지 접근한다고 생각해보자. 사이즈가 매우 크기 때문에 **어느 정도 데이터에 다다르면 처음에 접근했던, 즉 초기의 접근해서 캐시에 캐싱되었던 데이터들은 축출되어버릴 것이다. 그럼 굳이 CPU 연산만 낭비해 캐싱을 할 필요가 없는 것**이다. 어차피 큰 사이즈의 데이터를 읽다가 중간에 앞에서 캐싱해두었던 데이터들이 캐시에서 축출되어버리니 말이다. 그래서 이러한 경우 데이터를 읽을 때 캐싱을 한 후 읽지 않고 그냥 메모리에서 곧 바로 읽는 non temporal memory hint를 주기도 한다.         

밑에서 설명하겠지만 **이 "a non-temporal memory hint"는 대표적으로 Write Combined 타입의 페이지에 메모리 쓰기 동작을 수행할 때 사용된다.**                

다만 주의해야할 것들이 몇가지 있는데 non-temporal store를 할 때 만약 저장하려는 목표 주소의 캐시라인이 이미 캐시에 올라와 있다면 이 명령어는 캐시에 올라와 있는 캐시를 메모리로 축출하는 동작을 한다. 이는 경우에 따라 심각한 성능 저하를 낳을 수 있다. 그래서 대개 non temporal hint 연산은 Write - Combined 타입의 메모리에만 사용한다.         

----------------------

이 명령어는 특히 **Write - Combined 버퍼** ( Fill 버퍼 )를 활용하는데 도움을 준다.       
자자 우선 Write Combine 버퍼가 무엇인지부터 알아보자.       

Write Combined 버퍼는 Write Combined 최적화에 사용되는데 쉽게 말하면 **여러 Write 동작을 모아서 한꺼번에 수행한다는 것이다.** 쓰기 동작을 할 여러 데이터를 Write Combined 버퍼에 모아두었다가 한번에 쓴다는 개념이다.        

메모리로의 버스 트랜젝션을 한번 개통하는데 ( 버스 마스터링 ) 시간이 많이 걸리지만, 한번 개통이 되면 워드 사이즈 ( 64bit CPU - 8바이트, 32bit CPU - 4 바이트 )씩 연속적으로 빠르게 데이터 전송이 가능하다. ( 메모리로의 버스 대역폭이 8바이트이고, CPU 코어와 캐시 간의 버스 대역폭은 훨씬 넓다. 대개 해당 CPU가 지원하는 SIMD 레지스터 사이즈만큼의 대역폭을 가진다. [레퍼런스1](https://stackoverflow.com/a/47512806/7138899), [레퍼런스2](https://electronics.stackexchange.com/a/329955) )                          
버스 트랜젝션을 개통하는데 시간이 많이 걸리기 때문에 버스 트랜젝션을 할 때마다 최대한 버스 대역폭을 꽉꽉채워서 데이터를 전송하는 것이 성능상 유리하다.       

한번 버스 트랜젝션을 수행할 때마다 워드 사이즈씩 보낼 수 있는데, **버스트 모드 전송에서는 캐시 라인 사이즈만큼의 데이터를 한번의 버스 트랜젝션으로 수행할 수 있다.(정확히는 버스를 마스터링 후 캐시 라인 사이즈만큼의 데이터 전송을 끝낼 때까지 마스터링한 것을 Releast하지 않음)**      
그래서 **Write Combined 버퍼 ( 사이즈가 캐시라인 사이즈와 같다 )를 완전히 채워서 캐시 라인 사이즈 ( 대개 64바이트 )의 데이터를 전송하는 경우 버스트 모드 ( 한번 버스 개통하면 보낼 데이터 모두 전송하기까지 버스 release 안함 )으로 처리가 가능하지만, 만약 Write Combined 버퍼를 완전히 채우지 않은 상태로 버스 트랜젝션을 수행하는 경우 8번 ( 캐시 라인 사이즈/워드 사이즈, 64비트 CPU의 경우 한번의 버스 트랜젝션에 64비트 즉 워드 사이즈를 전송할 수 있다. ) 혹은 4번 ( 캐시 라인 사이즈/워드 사이즈 )의 버스 트랜젝션 ( 일반 모드로 한번의 버스 트랜젝션 당 워드 사이즈씩만 처리 가능 ) 을 통해서 처리해야한다.** ( 물론 한번의 버스트 트랜젝션이 한번의 워드 사이즈 일반 트랜젝션보다 빠르지는 않지만, 8번의 일반 트랜젝션보다는 월등히 빠르다. )                 
다만 위에서 말했듯이 **데이터들이 같은 캐시라인에 속해있어야 한번에 보낼 수 있다.** ( 같은 캐시 라인에 속하는 경우 Write Combined 버퍼에서 쓸 데이터가 임시로 저장된다. )                                    
                
버스트 모드는 CPU나 DMA가 버스를 통해 버스 트랜젝션을 발생하는 방식 중 하나이다.        
좀 자세히 설명하자면 버스트 모드에서는 CPU나 DMA 컨트롤러는 한번 버스를 개통하면 전송할 데이터를 모두 전송할 때까지 버스를 release하지 않는다.        
반면 사이클 훔치기 모드 ( Cycle Stealing Mode )에서는 DMA 컨트롤러가 CPU가 점유 중인 버스를 release하게하고 DMA를 마스터링한다. 단 그 시간이 짧다. 잠깐 CPU의 버스 컨트롤을 뺏아서 사용하는 것이다.       
Transparent 모드에서는 CPU가 버스를 필요로하지 않을 때만 버스를 마스터링한다.         


이러한 Write Combined 버퍼를 활용한 대표적인 예로는 **CPU가 캐시에 쓰기 동작을 하려고 할때 주소가 속한 캐시라인이 L1 캐시에 올라와 있지 않으면 해당 캐시라인을 L1 캐시까지 가지고 와야한다. ( Write - Allocate )** ( 여기서 헷갈릴 수 있는데 캐시에 데이터를 쓰려고할 때 캐시 라인 전체를 한꺼번에 쓰지 않는 경우, 즉 캐시 라인의 일부분만 캐시에 쓰려고 하는 경우 당연히 CPU는 쓰려는 위치의 데이터만 알고 캐시 라인내 다른 데이터는 알지 못하니 쓰기 전 우선 해당 캐시 라인을 캐시로 가져와야한다. 이를 Write - Allocate라고 한다. )           
( 이후에 설명하겠지만 캐시 라인의 일부만 쓰는게 아니라 캐시 라인 전체에 대해 쓰기 동작을 수행한다고 하면 굳이 L1 캐시로 캐시라인을 가져올 필요가 없을 것이다. )           
그럼 CPU는 캐시에 캐시 라인을 가져오는 동안 뭘해야하나?? 그냥 가만히 기다리고 있나???       
아니다, CPU는 **쓰려고 하는 데이터를 CPU 칩의 Write Combined 버퍼에 임시로 저장해두고 Write - Allocate에 따라 쓰려는 위치가 속하는 캐시라인을 L1캐시로 가져올 때 까지 기다리지 않고 다음 명령어를 수행하고 있는다.** 그 후 캐시를 쓸 수 있는 상태가 되면 이 Write Combined 버퍼를 L1캐시로 복사한다.        
          
쓰려고 하는 주소의 캐시라인이 메모리 계층상 더 멀리 있을 수록 Write Combine 기법으로 얻어지는 성능 향상은 더 커진다.     
( 왜냐면 캐시 라인을 가져오는 것을 완료하면 Write Combined 버퍼는 바로 비워지기 때문에 캐시 라인을 더 늦게 가져올 수록 이 Write Combine 버퍼을 활용할 여지가 더 커지기 때문이다. )                              
만약에 이 Write Combined 버퍼에 저장을 한 명령어의 다음 명령어도 이전 명령어가 쓰려던 위치와 같은 캐시라인에 속한 위치라면 다음 명령어도 같은 Write Combined 버퍼에 쓰일 것이다. 참고로 Write Combined 버퍼의 사이즈는 캐시 라인 사이즈와 같다. ( 쓰기 동작을 수행할 때 쓰려는 데이터는 여러 캐시 라인에 걸치면 안된다. 하나의 캐시라인에만 속해야한다. )                             
volatile과 Write-Combined 타입은 다른 것이다. volatile은 레지스터에 데이터를 임시로 저장하는 최적화를 수행하지 말라는 것이다. 즉 캐시에 써도 되고 메모리에 써도 된다.         
반면 non-temporal 쓰기, write-combined 타입의 페이지는 메인 메모리, DRAM에 쓰라는 것이다. 캐시에 쓰면 안된다. ( [참고](https://github.com/MicrosoftDocs/windows-driver-docs-ddi/blob/staging/wdk-ddi-src/content/ntifs/ns-ntifs-_memory_basic_information.md) )                

**만약에 같은 캐시라인에 쓰기 동작을 계속 수행하다가 해당 캐시라인에 대한 쓰기를 전부 수행해서 Write Combine 버퍼를 어떤 특정 캐시라인에 대한 쓰기로 64바이트 전부 채우면 어떻게 될까??? 나이스!! 캐시에 원본 캐시라인 데이터를 가져오지 않고도 그냥 L1 캐시에 바로 쓸 수 있다.** 위에서 말했듯이 캐시 라인의 일부분만 쓰려고 하는 경우에만 해당 캐시라인의 원본 데이터를 캐시로 가져온다.              

---------------------

근데 사실 위에서 말한 것과 같이 캐싱을 사용하는 메모리 연산에서 Write Combined 버퍼가 가져오는 성능 향상은 그렇게까지 크지는 않다.         
**Write Combined 버퍼가 진가를 발휘하는 곳**은 접근하는 메모리 영역이 Write Combined 타입의 페이지에 속한 경우이다. ( Write Combined 버퍼와 Write Combined 타입의 페이지는 구분하여야 한다. )                                        
          
**Write Combined 타입의 페이지**는 **메모리 영역 혹은 페이지에 붙는 일종의 flag** ( 페이지 테이블에 있는 페이지에 붙거나, MTRR - memory type range register를 통해 관리 )의 일종으로 **이 영역, 페이지에 대한 쓰기 동작은 Write Combined 버퍼에 임시로 저장**된다. 만약 **Write Combined 버퍼가 일부만 차있다면 메모리로의 쓰기는 지연**된다. 그렇기 때문에 메모리 in ordering이 보장되지 않는다. 다만 SFENCE 혹은 MFENCE 명령어, CPUID 실행, 캐싱이 되지 않는 메모리에 읽기 쓰기, 인터럽트 발생, LOCK 명령어가 발생하는 경우 Write Combined 버퍼가 일부만 찼더라도 메모리로 flush된다. 이러한 유형의 메모리 타입은 비디오 프레임 버퍼와 같은 데이터에 사용하기 적합한데 메모리 ordering이 중요하지 않고 캐싱이 되면 안되기 때문이다. 이러한 Write Combined 타입의 메모리에 대한 쓰기 동작을 수행하는 명령어로는 [MOVNTDQA](https://www.felixcloutier.com/x86/movntdqa)이 있다. 또한 Write Combined 타입의 페이지에 대한 쓰기 동작은 캐시를 활용하지 않고 메인 메모리에 쓰기가 수행된다. ( 프레임버퍼와 같이 GPU 즉 IO 장치에 매핑된 데이터는 당연히 Write Combined 타입이어야 GPU가 볼 수 있다. 그래서 프레임버퍼에 쓰기를 수행할 때도 Write Combined 버퍼를 통한 쓰기 최적화가 들어간다. )                  

잠깐 집고 넘어가야하는 것이 Write Combined 버퍼는 Write Combined 타입 메모리 쓰기에만 사용되는 것은 아니다. 위에서 배운 듯이 캐시에도 활용된다. Write Combined 타입 메모리 연산에 사용되니 Write Combined 버퍼라는 이름을 붙였지만 사실은 캐시에 쓸 때도 사용이 되니 정확하게는 "Store 버퍼"라는 용어가 더 정확한 것 같다.      

**캐시의 경우 그래도 속도가 빠르니 Write Combined 버퍼의 효과가 크게 두드러지지 않는데 DRAM의 Write Combined 타입의 데이터에 쓰기 동작을 수행할 때 Write Combined 버퍼는 엄청난 성능 향상을 불러온다.**          
DRAM에 데이터를 쓰려면 반드시 메모리 버스를 통해야 하는데 이**메모리 버스는 여러 코어가 공유하고 있고 DMA도 메모리 버스를 사용하기 때문에 메모리 버스를 자주 점유 ( 버스 마스터링 )하는 것은 성능상 매우 좋지 않다.** 그래서 데이터를 **모아두었다가** ( CPU의 Write Combined 버퍼에 ) **버스 마스터링을 한 후 한번만에 모아둔 데이터를 쓰는 것 ( 버스트 모드 )**이 **성능향상에 큰 도움**이 된다.            
이렇게 캐싱을 활용하지 않는 메모리 연산으로는 위에서 배운 것 처럼 **"non-temporal memory hint"** 연산이 경우가 대표적이다.         
또한 **메모리 맵 IO**가 또 다른 예인데 메모리 맵 IO의 경우 알다 싶이 CPU 입장에서는 일반적인 메모리 연산과 명령어 코드가 똑같고, 디바이스가 최신의 데이터를 보기 위해 캐싱을 하면 안된다는 특징을 가지고 있다. ( 바로 DRAM에 써야 디바이스가 최신의 데이터를 읽어갈 수 있다. )      


가장 중요한 것은 **완전히 채워지지 않는 Write Combined 버퍼를 flush 하는 것은 매우 매우 매우 최악의 행동**이라는 것이다. 메모리 버스 대역폭을 완전히 활용하지 못하는 쓰기 동작은 비효율적이니 임시로 Write Combined 버퍼에 데이터를 모아서 메모리 버스의 대역폭을 꽉꽉채워서 전송을 하자는 것이 Wrtie Combined 최적화의 핵심이다.       

그러니 CPU마다 가지고 있는 Write - Combine 버퍼의 개수에 맞추어서, 만약 Write Combine 버퍼의 개수가 4개라면 이 Write - Combine 기법의 이점을 최대한 활용하기 위해 4개보다 많은 캐시 라인을 연속적으로 건들면 안된다. 왜냐면 4개보다 많은 캐시라인을 건드는 순간부터 현재 flush 되지 않는 다른 캐시라인의 Write Combine 버퍼를 flush하게 되고 이는 매우 비효율적인 동작이기 때문이다. ( [예시](https://m.blog.naver.com/kimyoseob/220662359748) )      
그러니 절대로 **보유중인 Write - Combine 버퍼의 개수보다 많은 캐시라인은 동시에 건드려서 기존 버퍼가 다 차기 전에 flush 해버리는 치명적인 성능 하락을 만들지마라.**                         

아래 사진은 Write Combined 버퍼를 완전히 채우지 않고 flush 했을 경우의 성능을 비교한 사진이다.       
64바이트의 경우가 Write Combined 버퍼를 완전히 채운 경우이다.        

![write_combine](https://user-images.githubusercontent.com/33873804/134978510-deaee18d-f7ab-4350-a8d0-13df12d3aa6c.png)      
64바이트를 제외한 다른 쓰기 동작들은 모두 워드 단위로 쪼개져서 메모리에 써진다.            

이러한 Write Combined 버퍼는 L1캐시로 캐시라인을 가져오는데도, L1과 L2 캐시간 캐시라인 전송, DRAM 전송 등 여러군데서 활용되기 때문에 꽉 차지 않은 Write Combined 버퍼를 flush하는 것은 성능상 매우 매우 좋지 않다.      


자자 위의 내용들을 종합해서 한가지 더 알려주겠다.         
만약에 **Write Combined 타입의 페이지에서 메모리를 읽으려고하면 무슨일이 벌어질까?**               
우선 Write Combined 타입의 메모리를 읽는 것은 캐싱되지 않는다. 그리고 Write Combined 타입을 읽으려고 하면 **존재하는 write combined 버퍼를 모두 flush를 해야한다**. ( 당연히 write combined 버퍼를 flush해야 신선한(?), 최신의 데이터를 읽을 수 있다. ) 여기서 Write Combined 버퍼를 모두 flush 한다는 것은 높은 확률로 다 차지도 않은 write combined 버퍼를 flush 해버린다는 것이다. 이는 **위에서 말한대로 매우 매우 비효율적**이다. ( 물론 캐시되지 않은 데이터를 읽는 동작 자체가 느리기는 하지만 IO를 위한 데이터들은 캐싱을 하지 않고 메모리에 쓰니 캐싱을 하지 않는 상황을 가정하자. )              

그러니 특별한 이유가 없다면 **절대로 write-combining 메모리를 읽지마라.** 특히 렌더링 관점에서 작성 중인 constant buffers, vertex buffers, index buffers 는 절대 읽지마라. 이 버퍼들은 write combined 타입의 메모리 ( 이 버퍼들은 GPU에서 읽어가야하므로 캐싱을 하지 않고 바로 메모리에 쓰기 동작을 하는 Write-Combined 유형의 데이터들이다 )이기 때문에 읽으려는 것은 최악이다..                 

GPU와 관련해서 Write-Combined 버퍼가 제일 많이 활용되는 것이 GPU와 같은 IO 장치와 대량의 데이터를 주고 받는 **Memory mapped IO 통신을 할 때**이다. 메모리 맵된 IO의 프로세스 가상 주소 공간은 Write-Combined 타입 ( 캐싱이 안되는 )의 페이지로 non-temporal hint 명령어를 사용해 쓰기 동작을 수행할 때 Write-Combined 버퍼가 활용된다. 그래서 **VRAM으로부터 메모리 맵된 텍스쳐 버퍼에 non-temporal hint로 쓰기 동작을 수행하면 Write-Combined 버퍼가 활용되고 쓸 데이터 크기가 크다면 이 Write-Combined 버퍼를 활용해서 IO 장치에 데이터를 전송함으로서 오는 이득이 매우 클 것**이다.           

렌더링에서 활용되는 write-combined 버퍼에 대해서는 [이 글](https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/)을 읽어보기 바란다.       

또한 write-combined 버퍼에 쓰는 경우 경우 메모리 ordering을 보장 ( 코어간 데이터 일관성을 보장 )하지 않는다. 그래서 write - combined 최적화는 대량의 데이터를 빠르게 보내고자 할 때 사용되고 대량의 데이터를 전송하는 중간에 read를 하는 것이 필요없는 상황에서 사용되어야한다.            

근데 [Write - Combined 기법이 항상 빠르지는 않다는 글](https://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/)도 있다... 고려할 경우의 수가 너무 많다. 궁금하다면 한번 읽어보아라.          

참고 글 : [https://megayuchi.com/2021/06/06/ddraw-surface-d3d-dynamic-buffer-%EC%97%90%EC%84%9C%EC%9D%98-write-combine-memory/](https://megayuchi.com/2021/06/06/ddraw-surface-d3d-dynamic-buffer-%EC%97%90%EC%84%9C%EC%9D%98-write-combine-memory/), [https://stackoverflow.com/questions/45623007/wc-vs-wb-memory-other-types-of-memory-on-x86-64/45634024?fbclid=IwAR1XGxliAepTdP4f_uqKB-QFGjGn9bK8Q91NOuSSMu3R4SgiNJS96LgdYHw](https://stackoverflow.com/questions/45623007/wc-vs-wb-memory-other-types-of-memory-on-x86-64/45634024?fbclid=IwAR1XGxliAepTdP4f_uqKB-QFGjGn9bK8Q91NOuSSMu3R4SgiNJS96LgdYHw)               


references : [https://stackoverflow.com/a/37092/7138899](https://stackoverflow.com/a/37092/7138899), [https://mechanical-sympathy.blogspot.com/2011/07/write-combining.html](https://mechanical-sympathy.blogspot.com/2011/07/write-combining.html), [https://sites.utexas.edu/jdm4372/2018/01/01/notes-on-non-temporal-aka-streaming-stores/](https://sites.utexas.edu/jdm4372/2018/01/01/notes-on-non-temporal-aka-streaming-stores/), [https://stackoverflow.com/questions/14106477/how-do-non-temporal-instructions-work](https://stackoverflow.com/questions/14106477/how-do-non-temporal-instructions-work), [https://vgatherps.github.io/2018-09-02-nontemporal/](https://vgatherps.github.io/2018-09-02-nontemporal/),  [http://www.nic.uoregon.edu/~khuck/ts/acumem-report/manual_html/ch05s03.html](http://www.nic.uoregon.edu/~khuck/ts/acumem-report/manual_html/ch05s03.html), [https://stackoverflow.com/questions/49959963/where-is-the-write-combining-buffer-located-x86](https://stackoverflow.com/questions/49959963/where-is-the-write-combining-buffer-located-x86), [https://www.i-programmer.info/programming/hardware/3114-write-combining.html](https://www.i-programmer.info/programming/hardware/3114-write-combining.html)                                       
