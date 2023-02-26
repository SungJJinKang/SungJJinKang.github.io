---
layout: post
title:  "Tile Memory, Memory Less 모드"
date:   2022-04-14
tags: [ComputerGraphics]
---

모바일 GPU는 일반적으로 **Tile Memory**를 가지고 있다.         
Tile Memory는 간단히 설명하면 GPU의 on-chip 메모리로 상대적으로 높은 대역폭, 상대적으로 낮은 Latency, 그리고 일반 비디오 메모리(시스템 메모리)에 비해 전력을 덜 소모하는 특징을 가지고 있다. ( Tile Memory와 반대되는 특성을 가진 시스템 메모리도 존재한다. 모바일 기기에서는 일반적으로 CPU와 GPU가 이 (Physical한) 시스템 메모리를 공유한다. )             
                                    
Metal이나 Vulkan은 RenderPass를 명시적으로 선언할 수 있어 Tile 메모리를 디바이스(비디오) 메모리로 옮길지 여부를 결정할 수 있지만(Tile 메모리에 있는 렌더 패스의 결과물이 다음 렌더 패스에서 필요하지 않은 경우 명시적으로 Tile 메모리의 데이터를 버리는 것(**Memory Less**, Tile Memory의 데이터를 비디오 메모리로 옮기지 않는다) 혹은 Tile Memory의 데이터를 비디어 메모리로 옮기지 않고 이후 렌더 패스에서 그대로 사용하는 것등을 선언할 수 있다. 이를 통해 메모리 대역폭을 절약할 수도 있다.)              
<img width="378" alt="20220414210550" src="https://user-images.githubusercontent.com/33873804/163387255-6d2f3253-5032-478d-8175-62e81764b860.png">         
                 
그러나 OpenGL ES의 경우 RenderPass라는 개념이 없기 때문에 Tile 메모리의 데이터를 비디오 메모리로 옮기지 않고 버리려면 "glClear" 함수를 사용해서 GPU 드라이버에 이를 알릴 수 있다. (OpenGL ES Spec에 Tile 메모리에 대한 내용이 없는거보면 "glClear" 함수 호출시 이러한 동작을 수행해주는지 여부는 GPU 드라이버에 따라 다를 것으로 보임.)             
                      
                           
추가적으로, MSAA같은 경우도 결국 샘플링 개수만큼 픽셀 값을 저장하기 위한 메모리가 N배로 필요하게 되는데 타일 메모리의 사이즈가 MSAA에 필요한 메모리 사이즈보다 커 MSAA를 위한 샘플링 데이터를 타일 메모리에 저장할 수 있다면, 타일 메모리에 MSAA를 위한 샘플링 데이터를 저장한 후 타일 메모리에서 MSAA Resolve를 수행할 수 있다.           
결과적으로는 MSAA를 사용한 경우와 그렇지 않은 경우의 메모리 대역폭 사용량이 동일하게 된다.(타일 메모리에서 MSAA Resolve를 수행하여, 타일 메모리에서 비디오 메모리로 데이터를 옮겨지는 데이터 사이즈는 MSAA를 하지 않은 경우와 같기 때문에...) (타일 메모리는 GPU 코어와 매우 가깝기 때문에 비디오 메모리보다 훨씬 빠르게 데이터를 전송, 접근할 수 있다.)                           
          
Mali GPU에서는 [Pixel Local Storage](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/pixel-local-storage-on-arm-mali-gpus)라는 것을 지원하는데, 간단히 설명하면 쉐이더단에서 Tile Memory에 저장할 데이터를 선언하는 것이다. Pixel Local Storage로 선언된 타입의 데이터는 렌더 패스 동안 Tile Memory에서만 존재하고 비디오 메모리(시스템 메모리)로 복사되지 않고 버려져 불필요한 메모리 대역폭 낭비를 막을 수 있다. 모바일 GPU는 Tile Based Rendering으로 씬에 존재하는 모든 Primitive에 대해 Vertex Shading을 수행 후 각각의 타일(렌더 타겟을 일정 사이즈(Mali GPU 기준 16x16)로 나눈 타일)에 그려질 Primitive를 모은다. 씬 상의 모든 Primitive의 삼각형을 그려질 타일별로 (Geometry Working Set에) 모두 모은 후 (차폐된 Fragment는 버리고) 화면상에 보여질 삼각형들에 대해서만 Fragment Shading 수행한다.(Hidden Surface Removal)       
모바일 GPU에서는 이렇게 씬 상의 모든 삼각형에 대해 Vertex Shading을 한꺼번에 수행해 타일별로 모두 모은 후, 타일별로 Fragment Shading을 한꺼번에 수행한다.          
여기서 타일별로 Fragment Shading을 수행할 때 타일 사이즈만큼의 렌더 타겟을 Tile Memory에 저장한다. Pixel Local Storage는 이 Tile Memory의 저장될 데이터를 프로그래머가 직접 정의하는 것이고..                   
모바일 GPU에서 Deferred 렌더링을 수행할 때 G버퍼 패스와 Resolve 패스에 Pixel Local Storage를 활용하면 G버퍼를 비디오 메모리(시스템 메모리)로 옮기지 않고 Tile Memory에 저장해 Resolve 패스에서 곧 바로 읽으면 메모리 대역폭을 줄일 수 있다. 또한 Tile Memory에 저장된 Pixel Local Storage 데이터는 비디오 메모리로 옮겨지지 않으니(그냥 버려지니) 여기서도 메모리 대여폭을 줄일 수 있다.            
                  
                  
함께 읽으면 좋은 문서 : [https://developer.apple.com/documentation/metal/mtlstoragemode/memoryless](https://developer.apple.com/documentation/metal/mtlstoragemode/memoryless), [https://developer.apple.com/documentation/metal/resource_fundamentals/choosing_a_resource_storage_mode_for_apple_gpus](https://developer.apple.com/documentation/metal/resource_fundamentals/choosing_a_resource_storage_mode_for_apple_gpus), [https://interactive.arm.com/story/the-arm-manga-guide-to-the-mali-gpu/page/2/11](https://interactive.arm.com/story/the-arm-manga-guide-to-the-mali-gpu/page/2/11), [https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/pixel-local-storage-on-arm-mali-gpus](https://community.arm.com/arm-community-blogs/b/graphics-gaming-and-vr-blog/posts/pixel-local-storage-on-arm-mali-gpus), [GPU Framebuffer Memory: Understanding Tiling](https://developer.samsung.com/galaxy-gamedev/resources/articles/gpu-framebuffer.html#Immediate-mode-rasterizers)             
