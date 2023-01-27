---
layout: post
title:  "언리얼 엔진4 Masked vs Translucent ( 모바일 GPU에서의 Alpha Test, Alpha Blend )"
date:   2022-04-30
tags: [UE]
---

[Translucent vs Masked Rendering in Real-Time Applications](https://developer.oculus.com/blog/translucent-vs-masked-rendering-in-real-time-applications/?locale=ko_KR)        
         
위의 글에서는 오큘러스에서 캐릭터를 그리기 위해 시도했던 Translucent 렌더링과 Masked 렌더링을 비교해서 소개하고 있다.               

결론부터 말하면 Custom Depth 값을 활용하여 Transclucent Pass에서 그렸을 때보다,     
Early Depth Pre Pass의 결과 값을 활용한 Masked Rendering 수행하여 성능을 높일 수 있었다.       

-------------------------------            

Masked 렌더링은 Base(Opaque) Pass에서 수행된다. 비용이 상대적으로 저렴하다.                     

반면 **Translucent 오브젝트**를 렌더링하기 위해서는 **Sort 작업이 필요한데 여기 드는 비용 또한 적지 않다.**              
게다가 그것이 정확하지도 않아 렌더링시 이상하게 그려질 수도 있다.            
Depth Sort의 정확도를 높이려면 삼각형 단위의 Sort가 들어가면 되지만 이는 비용이 매우 크다.                 

----------------------------------

**모바일에서는 Masked 렌더링도 비용이 상대적으로 비싸다.**                          
이는 [Hidden Surface Removal](https://sungjjinkang.github.io/mobile_gpu_hidden_surface_removal)와 연관이 되어 있는데, **모바일 GPU의 경우 레스터라이저 단계 전에 GPU 커널단에서 가려진 Fragment들에 대한 연산을 생략**해주는 동작을 수행한다.              
그런데 **Masked 렌더링은 픽셀 쉐이더 단계에서 Alpha - Test를 수행하여 Fragment를 버리는 방식으로 동작**하기 때문에 "Hidden Surface Removal"의 도움을 받지 못한다. ( 어느 Fragment의 Depth 값이 버려질지 모르기 때문... 마찬가지로 Early - Z Test도 픽셀 쉐이더에서 Depth 값을 버리는 동작이 있으면 적용되지 않는다. )         
자세한건 [이 글](https://sungjjinkang.github.io/mobile_gpu_hidden_surface_removal)을 참고하자.....             

[또 다른 참고할만한 글1](https://developer.arm.com/documentation/102576/0100/Transparency-best-practise), [2](https://blog.katastros.com/a?ID=00600-4a63ac52-bd9e-42c7-8272-30a76b4a3346), [3](https://bgolus.medium.com/anti-aliased-alpha-test-the-esoteric-alpha-to-coverage-8b177335ae4f), [4](https://developer.arm.com/documentation/101897/0200/shader-code/discards)
              