---
layout: post
title:  "Tile Based Rendering"
date:   2022-04-14
tags: [ComputerGraphics]
---
                
<img width="378" alt="20220414210550" src="https://user-images.githubusercontent.com/33873804/163387255-6d2f3253-5032-478d-8175-62e81764b860.png">             
                  
Metal이나 Vulkan은 RenderPass를 명시적으로 선언할 수 있어 Tile 메모리를 디바이스(비디오) 메모리로 옮길지 여부를 결정할 수 있지만(Tile 메모리에 있는 렌더 패스의 결과물이 다음 렌더 패스에서 필요하지 않은 경우 명시적으로 Tile 메모리의 데이터를 버리는 것을 선언할 수 있다. 이를 통해 메모리 대역폭을 절약할 수도 있다.),               
               
그러나 OpenGL ES의 경우 RenderPass라는 개념이 없기 때문에 Tile 메모리의 데이터를 비디오 메모리로 옮기지 않고 버리려면 "glClear" 함수를 사용해서 GPU 드라이버에 이를 알릴 수 있다. (OpenGL ES Spec에 Tile 메모리에 대한 내용이 없는거보면 "glClear" 함수 호출시 이러한 동작을 수행해주는지 여부는 GPU 드라이버에 따라 다를 것으로 보임.)             
                      
                           
추가적으로, MSAA같은 경우도 결국 샘플링 개수만큼 픽셀 값을 저장하기 위한 메모리가 N배로 필요하게 되는데 타일 메모리의 사이즈가 MSAA에 필요한 메모리 사이즈보다 커 MSAA를 위한 샘플링 데이터를 타일 메모리에 저장할 수 있다면, 타일 메모리에 MSAA를 위한 샘플링 데이터를 저장한 후 타일 메모리에서 MSAA Resolve를 수행할 수 있다.           
결과적으로는 MSAA를 사용한 경우와 그렇지 않은 경우의 메모리 대역폭 사용량이 동일하게 된다.(타일 메모리에서 MSAA Resolve를 수행하여, 타일 메모리에서 비디오 메모리로 데이터를 옮겨지는 데이터 사이즈는 MSAA를 하지 않은 경우와 같기 때문에...) (타일 메모리는 GPU 코어와 매우 가깝기 때문에 비디오 메모리보다 훨씬 빠르게 데이터를 전송, 접근할 수 있다.)                           
                    

함께 읽으면 좋은 문서 : [https://developer.apple.com/documentation/metal/resource_fundamentals/understanding_gpu_family_4/about_tile_shading](https://developer.apple.com/documentation/metal/resource_fundamentals/understanding_gpu_family_4/about_tile_shading), [https://developer.arm.com/documentation/102662/0100/Tile-based-GPUs](https://developer.arm.com/documentation/102662/0100/Tile-based-GPUs)                  