---
layout: post
title:  "Robin-Hood Hash Table"
date:   2023-04-04
tags: [ComputerScience]
---          
             
기존 노드 기반 해쉬 테이블(각 노드들을 각각 별도로 힙할당)과 달리 Bucket을 Linear하게(TArray) 배치하므로 캐시 친화적이다.         
언리얼 엔진4에서도 TRobinHoodHashMap, TRobinHoodHashSet라는 이름으로 구현되어 있다.         
                  
[알고리즘 설명](https://programming.guide/robin-hood-hashing.html)             