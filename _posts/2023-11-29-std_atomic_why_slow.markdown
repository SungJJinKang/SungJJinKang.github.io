---
layout: post
title:  "std::atomic 느린 이유"
date:   2023-11-29
tags: [C++, ComputerScience]
---

std::atomic 연산은 mfence 명령어를 동반하는데 이는 CPU의 [Write-Combined 버퍼](https://sungjjinkang.github.io/nonTemporalMemoryHint)를 flush 해버린다. ( Cache에만 써도 Cache coherency로 코어간 데이터 동기화가 된다. )                  


또한 memory reordering 옵션이 있는 경우 reordering을 제한하여서 out-of-order 명령어 처리를 수행하지 못하는데서 오는 성능 하락도 있다.          

[https://www.felixcloutier.com/x86/mfence](https://www.felixcloutier.com/x86/mfence)         

-----------------------

No. x86 gets release/acquire ordering for no additional cost (paid for by the chip). The default for std::atomic is SC, which **requires an mfence for stores on x86.** Mfence I wouldn't describe as cheap, but I don't know if it's accurate to say that it flushes all writes.

Also, std::atomic is not guaranteed to be lockfree. It can be promoted to use a mutex. On x86 you can get up to 16 bytes lockfree atomic. If you need to check, member functions are provided.

Neither of these are what I would call slow. You should always say what you're comparing against and/or what measurement.

See https://stackoverflow.com/questions/tagged/c%2b%2b%20multithreading chapter 10 for a comprehensive list of atomic code generation on common arches.

references : https://www.reddit.com/r/cpp/comments/r2mj9q/comment/hmfyq0p/?utm_source=share&utm_medium=web2x&context=3
