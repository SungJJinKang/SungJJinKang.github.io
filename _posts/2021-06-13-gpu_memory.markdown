---
layout: post
title:  "어떻게 GPU는 DRAM에 있는 데이터에 접근하는가?"
date:   2021-06-21
tags: [ComputerScience, ComputerGraphics]
---

어떻게 GPU는 DRAM에 있는 데이터에 접근하는가?  

우선 흔히 사용하는 외장 GPU부터 다루어 보겠다. 외장 GPU는 DRAM에 DMA라는 기능을 통해 메모리에 있는 데이터에 접근한다. 이 DMA는 주변 장치가 DRAM에 접근할 때 CPU를 거치지 않고 ( 일반적으로는 CPU가 주소버스에 GPU가 접근할 주소를 써주어야한다 ) 직접 DRAM에 접근하기 위해 고안된 기능이다. 이 방식을 통해 주변 장치가 DRAM에 접근 중일 때 CPU는 다른 작업을 수행할 수 있어 효율성이 높아진다. ( 여기서 나온 이 DMA라는 장치는 SSD 와 DRAM간에도 존재하여 CPU의 도움 없이 SSD에 있는 데이터를 곧 바로 DRAM으로 전송할 수 있게 ( 물론 그 반대도 가능 ) 도와준다. ) ( 조금 더 구체적으로 말하면 DRAM에서 GPU와 같은 다른 IO 장치로 데이터를 전송하기 위해서는 일반적으로는 CPU가 직접 데이터를 PCI버스에 보낼 메모리 주소를 써주어야만 메모리 디바이스 컨트롤러가 버스에 데이터를 올리고 GPU의 디바이스 컨트롤러가 데이터를 읽을 수 있는데 할 일이 많은 CPU가 이걸 직접하려고 하면 다른 일을 할 수 없으니 이 일을 DMA 컨트롤러가 대신 해주는 것이다. )                                    

이 DMA는 메인보드와 GPU 사이의 PCI-E 버스를 통해서 이루어지고 GPU는 메모리 버스 통제권 가지고 메모리 컨트롤러를 통해 DRAM에 직접적으로 접근한다.       
그리고 일반적으로 사용하는 GPU는 오직 하나의 메모리 컨트롤러(혹은 MMU)를 가지고 있기 때문에 한 시점에 하나의 데이터 전송 밖에 수행하지 못한다.        

그리고 마찬가지로 CPU 또한 외장 GPU가 가지고 있는 GPU 전용 램(VRAM)에 DMA를 통해 접근할 수 있다.
각각 메모리 버스 통제권을 얻을 수 있다. 버스 마스터가 될 수 있는 주변장치는 필요에 따라 메모리 주소와 제어 신호를 제공 받아 CPU를 거치지 않고 DMA를 통해 시스템 메모리(DRAM)로 데이터를 전송할 수 있다.                   

추가적으로 DMA에 대한 깊은 이해를 하고 싶으면 [이 글](https://sungjjinkang.github.io/computerscience/2021/09/26/IO_System.html)을 읽기바란다.            
[메모리 맵 IO와 DMA의 차이](https://stackoverflow.com/questions/3851677/what-is-the-difference-between-dma-and-memory-mapped-io)        
[GPU에서 주소 공간 관리](https://nemoux00.wordpress.com/2014/09/09/wayland-gpu-%EB%8F%99%EC%9E%91-%EC%9B%90%EB%A6%AC-%EB%A9%94%EB%AA%A8%EB%A6%AC-%EC%A3%BC%EC%86%8C-%EA%B3%B5%EA%B0%84-%EA%B4%80%EB%A6%AC/)


references : [https://stackoverflow.com/questions/11355426/gpu-system-memory-mapping](https://stackoverflow.com/questions/11355426/gpu-system-memory-mapping)  ,  [https://www.quora.com/How-does-a-GPU-share-memory-with-a-CPU-How-can-they-access-it-at-the-same-time](https://www.quora.com/How-does-a-GPU-share-memory-with-a-CPU-How-can-they-access-it-at-the-same-time)  ,  [https://superuser.com/questions/1545812/can-the-gpu-use-the-main-computer-ram-as-an-extension](https://superuser.com/questions/1545812/can-the-gpu-use-the-main-computer-ram-as-an-extension)  ,  
