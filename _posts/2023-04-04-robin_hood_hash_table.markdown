---
layout: post
title:  "Robin-Hood Hash Table"
date:   2023-04-04
tags: [ComputerScience, Recommend]
---          
             
[기존 노드 기반 해쉬 테이블](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/hashtable.h)(각 노드들을 각각 별도로 힙할당)과 달리 Robin-Hood Hash Table은 Element들을 Linear하게 배치(TSparseArray로 Element들을 관리)하고 저장되어야 할 Bucket 인덱스와 충돌로 인해 Open Addressing으로 실제로 저장된 Bucket 인덱스과의 거리를 기준으로 최대한 저장되어야 할 Bucket 인덱스와 충돌로 인해 Open Addressing으로 실제로 저장된 Bucket 인덱스 사이의 거리를 좁혀 탐색시 TMap에 비해 더 캐시 친화적이다.                
언리얼 엔진4에서도 TRobinHoodHashMap, TRobinHoodHashSet라는 이름으로 구현되어 있다.      
언리얼 엔진4, 5에서 현재 시점 기준 Experimental이기는하지만 탐색시 높은 [퍼포먼스를 요구하는 코드](https://github.com/EpicGames/UnrealEngine/commit/3af7b646b76c976b02e55791ac3f2d0e692b856b)에서 종종 사용되고 있다.    
       
다만 언리얼 엔진의 TMap, TSet도 Element를 TSparseArray로 Element들이 Linear하게 메모리상에 배치하기 때문에 역시 캐시 친화적이기는 하다.           
                  
[알고리즘 설명](https://programming.guide/robin-hood-hashing.html), [알고리즘 설명2](https://study.com/academy/lesson/robin-hood-hashing-concepts-algorithms.html)            
