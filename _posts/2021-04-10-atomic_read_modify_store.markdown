---
layout: post
title:  "atomic변수에서 Read-Modify-Store Operation은 locking없이 atomic할까?"
date:   2021-04-10
categories: ComputerScience
---
[https://stackoverflow.com/questions/39393850/can-num-be-atomic-for-int-num](https://stackoverflow.com/questions/39393850/can-num-be-atomic-for-int-num)      
[https://stackoverflow.com/questions/67034400/atomic-variable-also-require-lock-on-read-modify-store-operation](https://stackoverflow.com/questions/67034400/atomic-variable-also-require-lock-on-read-modify-store-operation)       


--------------------------------------

```c++
std::atomic<int> a;
a++; --> a.fetch_add(1)
++a; --> a.fetch_add(1) + 1
```
