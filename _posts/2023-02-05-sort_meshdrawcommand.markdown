---
layout: post
title:  "Sort MeshDrawCommand"
date:   2023-02-05
tags: [UE]
---         
                
언리얼 엔진4의 MeshDrawCommand를 Sorting하는 방식이 인상적이어서 캡쳐해둠..        
<img width="481" alt="MeshDrawCommand_Sort" src="https://user-images.githubusercontent.com/33873804/216826641-095b9e8f-bec4-4b9e-a802-9ed3c225d68f.png">        
BasePass의 경우 Masked 여부(최상위 비트) -> PixelShaderHash -> VertexShaderHash순으로 고려해 MeshDrawCommand를 정렬함. ( ex. Masked가 아닌 것들을 우선 연속해서 Draw한 후 Masked인 것들을 연속해서 Draw함 )                        
                     
[CPP Bit Field](https://en.cppreference.com/w/cpp/language/bit_field)           
