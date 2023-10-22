---
layout: post
title:  "모바일 GPU에서 Depth Pre Pass의 유용성? ( Hidden Surface Removal )"
date:   2022-04-24
tags: [ComputerGraphics, Recommend]
---

모바일 GPU에서는 Depth Pre Pass를 수행하지 마라?               
**모바일 GPU는 기본적으로 [Hidden Surface Removal](https://sungjjinkang.github.io/forward_pixel_kill_hidden_surface_removal_early_zs)를 수행**하기 때문에 Depth Pre Pass를 수행하면 오히려 성능적으로 손해(Depth Pre Pass도 결국 하나 하나의 드로우 콜이니... 물론 PS 비용이 상대적으로 싸지만...)일 수 있다.         
           
**Hidden Surface Removal**은 **Tile Based Rendering**을 기반으로 동작한다.        

**Tile Based Rendering**은 Vertex Shading 처리가 끝난 삼각형을 곧 바로 Fragment Shading에 태우지 않고 해당 삼각형이 화면 상에 어떤 Tile에 속해 있는지 연산하고 그 결과 값을 저장한다. 모바일 GPU는 한 프레임 동안 처리되는 모든 삼각형에 대해 이러한 동작(삼각형을 해당 삼각형이 속하는 타일에 모으는 동작)을 수행한다.         
그럼 각 타일에 모아지는 삼각형들의 Depth 값들을 Quad 단위로 비교해서 카메라에 가장 가까운(Depth 값이 가장 작은) Quad만을 남겨둔다(Forward Pixel Kill).       
이를 통해 다른 삼각형에 의해 가려지는 Quad에 대한 Pixel 쉐이딩 연산을 생략하는 것이다. ( Early-Z를 통과한 것들에 대해서 한번 더 검사를 하는 개념.. )                
![img](https://user-images.githubusercontent.com/33873804/215321316-f3b16e68-c59d-4ecd-879a-b17402bb6dc7.jpg)         
            
즉 "Depth Pre Pass"가 목표로 하는 효과를 모바일 GPU에서는 HW단에서 기본적으로 수행해주니, 굳이 Depth Pre Pass를 수행할 필요가 없다는 것이다. ( Arm Mali 문서에는 Depth Pre Pass 수행시 더 느릴 수 있다고 나와있다. )             
<img width="942" alt="그림1" src="https://user-images.githubusercontent.com/33873804/218688150-e6b69efe-8c23-4d45-b924-a226641ba439.png">                
             
               
필자가 확인한 바로는 Mali GPU와 애플의 아이폰에 사용되는 GPU에서는 Hidden Surface Removal이 커널단에서 적용되는 것으로 보인다. Adreno 계열도 확인 필요...              
            
또한 주의할 것은 **Alpha Blending이나, Alpha Test를 수행하는 경우, 픽셀 쉐이더에서 Depth 값을 쓰거나(갱신), 픽셀 쉐이더에서 Fragmnet를 버리는 동작(dicard)이 있는 경우 GPU가 해당 Primitive의 Fragment가 가려질지 여부를 판단하지 못하기 때문에 Hidden Surface Removal의 도움을 받지 못한다.**            
그래서 **모바일 환경의 경우, 위의 상황들을 최대한 피하여야 한다.**                            
                 
refererence : [https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses](https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses), [https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html](https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html), [https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/](https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/)
