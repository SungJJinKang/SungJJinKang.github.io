---
layout: post
title:  "현대 GPU 아키텍쳐 이해하기 2 ( Turing Architecture )"
date:   2021-09-09
categories: ComputerScience ComputerGraphics
---

이 글은 [Graphics API에서 커맨드나 데이터들이 어떻게 GPU로 전달되는지에 대한 글](https://sungjjinkang.github.io/computerscience/computergraphics/2021/09/04/gpu_architecture.html)의 2탄이다.        
이 글에서는 GPU 밖에서 GPU로 커맨드나 데이터가 전달된 후 그것들이 어떻게 GPU 내부에서 처리되는지에 대한 것들을 다룰 것이다.              
구체적으로는 현 세대 GPU ( RTX~~ )의 아키텍쳐인 Turing ( 튜링 ) 아키텍쳐에 대해 다룰 것이다.        

---------------------------------

references : [https://mkblog.co.kr/2018/10/01/gpgpu-series-3-gpu-architecture-overview/](https://mkblog.co.kr/2018/10/01/gpgpu-series-3-gpu-architecture-overview/),  [https://blog.cherryservers.com/everything-you-need-to-know-about-gpu-architecture](https://blog.cherryservers.com/everything-you-need-to-know-about-gpu-architecture), [https://developer.nvidia.com/blog/nvidia-turing-architecture-in-depth/](https://developer.nvidia.com/blog/nvidia-turing-architecture-in-depth/), [https://gpltech.com/wp-content/uploads/2018/11/NVIDIA-Turing-Architecture-Whitepaper.pdf](https://gpltech.com/wp-content/uploads/2018/11/NVIDIA-Turing-Architecture-Whitepaper.pdf), [https://blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=padagi20&logNo=221204107315](https://blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=padagi20&logNo=221204107315), [https://mkblog.co.kr/2019/03/20/gpgpu-series-9-memory-hierarchy-on-gpus/](https://mkblog.co.kr/2019/03/20/gpgpu-series-9-memory-hierarchy-on-gpus/)