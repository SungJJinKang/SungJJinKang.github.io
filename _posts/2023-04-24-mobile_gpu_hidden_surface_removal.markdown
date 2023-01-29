---
layout: post
title:  "모바일 GPU에서 Depth Pre Pass의 유용성? ( Hidden Surface Removal )"
date:   2023-04-24
tags: [ComputerGraphics]
---

모바일 GPU에서는 Depth Pre Pass를 수행하지 마라?               
**모바일 GPU는 기본적으로 Hidden Surface Removal를 수행**하기 때문에 Depth Pre Pass를 수행하면 오히려 성능적으로 손해(Depth Pre Pass도 결국 하나 하나의 드로우 콜이니... 물론 PS 비용이 상대적으로 싸지만...)일 수 있다.         
           
**Hidden Surface Removal**은 **Tile Based Rendering**을 기반으로 동작한다.        

**Tile Based Rendering**은 Vertex Shading 처리가 끝난 삼각형을 곧 바로 Fragment Shading에 태우지 않고 해당 삼각형이 화면 상에 어떤 Tile에 속해 있는지 연산하고 그 결과 값을 저장한다. 모바일 GPU는 한 프레임 동안 처리되는 모든 삼각형에 대해 이러한 동작을 수행한다.         
이러한 동작이 모두 끝나면, GPU는 각 타일마다 해당 타일에 어떤 삼각형이 그려질지를 모두 알게 된다.      
해당 타일 내에 속한 삼각형들의 Depth 값을 비교하여 연산할 필요가 없는 삼각형은 버린다.(Hidden Surface Removal)               
![img](https://user-images.githubusercontent.com/33873804/215321316-f3b16e68-c59d-4ecd-879a-b17402bb6dc7.jpg)         
         

필자가 확인한 바로는 Mali GPU와 애플의 아이폰에 사용되는 GPU에서는 Hidden Surface Removal이 커널단에서 적용되는 것으로 보인다. 스냅드래곤쪽도 확인 필요...              
            
또한 주의할 것은 **Alpha Blending이나, Alpha Test를 수행하는 경우, 픽셀 쉐이더에서 Depth 값을 쓰거나, 픽셀 쉐이더에서 Fragmnet를 버리는 동작이 있는 경우, GPU가 해당 Primitive의 Fragment가 가려질지 여부를 판단하지 못하기 때문에 Hidden Surface Removal의 도움을 받지 못한다.**            
그래서 **모바일 환경의 경우, 위의 상황들을 최소화해야한다.**                            
       
                 
refererence : [https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses](https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses), [https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html](https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html), [https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/](https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/)