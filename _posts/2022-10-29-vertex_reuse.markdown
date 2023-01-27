---
layout: post
title:  "메시 버텍스, 인덱스 버퍼 최적화 ( 대역폭, Post Transform Vertex Cache, GPU 메모리 캐시 )"
date:   2022-10-29
tags: [ComputerGraphics]
---          
                 
최근 업무 중 버텍스 최적화 관련 이슈를 살펴보면서 버텍스 재사용이라는걸 알았다.           
그래서 조금 조사해보았고 여러 가지 사실들을 알았다.           
                
1. DrawIndexed를 사용해라. 불필요한 버텍스 재연산을 줄일 수 있다.                    
2. 우선 불필요하게 중복되는 버텍스를 제거한다. ( 버텍스 버퍼 상에서 동일한 위치(동일한 값)의 버텍스가 중복해서 들어가는 것들을 찾아 제거한다. Weird Vertex ), 이는 GPU 대역폭 사용량을 낮추어 줄 것이다. ( 회사에서 진행하던 프로젝트에서는 캐릭터를 Scale up해서 한번 더 그리는 방식으로 Outline을 그렸는데 이로 인해 인접한 버텍스를 하나로 만드는데 어려움이 있었다. )                        
3. 불필요한 인덱스를 제거하자. GPU 파이프라인은 4개의 ( Mali-G78 기준 ) 연속된 Index를 그룹핑해서 처리하는데 이 연속된 4개의 Index 중에 쓸모 없는 ( 어떠한 버텍스도 가르키지 않는 쓰레기 값 ) 값이 들어 있으면, 그 쓸모 없는 인덱스에 대한 버텍스 쉐이딩 연산이 발생한다. 이러한 쓰레기 값의 인덱스를 찾아 제거하자.                         
4. Post Transform Vertex Cache ( 버텍스 쉐이딩 처리된 버텍스에 대한 캐시 ) 캐시 히트율을  Index Buffer를 재정렬하라. Post Transform Vertex Cache는 최근에 쉐이딩 처리한 버텍스의 clip space()Vertex Shader 처리 후)에서의 위치 값을 가지고 있다. 만약 캐시에서 있는 버텍스에 대한 버텍스 쉐이더 처리 요청이 들어온다면 이 캐시를 활용하여 해당 버텍스에 대한 버텍스 쉐이딩 연산을 생략할 수 있다. 이 Post Transform Vertex Cache의 사이즈는 매우 작기 때문에 인덱스 버퍼를 정렬하여 동일한 버텍스가 곧 바로 처리되게 하는 것이 중요하다.                
5. GPU 메모리 캐시 히트율을 높이기 위해 인덱스 버퍼를 기준으로 연속되어 그려지는 버텍스들을 메모리 상에 인접되게 위치하도록 만들어라.                      
                      
                         
references :            
[SHADED VERTEX REUSE ON MODERN GPUS](https://interplayoflight.wordpress.com/2021/11/14/shaded-vertex-reuse-on-modern-gpus/)              
[Vertex Shader Tricks by Bill Bilodeau - AMD at GDC14](https://www.slideshare.net/DevCentralAMD/vertex-shader-tricks-bill-bilodeau)            
[https://github.com/zeux/meshoptimizer/blob/master/README.md](https://github.com/zeux/meshoptimizer/blob/master/README.md)                 
[https://www.reddit.com/r/opengl/comments/js9a9t/is_vertex_cache_optimization_still_a_thing/](https://www.reddit.com/r/opengl/comments/js9a9t/is_vertex_cache_optimization_still_a_thing/)                  
[https://developer.arm.com/documentation/102626/0100/?lang=en](https://developer.arm.com/documentation/102626/0100/?lang=en)               