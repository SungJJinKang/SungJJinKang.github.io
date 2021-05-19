---
layout: post
title:  "std::function은 왜 느릴까?(작성 예정)"
date:   2021-05-20
categories: C++
---

어떤 함수를 호출할지 알 수 없기 때문에(내부적으로 함수포인터로 구현) Memory Stall에 대비한 Instruction Reordering, Cache prefetcher의 이점을 못얻음
