---
layout: post
title:  "Robin-Hood Hash Table"
date:   2023-04-04
tags: [ComputerScience]
---          
             
[기존 노드 기반 해쉬 테이블](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/include/bits/hashtable.h)(각 노드들을 각각 별도로 힙할당)과 달리 Robin-Hood Hash Table은 Element들을 Linear하게 배치(TArray로 Element들을 관리)하므로 캐시 친화적이다.             
언리얼 엔진4에서도 TRobinHoodHashMap, TRobinHoodHashSet라는 이름으로 구현되어 있다.      
다만 언리얼 엔진의 TMap, TSet도 Element를 TSparseArray로 Element들이 Linear하게 메모리상에 배치하기 때문에 역시 캐시 친화적이기는 하다.                   
                  
[알고리즘 설명](https://programming.guide/robin-hood-hashing.html), [알고리즘 설명2](https://study.com/academy/lesson/robin-hood-hashing-concepts-algorithms.html)            
