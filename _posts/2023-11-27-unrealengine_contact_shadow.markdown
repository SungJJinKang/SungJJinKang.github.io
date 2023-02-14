---
layout: post
title:  "언리얼 엔진 Contact Shadow"
date:   2023-11-27
tags: [UE]
---            

일반적인 Shadow Map을 활용한 쉐도우 연산 :               
라이팅 연산 대상의 픽셀의 WS 원점을 라이트를 기준으로 Project한 후 그 Z값을 ShadowMap에서 샘플링한 Depth 값과 비교하여, 값이 더 큰 경우 다른 오브젝트에 의해 가려졌다고(Occlude) 판단하여 Shadow 값을 라이팅 연산에 포함시킴.                 
                   
Contact Shadow :                      
일반적인 Shadow Map을 활용한 쉐도우 연산은 ShadowMap과 단 한번만 비교하여 Shadow 여부를 판단하는데 비해, Contact Shadow는 라이트 원점에서 라이팅 대상 픽셀의 원점으로 RayMarching을 수행하여 더 정밀한 쉐도우 연산을 수행. 다만 RayMarching에서 사용하는 Depth 값도 스크린 스페이스를 기준으로 ShadowMap에서 샘플링해서 가져오기 때문에 아티팩트 가능성 존재. 쉽게말하면 "일반적인 Shadow Map을 활용한 쉐도우 연산"에서 수행하는 ShadowMap에서 샘플링한 Depth 값 비교 동작을, 라이트 원점에서 라이팅 연산 대상 픽셀 지점 사이의 중간 지점들에 대해서도 수행한다고 생각하면 된다. ( 정확도를 높이기 위해 )                                              
            
[Contact Shadow - Unreal Engine Documents](https://docs.unrealengine.com/5.0/en-US/contact-shadows-in-unreal-engine/)                  
                 
[Ray Marching](https://adrianb.io/2016/10/01/raymarching.html)                     
![figure3](https://user-images.githubusercontent.com/33873804/204130892-89948484-1402-4518-9b23-ef9ffb2408e9.png)                
                                 
                                      
"ContactShadowNonShadowCastingIntensity"는 Contact Shadow Casting 셋팅이 Disable되어 있는 오브젝트에 대해서 Contact Shadow를 적용하고자 할 때 그 적용의 강도(Intensity) 값. 콘솔 변수로 기본적으로는 0이다.               
                
                                       
Shaders\Private\DeferredLightingCommon.ush 파일 참고              
<img width="558" alt="ContactShadow" src="https://user-images.githubusercontent.com/33873804/204130610-04d36c2d-cfcb-483d-925a-8c42d5dd3107.png">                    
<img width="489" alt="ContactShadow2" src="https://user-images.githubusercontent.com/33873804/204130611-5f08e06f-e0f1-4076-8369-feae13d96da2.png">                     


