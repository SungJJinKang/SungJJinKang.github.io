---
layout: post
title:  "모바일 GPU에서 Depth Pre Pass의 유용성? ( Early Z, Hidden Surface Removal )"
date:   2022-04-24
categories: ComputerScience ComputerGraphics
---

모바일 GPU에서는 Depth Pre Pass를 수행하지 마라?               
모바일 GPU는 기본적으로 커널단에서 Hidden Surface Removal를 수행하기 때문에 Depth Pre Pass를 수행하면 오히려 성능적으로 손해일 수 있다.                        
                
필자가 확인한 바로는 Mali GPU와 애플의 아이폰에 사용되는 GPU에서는 Hidden Surface Removal이 커널단에서 적용되는 것으로 보인다. 스냅드래곤쪽도 확인 필요...             
                 
refererence : [https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses](https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses), [https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html](https://docs.unity3d.com/2019.1/Documentation/Manual/iphone-Hardware.html), [https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/](https://metalkit.org/2020/07/03/wwdc20-whats-new-in-metal/)           