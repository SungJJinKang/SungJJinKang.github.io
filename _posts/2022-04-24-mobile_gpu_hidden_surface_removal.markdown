---
layout: post
title:  "모바일 GPU에서 Depth Pre Pass의 유용성? ( Hidden Surface Removal )"
date:   2022-04-24
categories: ComputerScience ComputerGraphics
---

모바일 GPU에서는 Depth Pre Pass를 수행하지 마라?               
**모바일 GPU는 기본적으로 커널단에서 Hidden Surface Removal를 수행**하기 때문에 Depth Pre Pass를 수행하면 오히려 성능적으로 손해일 수 있다.                        
**Hidden Surface Removal은 GPU단에서 레스터라이즈 단계 전 가려진 Fragment에 대해서는 연산을 생략해주는 동작**이다.          
( Early - Z Test는 레스터라이즈 단계 후, Fragment 쉐이딩 단계 전 수행된다. )              
                
필자가 확인한 바로는 Mali GPU와 애플의 아이폰에 사용되는 GPU에서는 Hidden Surface Removal이 커널단에서 적용되는 것으로 보인다. 스냅드래곤쪽도 확인 필요...              
            
또한 주의할 것은 **Blending이나, Alpha Test를 수행하는 경우, 픽셀 쉐이더에서 Depth 값을 쓰거나, 픽셀 쉐이더에서 Fragmnet를 버리는 동작이 있는 경우, GPU가 해당 Primitive의 Fragment가 가려질지 여부를 판단하지 못하기 때문에 Hidden Surface Removal의 도움을 받지 못한다.**            
그래서 **모바일 환경의 경우, 위의 상황들을 최소화해야한다.**                                   
                 
refererence : [https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses](https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses), [https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html](https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html), [https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/](https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/), [https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Performance/Performance.html](https://developer.apple.com/library/archive/documentation/3DDrawing/Conceptual/OpenGLES_ProgrammingGuide/Performance/Performance.html)           