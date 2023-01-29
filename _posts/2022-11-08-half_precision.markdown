---
layout: post
title:  "Half Precision ( GPU )"
date:   2022-11-08
tags: [ComputerGraphics, UE]
---          
                 
언리얼은 기본적(FullPrecision을 강제하지 않는 경우)으로 유저가 작성한(MaterialExpressions) VS 코드(ex. WPO)는 항상 Full Precision으로 처리한다.           
![vs full](https://user-images.githubusercontent.com/33873804/200571664-4b4ce519-a3c4-40e8-a927-045fb0114457.PNG)              
                        
[Half Precision 관련 참고할만한 글](https://solidpixel.github.io/2021/11/23/floats_in_shaders.html)          
                
references : [https://developer.nvidia.com/gpugems/gpugems2/part-iv-general-purpose-computation-gpus-primer/chapter-35-gpu-program-optimization](https://developer.nvidia.com/gpugems/gpugems2/part-iv-general-purpose-computation-gpus-primer/chapter-35-gpu-program-optimization)