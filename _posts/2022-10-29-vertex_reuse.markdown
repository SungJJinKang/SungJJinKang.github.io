---
layout: post
title:  "메시 버텍스, 인덱스 버퍼 최적화 ( 버텍스 캐시, 대역폭, Post Transform Vertex Cache, GPU 메모리 캐시, 버텍스 재사용 )"
date:   2022-10-29
tags: [ComputerGraphics, Recommend]
---          
                        
그래서 버텍스 재사용 관련 내용을 조사하다 여러 가지 지식들을 습득할 수 있었다.                     
                
1. DrawIndexed를 사용해라. 불필요한 버텍스 재연산을 줄일 수 있다.                    
2. 우선 불필요하게 중복되는 버텍스를 제거한다. ( 버텍스 버퍼 상에서 동일한 위치(동일한 값)의 버텍스가 중복해서 들어가는 것들을 찾아 제거한다. Weird Vertex ), 이는 GPU 대역폭 사용량을 낮추어 줄 것이다.                                    
3. 불필요한 인덱스를 제거하자. GPU 파이프라인은 4개의 ( Mali-G78 기준 ) 연속된 Index를 그룹핑해서 처리하는데 이 연속된 4개의 Index 중에 쓸모 없는 ( 어떠한 버텍스도 가르키지 않는 쓰레기 값 ) 값이 들어 있으면, 그 쓸모 없는 인덱스에 대한 버텍스 쉐이딩 연산이 발생한다. 이러한 쓰레기 값의 인덱스를 찾아 제거하자.                         
4. Index Buffer를 재정렬하여 Post Transform Vertex Cache ( 버텍스 쉐이딩 처리된 버텍스에 대한 캐시 ) 히트율을 높여라 ( 동일한 인덱스가 최대한 근접하게 배치되게 인덱스 버퍼를 정렬하라 ) . Post Transform Vertex Cache는 최근에 쉐이딩 처리한 버텍스의 clip space(Vertex Shader 처리 후)에서의 위치 값을 가지고 있다. 만약 캐시에서 있는 버텍스에 대한 버텍스 쉐이더 처리 요청이 들어온다면 이 캐시를 활용하여 해당 버텍스에 대한 버텍스 쉐이딩 연산을 생략할 수 있다. 이 Post Transform Vertex Cache의 사이즈는 매우 작기 때문에 인덱스 버퍼를 정렬하여 동일한 버텍스가 곧 바로 처리되게 하는 것이 중요하다. ( [ARM Mali GPU Best Practice](https://armkeil.blob.core.windows.net/developer/Arm%20Developer%20Community/PDF/Arm%20Mali%20GPU%20Best%20Practices.pdf)의 20페이지에도 "Post Transform Cache"에 대한 내용이 있는 것을 보면 모바일 GPU에도 해당되는 얘기이다. )                                  
5. GPU 메모리 캐시 히트율을 높이기 위해 인덱스 버퍼를 기준으로 연속되어 그려지는 버텍스들을 메모리 상에 인접되게 위치하도록 만들어라.                      
                      
                       
-----------------------------           
               
[버텍스 캐시 재방문. 현대 GPU에서 버텍스 처리 이해 및 최적화 - 2018년 논문](https://arbook.icg.tugraz.at/schmalstieg/Schmalstieg_351.pdf)             
[버텍스 캐시 최적화 여전히 유효한가?](https://www.reddit.com/r/opengl/comments/js9a9t/is_vertex_cache_optimization_still_a_thing/)              
현대 GPU에는 버텍스 캐시라는 개념이 존재하지 않는다고 한다.(NVIDIA 공식 문서에서도 버텍스 캐시, Post Transform Cache와 관련된 내용은 안보이네요) 그러나 GPU가 여러 버텍스를 묶고(Batch), 그 묶음 내에서는 중복된 인덱스를 없애 버텍스 캐시와 비슷한 효과를 낸다고 한다(Batch 내에서는 동일한 버텍스를 중복 연산하지 않는다 ).           
           
[버텍스 재사용율 - 스터디 발표 자료](https://docs.google.com/presentation/d/13tLXdeyRdsjksKXQ2ILItW85dZGv4rnXUJlUYEKyKdQ/edit?usp=sharing)         
                  
---------------------               
           
references :            
[SHADED VERTEX REUSE ON MODERN GPUS](https://interplayoflight.wordpress.com/2021/11/14/shaded-vertex-reuse-on-modern-gpus/)              
[Vertex Shader Tricks by Bill Bilodeau - AMD at GDC14](https://www.slideshare.net/DevCentralAMD/vertex-shader-tricks-bill-bilodeau)            
[https://github.com/zeux/meshoptimizer/blob/master/README.md](https://github.com/zeux/meshoptimizer/blob/master/README.md)                 
[https://www.reddit.com/r/opengl/comments/js9a9t/is_vertex_cache_optimization_still_a_thing/](https://www.reddit.com/r/opengl/comments/js9a9t/is_vertex_cache_optimization_still_a_thing/)                  
[https://developer.arm.com/documentation/102626/0100/?lang=en](https://developer.arm.com/documentation/102626/0100/?lang=en)         
[https://developer.nvidia.com/docs/drive/drive-os/latest/linux/sdk/common/topics/graphics_content/Geometry28.html](https://developer.nvidia.com/docs/drive/drive-os/latest/linux/sdk/common/topics/graphics_content/Geometry28.html)          
[https://gpuopen.com/archived/tootle/](https://gpuopen.com/archived/tootle/)        
