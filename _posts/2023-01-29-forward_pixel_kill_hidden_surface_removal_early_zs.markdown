---
layout: post
title:  "Forward Pixel Kill(FPK)"
date:   2023-01-29
tags: [ComputerGraphics]
---            

**"Forward Pixel Kill(FPK)"**은 일부 ARM Mali GPU에 도입된 기술이다.       
쉽게 말하면 **Early Depth Test를 통과한 픽셀(Quad)을 곧 바로 Fragment Shading 처리하지 않고 잠시 대기(지연) 시킨 후, 이후 처리되는 삼각형의 픽셀의 Depth 값과 비교해 Overdraw된다면 대기 중이던 Pixel을 Kill(Fragment Shading 연산에서 제외)하는 기법**이다.           
                     
아래 사진은 Early Depth Test 이후, Fragment Shading 단계 전 FIFO 버퍼를 추가하여 FPK를 구현한 구조를 보여준다. ( 이 구현은 이미 Fragment Shading 처리 중인 쉐이더를 죽이는 동작은 아니고 Fragment Shading 처리가 시작되기 이전에 Fragment Shading 연산 대상에서 제외시키는 구현이다. )        
![209149-20200106145650469-450617726](https://user-images.githubusercontent.com/33873804/215318128-844f0eac-6fbc-443f-b0f2-d13b4bc45b80.png)           
Early Depth Test 단계를 통과하여 새롭게 FPK FIFO 버퍼에 추가 될 Quad(Position 10, Depth 0)(**Early Depth Test, FPK은 Quad 단위로 Culling 된다**)은 이미 FIFO 버퍼에 추가되어 있는 Quad(Position 10, Depth 10)보다 Depth 값이 더 작으므로 기존 Quad를 대체(**기존에 버퍼에 들어 있던, Fragment Shading 처리를 대기 중이던 Quad를 Fragment Shading 연산 대상에서 제외한다, 버린다**)한다. (Opaque한 삼각형을 렌더링하고, Fragment Shader에서 Depth 값을 수정하지 않고, discard 동작을 수행하지 않는다면 Fragment Shading 수행 전 Depth 값을 알 수 있다. )        
먼저(Forward) 처리될 예정이 었던 픽셀(Pixel, 정확히는 Quad)을 Kill한다고 해서 "Forward Pixel Kill"이라고 부른다...            
새롭게 Queue에 추가 될 Quad(Position 10, Depth 0)에 의해 오버드로우 될 예정인 기존 Quad(Position 10, Depth 10)를 (Fragment Shading이 시작되기 전에 미리) 제거함으로서 해당 Quad(Overdraw 될 Quad)에 대한 불필요한 연산(Fragment Shading) 낭비를 막을 수 있게 된 것이다.              
**Quad를 Fragment Shading 단계 전에 이렇게 FIFO 버퍼는 추가하는 것도 Quad가 Fragment Shading 처리되는 것을 의도적으로 지연(!)시켜서 FPK를 극대화하기 위함**이다.           
FPK는 Early Depth Test에서 걸러지지 못한(Overdraw될 것임을 확정하지 못해 Early Depth Test를 통과한) 픽셀들을 추가로 걸러준다.            
다만 이러한 FIFO 버퍼도 저장할 수 있는 Quad 데이터의 개수가 한정되어 있기 때문에 이 FIFO 버퍼가 꽉 찬다면 맨 앞에 있는 Quad는 Fragment Shading 처리해주어 새로운 Quad가 들어올 공간을 마련해주어야 한다.(FPK의 효과를 극대화하기 위해서는 FIFO 버퍼 사이즈가 충분히 커야한다.)            
          
흔히 Early Depth Test(Early Z Kill)의 효과를 극대화하기 위해 카메라로부터 가까운 오브젝트부터 먼저 그리는데, 이 FPK 기술은 FIFO 버퍼를 충분한 크기로 확보한다면 오브젝트의 카메라로부터의 거리를 고려하지 않고(Sorting하지 않고) 렌더링을 수행하여도 Sorting을 했을 때 못지 않은 퍼포먼스를 보여준다고 ARM에서는 설명한다.(먼 거리의 오브젝트부터 그려도 해당 오브젝트의 픽셀들이 곧 바로 Fragment Shading 처리되지 않고 FIFO 버퍼에 대기하기 때문에 이후 보다 가까운 오브젝트를 처리할 때 FPK의 효과로 FIFO 버퍼에 대기 중이던 먼 거리의 오브젝트의 픽셀들을 Kill할 수 있기 때문이다..) ( 필자의 생각을 조금 덧붙여보자면 이렇게 FIFO 버퍼에 픽셀을 대기시켜두면 GPU가 그 만큼 놀게(Idle)되어 오히려 성능 저하가 발생할 수 있지 않을까 하는 생각도 해본다.. 그 부분까지 고려를 했으려나... ) 다만 ARM에서도 단서를 붙인 것이 그럼에도 불구하고 Early-Z Test가 FPK에 비해 전력 효율이 좋고, 오래된 Mali GPU에서도 동작하기 때문에(9년 전 나온 Samsung Exynos5420에도 FPK가 적용되어 있는걸 보면, 최신 Mali GPU에서는 왠만하면 다 적용되어 있을 듯.) 오브젝트 Sorting을 수행하는 것은 여전히 퍼포먼스를 위해 필수적이라고 말한다.           
               
                 
ARM쪽 설명에 의하면 기존 Frame Buffer 의 값을 활용하거나(Blending하거나), Alpha Test를 사용하거나 삼각형의 크기가 너무 작거나, 프레임 버퍼를 공유하는 경우 FPK가 적용되지 않는다고 한다.          
         
           
references : [https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/killing-pixels---a-new-optimization-for-shading-on-arm-mali-gpus](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/killing-pixels---a-new-optimization-for-shading-on-arm-mali-gpus), [https://www.cnblogs.com/timlly/p/15546797.html](https://www.cnblogs.com/timlly/p/15546797.html), [https://zhuanlan.zhihu.com/p/464337040](https://zhuanlan.zhihu.com/p/464337040)                   