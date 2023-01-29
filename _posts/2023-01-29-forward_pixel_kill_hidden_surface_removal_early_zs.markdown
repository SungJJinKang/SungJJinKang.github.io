---
layout: post
title:  "Forward Pixel Kill"
date:   2023-01-29
tags: [ComputerGraphics]
---            

**"Forward Pixel Kill"**은 일부 ARM Mali GPU에 도입된 기술이다.       
쉽게 말하면 픽셀이 이후 처리된 다른 픽셀에 의해 오버드로우 될 것을 감지하면 이미 픽셀 쉐이딩 연산을 수행 중인 스레드의 연산을 멈추는 기법이다.(혹은 픽셀 쉐이딩 연산 전에 해당 픽셀을 버린다.)           
GPU는 **수 많은 스레드들이** Warp 단위로 **동시에 연산을 수행**하기 때문에 현재 스레드가 처리 중인 픽셀의 동일 위치에 다른 스레드들도 동시에 픽셀 처리 연산을 수행하고 있을 수 있다. 또한 현재 스레드가 먼저 픽셀 쉐이딩 연산을 시작하였어도 데이터 Fetch나 연산량 등의 변수로 뒤 늦게 픽셀 쉐이딩 연산을 수행하기 시작한 스레드보다 늦게 픽셀 쉐이딩 연산을 마칠 가능성도 있다.             
"Forward Pixel Kill" 기법은 GPU의 이러한 특징을 바탕으로 **픽셀 쉐이딩 연산 도중에도** 다른 스레드가 Depth Buffer에 쓴 Depth 값으로 인해 Depth Test를 통과하지 못할 것이 확인된 경우 현재 수행 중인 픽셀 쉐이딩 연산을 중단하여 불필요한 연산을 줄일 수 있게 해준다.           
좀 더 정확하게는 **Quad 단위로 FPK Culling을 수행**한다.            
              
아래 사진은 Early Z 이후, Fragment Shading 단계 전 FIFO 버퍼를 추가하여 FPK를 극대화하는 구조를 보여준다.        
![209149-20200106145650469-450617726](https://user-images.githubusercontent.com/33873804/215318128-844f0eac-6fbc-443f-b0f2-d13b4bc45b80.png)           
새롭게 FIFO 버퍼에 추가 될 Quad(Position 10, Depth 0)은 이미 FIFO 버퍼에 추가되어 있는 Quad(Position 10, Depth 10)보다 Depth 값이 더 작으므로 기존 Quad를 대체한다.         
새롭게 추가 될 Quad(Position 10, Depth 0)에 의해 오버드로우 될 예정인 기존 Quad(Position 10, Depth 10)를 (Fragment Shading이 시작되기 전에 미리) 제거함으로서 해당 Quad(Overdraw 될 Quad)에 대한 불필요한 연산 낭비를 막을 수 있게 된 것이다.              
               
                 
이러한 FPK는 Depth Test를 켜져 있고, Opaque한 오브젝트들에 대해서만 동작을 한다.        
         
           
references : [https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/killing-pixels---a-new-optimization-for-shading-on-arm-mali-gpus](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/killing-pixels---a-new-optimization-for-shading-on-arm-mali-gpus), [https://www.cnblogs.com/timlly/p/15546797.html](https://www.cnblogs.com/timlly/p/15546797.html)           