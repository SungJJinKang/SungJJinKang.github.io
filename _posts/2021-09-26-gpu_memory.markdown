---
layout: post
title:  "GPU 메모리 이해하기"
date:   2021-09-26
categories: ComputerScience ComputerGraphics
---

이 글에서는 GPU에 있는 메모리 종류부터 어떻게 DRAM에서 GPU로 데이터가 전송되는지 등 GPU의 메모리와 관련해서 포괄적으로 많은 내용들을 다룰 것이다.          

DMA , Memory Mapped IO, IO mapped IO

-----------------





refereucens :          

[GPU Memory 종류](https://mkblog.co.kr/2016/11/26/nvidia-gpu-memory-types/)         
[Memory Hierarchy on GPUs](https://mkblog.co.kr/2019/03/20/gpgpu-series-9-memory-hierarchy-on-gpus/)            
[Memory Coalescing이란?](https://mkblog.co.kr/2016/12/01/nvidia-gpu-memory-coalescing-coalesced-memory-access/)       
[GPU 메모리 종류](https://www.ce.jhu.edu/dalrymple/classes/602/Class13.pdf)          
[DX12 메모리 관리 전략](https://docs.microsoft.com/ko-kr/windows/win32/direct3d12/memory-management-strategies)                
[GPU - Pinned System Memory](https://mkblog.co.kr/2017/03/07/nvidia-gpu-pinned-host-memory-cuda/)           