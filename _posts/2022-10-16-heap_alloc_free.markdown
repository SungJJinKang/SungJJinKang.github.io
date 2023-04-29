---
layout: post
title:  "힙 할당, 해제 구현(glibc, 최적화를 위한 여러 기법들..., Arena, Binnning)"
date:   2022-10-16
tags: [ComputerScience, Recommend]
---         

[https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/](https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/),              
[https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/](https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/),                           
[동적 할당 메타 데이터 구조](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1059)            
           
<img width="611" alt="mallocfree1" src="https://user-images.githubusercontent.com/33873804/196029468-e99ddeef-001a-471f-89b5-54d5fcba4f59.png">                      
<img width="611" alt="mallofree2" src="https://user-images.githubusercontent.com/33873804/196029470-cddb2fa6-c7cd-4a48-95b5-beb86f728fe8.png">

---------------------- 

추가로 스터디에서 언리얼 엔진4의 메모리 할당자 관련 발표한 내용도 첨부함.        
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-37](https://user-images.githubusercontent.com/33873804/218701451-afa685ec-f312-4ff3-bab4-11fb40681b3c.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-38](https://user-images.githubusercontent.com/33873804/218701457-4e480a0c-54be-4994-87d5-8181bd0ff33a.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-39](https://user-images.githubusercontent.com/33873804/218701462-f889d77f-de15-4718-a43d-5686613fa746.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-40](https://user-images.githubusercontent.com/33873804/218701464-7e67865b-0df7-4897-9b7e-fbdc344c267a.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-41](https://user-images.githubusercontent.com/33873804/218701467-4f6e1875-f102-4df8-8644-a5ddf9f68324.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-42](https://user-images.githubusercontent.com/33873804/218701469-8efa0eea-fb67-43fc-9ac7-2cdf166dad8c.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-43](https://user-images.githubusercontent.com/33873804/218701471-4714900e-464c-4eb8-9883-d4b7dab979f2.png)
![메모리 측면에서 안전하고 빠른 언리얼 엔진 알아보기(외부용)-44](https://user-images.githubusercontent.com/33873804/218701472-ada45155-7149-4c4e-8640-99ed7221c753.png)
              
질문 및 답변      
1. MallocBinned2의 소형 메모리 풀에서 메모리를 어떻게 할당 받아 어떻게 나누나?
```
각각의 스몰 사이즈 풀에서 각자 메모리를 할당 받음.
각 사이즈의 스몰 사이즈 풀의 메모리가 부족하면 최소 64KB의 연속된 메모리를 왕창 할당 받아서 이걸 해당 스몰 사이즈 풀에서 쪼개서 사용
ex) 16 바이트용 풀에 메모리가 부족하다? → 일단 64KB를 할당 받고 이 64KB를 쪼개서 16 바이트씩 유저에게 반환해줌. ( FMallocBinned2::MallocExternalSmall 참고 )
64KB를 할당 받을 때 안드로이드 기준 "FPooledVirtualMemoryAllocator" 여기서 할당을 받아옴.

64KB의 연속된 메모리를 OS로부터 할당받으면 맨 앞 16바이트는 이 64KB짜리 FreeBlock의 헤더로 사용
그럼 어떻게 특정 주소가 Free되었을 때 그 주소가 어느 사이즈의 Pool에 속하였는지 확인하나? ( 스몰 사이즈 풀에 속하는 경우 )
 Free된 주소보다 작으면서, Free된 주소에 가장 가까운 64KB에 Align되어 있는 주소를 구함. ( FMallocBinned2::GetPoolHeaderFromPointer 함수 참고 )
 이렇게 구한 주소의 첫 16바이트는 위의 FFreeBlock의 헤더 데이터이다. ( FFreeBlock은 항상 64KB에 Align 되어 있다 )
 64KB의 FFreeBlock의 첫 16바이트는 헤더 데이터로 사용되어 보존되어야 하기 때문에 유저에 할당되지 않는다.
 이 헤더 정보를 읽어서 그 Free된 주소가 어떤 풀에 속한지를 찾아내고 해당 풀의 Free Block 리스트에 추가한다. ( 풀에서는 링크드 리스트로 관리됨, FBundleNode 참고 )
 이러한 구조를 보았을 때 각각의 힙 할당마다 해당 할당에 메타데이터를 함께 관리하지 않아도 되는 장점이 있어 보임.
```

2. MallocBinned2 할당자의 스레드 전용 메모리 풀과 관련하여...

```
우선 스레드 전용 메모리 풀이 필요한 이유는 간단히 설명하면 스레드들이 공유하는 메모리 풀 접근시 Race Condition을 막기 위한 lock 동작으로 성능 저하가 있다 ( 플랫폼에 따라 다르지만 lock 동작도 경합(contention) 상태가 아닌 경우에는 성능 저하가 크지는 않은 것으로 앎,  )
스레드 전용 메모리 풀에 원하는 사이즈의 메모리 블록이 있는 경우 여기서 꺼내서 사용 → 스레드 전용 메모리 풀이니 스레드간 Race Condtion을 막기 위한 lock 동작이 필요 없어 이로 인한 성능 저하를 줄일 수 있다. 
구체적인 구현은?
 스레드 전용 메모리 풀 내에서도 여러 스몰 사이즈 풀들을 관리.
 그럼 이 스레드 전용 스몰 사이즈 풀에는 언제 메모리 블록이 충전되나?
     스레드 전용 스몰 사이즈 풀에 원하는 사이즈의 메모리 블록이 없는 경우 스레드들이 공유하는 메모리 블록에서 메모리 블록을 할당 받음.
         이때 스레드 공용 메모리 풀에 있는 다른 사이즈의 Free 블록들도 스레드 전용 풀로 옮긴다. ( 스레드들이 공유하는 풀에 접근하며 Lock을 했으니 겸사 겸사 스레드 전용 풀에 메모리 블록들을 왕창 충전하는 개념 )
     마찬가지로 유저가 메모리를 Free하면 Free된 메모리 블록은 기본적으로 스레드 전용 메모리 풀로 들어간다.
 참고로 이러한 스레드 전용 메모리 풀은 많은 할당자 구현에서 채택하고 있는 방식.
```
