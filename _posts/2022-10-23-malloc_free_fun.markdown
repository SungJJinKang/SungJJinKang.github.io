---
layout: post
title:  "malloc, free에 관한 재밌는 관점"
date:   2022-10-23
tags: [ComputerScience]
---          
 
간단히 설명하면 malloc시 반환해주는 메모리 블록의 실제(!) 사이즈를 알려주면 성능 향상을 얻을 수 불필요한 메모리 할당도 줄일 수 있다.    
마찬가지로 free시 해당 메모리 블록의 사이즈를 명시적으로 알려주면 성능 향상을 얻을 수 있는 여지가 생긴다.                          
                 
[malloc() and free() are a bad API](https://www.foonathan.net/2022/08/malloc-interface/#content)          
[Size feedback in operator new](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p0901r9.html#biblio-jemalloc)           
