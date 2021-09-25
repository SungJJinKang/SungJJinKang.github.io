---
layout: post
title:  "어떻게 GPU는 DRAM에 있는 데이터에 접근하는가?"
date:   2021-06-21
categories: ComputerScience ComputerGraphics
---

어떻게 GPU는 DRAM에 있는 데이터에 접근하는가?  

우선 흔히 사용하는 외장 GPU부터 다루어 보겠다. 외장 GPU는 DRAM에 DMA라는 기능을 통해 메모리에 있는 데이터에 접근한다. 이 DMA는 주변 장치가 DRAM에 접근할 때 CPU를 거치지 않고 직접 DRAM에 접근하기 위해 고안된 기능이다. 이 방식을 통해 주변 장치가 DRAM에 접근 중일 때 CPU는 다른 작업을 수행할 수 있어 효율성이 높아진다. ( 메모리에 있는 데이터를 전송하거나 어떤 작업을 수행하려면 원래는 무조건 CPU의 레지스터에 메모리에 있는 데이터를 옮긴 후 데이터를 전송하거나 어떤 작업을 수행해야한다. ) ( 여기서 나온 이 DMA라는 장치는 SSD 와 DRAM간에도 존재하여 CPU 레지스터의 도움 없이 SSD에 있는 데이터를 곧 바로 DRAM으로 전송할 수 있게 ( 물론 그 반대도 가능 ) 도와준다. )                     

이 DMA는 메인보드와 GPU 사이의 PCI-E 버스를 통해서 이루어지고 GPU는 메모리 버스 통제권 가지고 메모리 컨트롤러를 통해 DRAM에 직접적으로 접근한다.       
그리고 일반적으로 사용하는 GPU는 오직 하나의 메모리 컨트롤러(혹은 MMU)를 가지고 있기 때문에 한 시점에 하나의 데이터 전송 밖에 수행하지 못한다.        

그리고 마찬가지로 CPU 또한 외장 GPU가 가지고 있는 GPU 전용 램(VRAM)에 DMA를 통해 접근할 수 있다.
각각 메모리 버스 통제권을 얻을 수 있다. 버스 마스터가 될 수 있는 주변장치는 필요에 따라 메모리 주소와 제어 신호를 제공 받아 CPU를 거치지 않고 시스템 메모리(DRAM)에 직접 쓸 수 있다.         

추가적으로 DMA에 대한 깊은 이해를 하고 싶으면 [이 글](https://sungjjinkang.github.io/computerscience/2021/09/26/IO_System.html)을 읽기바란다.            
[메모리 맵 IO와 DMA의 차이](https://stackoverflow.com/questions/3851677/what-is-the-difference-between-dma-and-memory-mapped-io)        


references : [https://stackoverflow.com/questions/11355426/gpu-system-memory-mapping](https://stackoverflow.com/questions/11355426/gpu-system-memory-mapping)  ,  [https://www.quora.com/How-does-a-GPU-share-memory-with-a-CPU-How-can-they-access-it-at-the-same-time](https://www.quora.com/How-does-a-GPU-share-memory-with-a-CPU-How-can-they-access-it-at-the-same-time)  ,  [https://superuser.com/questions/1545812/can-the-gpu-use-the-main-computer-ram-as-an-extension](https://superuser.com/questions/1545812/can-the-gpu-use-the-main-computer-ram-as-an-extension)  ,  
