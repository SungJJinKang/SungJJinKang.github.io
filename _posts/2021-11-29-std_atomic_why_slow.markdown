---
layout: post
title:  "std::atomic 느린 이유 ( 작성 중 )"
date:   2021-11-29
categories: ComputerScience
---

No. x86 gets release/acquire ordering for no additional cost (paid for by the chip). The default for std::atomic is SC, which requires an mfence for stores on x86. Mfence I wouldn't describe as cheap, but I don't know if it's accurate to say that it flushes all writes.

Also, std::atomic is not guaranteed to be lockfree. It can be promoted to use a mutex. On x86 you can get up to 16 bytes lockfree atomic. If you need to check, member functions are provided.

Neither of these are what I would call slow. You should always say what you're comparing against and/or what measurement.

See https://stackoverflow.com/questions/tagged/c%2b%2b%20multithreading chapter 10 for a comprehensive list of atomic code generation on common arches.

references : https://www.reddit.com/r/cpp/comments/r2mj9q/comment/hmfyq0p/?utm_source=share&utm_medium=web2x&context=3