---
layout: post
title:  "GPU 메모리 이해하기 ( NVIDEA )"
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
[GPU - Pinned System Memory](https://mkblog.co.kr/2017/03/07/nvidia-gpu-pinned-host-memory-cuda/)      
[GPU 메모리 관리](https://gpuopen.com/wp-content/uploads/2018/05/gdc_2018_tutorial_memory_management_vulkan_dx12.pptx)            
[gpu-architecture-overview](https://insujang.github.io/2017-04-27/gpu-architecture-overview/)         
[https://insujang.github.io/2017-06-23/gpu-resource-management/](https://insujang.github.io/2017-06-23/gpu-resource-management/)          

----------------

[OpenGL Memory 전송 방식](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-AsynchronousBufferTransfers.pdf)         
[DX12 메모리 관리 전략](https://docs.microsoft.com/ko-kr/windows/win32/direct3d12/memory-management-strategies)                                  
[DX12 커맨드 큐, 커맨드 리스트 디자인 철학](https://docs.microsoft.com/en-us/windows/win32/direct3d12/design-philosophy-of-command-queues-and-command-lists)           
[DX12의 리소스 업로드](https://docs.microsoft.com/en-us/windows/win32/direct3d12/uploading-resources)           
[DX12의 텍스쳐 데이터 업로드, 다시 읽기](https://docs.microsoft.com/en-us/windows/win32/direct3d12/upload-and-readback-of-texture-data)            

