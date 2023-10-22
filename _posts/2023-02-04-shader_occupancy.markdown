---
layout: post
title:  "Shader Occupancy란 무엇이고, 왜 그것에 관심을 기울여야 하는가(번역)"
date:   2023-02-04
tags: [ComputerGraphics, Recommend]
---         
  
Xcode GPU Frame Capture를 보던 중 Occupancy라는 데이터가 있어 이것이 뭔지 궁금해서 찾아보았다.       
이 글은 [WHAT IS SHADER OCCUPANCY AND WHY DO WE CARE ABOUT IT?](https://interplayoflight.wordpress.com/2020/11/11/what-is-shader-occupancy-and-why-do-we-care-about-it/)을 번역한 글입니다. 약간의 의역이 포함되어 있습니다.

-----------------------               


GPU는 64개 혹은 32개의 픽셀 혹은 버텍스들을 [Warp](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline) 단위로 묶어서 실행한다. ( 정확히는 64개 혹은 32개의 스레드들을 묶은 것이 Warp다. GPU는 이 32개의 스레드를 묶어 동시에 병렬적으로 처리를한다. GPU의 특성상 Warp 내의 스레드들은 모두 동시점에 동일한 명령어를 수행하여야 한다. ) GPU 코어가 메모리로부터 데이터를 읽어올 때(쉐이더에서 메모리를 읽는 명령어를 실행하는 경우) 상당한 Latency가 발생한다. 즉 메모리로부터 데이터를 읽어오는 명령어를 실행하고 실제 데이터가 GPU 코어에 도착하기까지 상당한 시간이 걸린다는 것이다. GPU 캐시에 데이터가 있다면 다행이지만 없다면 비디오 메모리에 접근하여야 하니 Latency는 더 커진다. 이렇게 데이터가 도착하기를 기다리는 동안 GPU는 Stall될 수 있다. ( 당연히 GPU를 갈구지 못하고 Stall되게 한다는 것은 성능상 좋지 않다... )         

메모리로부터 데이터를 읽어오는 동작을 가진 쉐이더 코드를 컴파일하면 쉐이더 컴파일러는 컴파일된 쉐이더 코드 맨 앞쪽에 "데이터를 요청하는 명령어"를 넣는다. 실제 데이터를 사용해야할 때 "데이터를 요청하는 명령어"를 수행하는 것이 아니라 미리 "데이터를 요청하는 명령어(EX. 텍스처 읽기)"를 수행하고 이후 해당 데이터를 사용하기 전 데이터가 사용 가능한지(데이터가 도착했는지)를 검사한다. 만약 데이터가 사용 가능하지 않은 경우 GPU 코어는 데이터가 도착하기를 기다린다. ( 쉐이더 컴파일러가 이렇게 쉐이더 코드를 컴파일하는 이유는 메모리에 데이터를 미리 미리 요청해서 실제 데이터를 사용할 때 GPU 코어가 Stall되지 않게 하기 위함이다. GPU코어가 데이터를 요청하는 명령어를 수행하면 데이터가 도착할 때까지 Block되지 않고 그냥 다음 명령어를 바로 수행한다. )

```
Texture1D	Materials : register(t0);
 
PSInput VSMain(VSInput input)
{
   PSInput result = (PSInput)0; 
   result.position = mul(WorldViewProjection, float4(input.position.xyz, 1));
   float uvScale = Materials[MeshIndex].x; <--- 메모리로부터 데이터 읽기
   result.uv = uvScale * input.uv;  <--- 읽어온 데이터를 사용
   return result;
}
위의 쉐이더 코드는 아래의 ISA로 컴파일된다.

  s_swappc_b64  s[0:1], s[0:1]                          
  s_buffer_load_dword  s0, s[20:23], 0x00               
  s_waitcnt     lgkmcnt(0)                              
  v_mov_b32     v0, s0                                  
  v_mov_b32     v1, 0                                   
  image_load_mip  v0, v[0:3], s[8:15] unorm  <--- 메모리로부터 데이터 읽기      
  s_buffer_load_dwordx8  s[0:7], s[16:19], 0x40         
  s_buffer_load_dwordx8  s[8:15], s[16:19], 0x60        
  s_waitcnt     lgkmcnt(0)                              
  v_mul_f32     v1, s4, v5                              
  v_mul_f32     v2, s5, v5                              
  v_mul_f32     v3, s6, v5                              
  v_mul_f32     v5, s7, v5                              
  v_mac_f32     v1, s0, v4                              
  v_mac_f32     v2, s1, v4                              
  v_mac_f32     v3, s2, v4                              
  v_mac_f32     v5, s3, v4                              
  v_mac_f32     v1, s8, v6                              
  v_mac_f32     v2, s9, v6                              
  v_mac_f32     v3, s10, v6                             
  v_mac_f32     v5, s11, v6                             
  v_add_f32     v1, s12, v1                             
  v_add_f32     v2, s13, v2                             
  v_add_f32     v3, s14, v3                             
  v_add_f32     v4, s15, v5                             
  exp           pos0, v1, v2, v3, v4 done               
  v_mov_b32     v5, 0                                   
  exp           param0, v5, v5, v5, off                 
  s_waitcnt     vmcnt(0)                     <--- !!!!!!          
  v_mul_f32     v6, v8, v0                   <--- 읽어온 데이터를 사용          
  v_mul_f32     v0, v9, v0                              
  exp           param1, v5, v5, v5, off                 
  exp           param2, v6, v0, off, off                
  exp           param3, v5, v5, v5, off                 
  s_endpgm                                              
```

GPU 코어는 쉐이더 프로그램 초반부에 텍스쳐를 로드하는 명령어를 수행하고 이후 명령어들을 계속 수행한다. 그리고 실제로 데이터(uvScale)를 사용하기(v_mul_f32     v6, v8, v0) 직전 wait 명령어(s_waitcnt     vmcnt(0))를 수행하여 앞서 요청한 데이터가 도착했는지를 확인한다. 데이터가 도착한 경우 뒤의 곱셈 연산을 바로 수행하지만, 데이터가 도착하지 않은 경우 GPU 코어는 데이터가 도착할 때까지 Stall한다.          
요약하자면 GPU코어는 데이터를 요청할 때 Stall하는 것이 아닌, 데이터를 실제 사용하기 직전에 Stall한다는 것이다.       

GPU 코어가 Stall되어 노는 시간을 최대한 줄이기 위해(최대한 GPU를 바쁘게 굴리기 위해) GPU는 Stall된 Warp 대신 다른 Warp를 실행한다.      
(여기서 뭔 말인지 이해가 되지 않을 수 있어 조금 더 설명하자면, Nvidia Fermi 아키텍쳐 기준 GPU는 여러 SM(Streaming Multiprocessor)를 가지는데 이 SM은 32개의 코어를 가지고 전용 캐시, 텍스쳐 메모리 공간을 가지는 등 하나의 Warp를 실행하기 위한 구조로 Warp 단위로 연산을 수행한다. 자세한건 [Life of a triangle - NVIDIA's logical pipeline](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline)을 참고해주세요 )       
메모리 읽기로 GPU 코어가 Stall되는건 흔하기 때문에 GPU는 이 Warp를 여러 개 준비해두고 Warp를 처리하는 중 Stall이 발생하면 다른 Warp를 배정해(Nvidia 기준 SM에는 Warp Scheduler라는 장치가 있다.) GPU코어가 Idle 상태로 대기하는 시간을 줄인다.(GPU 코어를 쉬지 못하게 계속 갈군다...) ( CPU의 하이퍼스레딩과 비슷한 개념인듯. 메모리 Stall이 발생하면 다른 스레드를 코어에 배정한다. )          
GPU가 얼마나 많은 Warp를 준비(대기) 시켜둘 수 있는지는 해당 "쉐이더 코드의 실행에 필요한 벡터 레지스터(VGPR)"의 수에 달려있다. 쉐이더 코드의 VGPR 개수는 쉐이더 컴파일시 결정된다. GPU 아키텍쳐마다 다르지만 하나의 SM이 가질 수 있는 VGPR의 제한 수량이 정해져 있기 때문에, 쉐이더에 의해 사용되는 VGPR의 수에 따라 GPU가 준비(대기) 시켜둘 수 있는 Warp 수도 달란진다. 쉐이더 코드가 더 많은 VGPR을 필요로 할 수록 준비해둘 수 있는 Warp의 수는 줄어드는 것이다.      
             
Nvidia쪽 문서에 따르면 아래 사진에서 보이는 SM 내부의 Register File(레지스터의 데이터를 저장하는 공간) 사이즈에 따라 대기 시켜둘 수 있는 Warp의 수가 달라진다. Warp를 처리하는 중 Stall이 발생해 다른 Warp를 SM에 배정하려면 기존 처리 중이던 Warp의 레지스터 데이터를 저장해두어야 한다. 그래야 다시 해당 Warp가 배정되었을 때 기존에 수행하던 처리를 이어서 할 수 있으니.. ( OS의 컨테스트 스위칭과 같다 ) 그래서 Register File의 사이즈가 클 수록 더 많은 Warp의 레지스터 데이터를 저장해둘 수 있고 그 말은 SM이 더 많은 Warp를 준비(대기) 시켜둘 수 있다는 의미이다.             
![fermipipeline_sm](https://user-images.githubusercontent.com/33873804/216777763-9b7a23f4-e8e1-409f-8e05-133858ce13ae.png)              
                
아래 사진을 보면 쉐이더 코드의 VGPR 개수에 따라 SM이 준비(대기) 시켜둘 수 있는 Warp(AMD GCN 아키텍쳐에서는 Wave라고 부름)의 개수가 나와있다.    
![BkyOju4CYAAzvAS](https://user-images.githubusercontent.com/33873804/216778000-c7ecae90-9223-49dd-8b0e-817db1dd357c.jpg)        
        
만약 너가 32개의 VGPR를 필요로하는 쉐이더 코드로 렌더링을 수행한다면 SM은 8개의 Warp를 준비(대기)해둘 수 있다. 이것은 쉐이더 코드 실행 중 메모리 Stall이 발생하여도 충분히 많은 Warp가 대기 중이기 때문에 다른 Warp를 SM에 배정하여 GPU 코어의 Stall을 줄일 수 있다는 의미이다.           
그리고 이 **"특정 쉐이더 코드를 실행할 때 SM이 대기 시켜둘 수 있는 Warp의 수(해당 쉐이더의 VGPR에 따라 다르다) / GPU 아키텍쳐 상 SM이 대기 시켜둘 수 있는 Warp의 최대 개수(쉐이더 코드의 VPGR이 1개일 때)"를 쉐이더 Occupancy라고 부른다.**                   
만약 128개의 VGPR를 필요로하는 쉐이더 코드로 렌더링을 수행한다면 SM은 단 하나의 Warp만을 준비해둘 수 있고, 이 말은 쉐이더 코드 실행 중 메모리 Stall 발생하여도 교체할 수 있는 Warp가 없기 때문에 GPU 코어는 메모리로부터 데이터가 도착할 때까지 Idle 상태로 대기하여야 한다는 의미이다. (GPU 코어를 그냥 놀게 냅두다니... 얼마나 큰 자원 낭비인가...)           
          
그럼 낮은 쉐이더 Occupancy 항상 나쁜 것일까?             
정답을 그렇지 않다이다.           
위의 컴파일된 쉐이더 코드에서도 보이듯이 쉐이더 컴파일러는 컴파일시 명령어의 순서를 재배치하여 "메모리에 데이터를 요청하는 명령어"와 "실제 데이터를 사용하는 명령어"를 최대한 떨어트려두어 최대한 실제 데이터를 사용할 때 데이터가 준비되어 있게끔 한다. 그래서 쉐이더 Occupancy가 낮다고 반드시 메모리 Stall이 발생하는 것은 아니다.(쉐이더 컴파일러의 노력 덕분에 데이터가 실제로 사용될 때 이미 해당 데이터가 준비되어 있을 가능성이 적지 않다는 것이다.) 그래서 쉐이더 Occupancy가 아니라 메모리 읽기(ex. 텍스쳐 읽기)로 인해 Stall이 실제 발생하였는지(SM에 다른 Warp를 배정하였음에도, 혹은 다른 Warp도 데이터가 준비되지 않아 GPU 코어가 Stall되었는지)를 보여주는 지표를 기준으로 최적화를 하여야 한다. 실제로 GPU 코어가 Stall되었다는 지표를 보인다면 쉐이더 Occupancy를 높이는 최적화(쉐이더 코드를 수정하여 VGPR을 줄여보는)를 수행한다면 성능 향상을 얻을 수 있을 것이다.                                                                         
           
그럼 높은 쉐이더 Occupancy는 항상 좋은 것일가?            
정답은 마찬가지로 그렇지 않다이다.         
만약 쉐이더 코드가 많은 메모리 읽기 명령어를 가지고 있고 많은 메모리 트래픽을 요구한다면, SM에 대기 중인 Warp들은 제한된 쉐이더 캐시 사이즈를 놓고 서로 싸울 것이다. 많은 수의 Warp들이 메모리에 데이터를 계속 요청하면서 Warp들이 서로의 캐시 데이터를 무효화한다는 것이다. SM에 대기 중인 Warp들이 많으니 그 만큼 연산되는 Warp도 더 자주 교체되고, 더 많은 수의 Warp들이 데이터를 메모리로부터 읽어오려 하니 한정된 메모리(캐시) 자원이 비효율적으로 사용된다는 것이다. 그리고 이로 인해 성능이 저하될 수 있다는 것이다. 높은 쉐이더 Occupancy가 항상 좋은 것만은 아니라는 것이다..          
                                        
그리고 쉐이더가 많은 수의 메모리 읽기 명령어를 수행하는 경우 쉐이더 Occupancy가 높은 것(쉐이더의 VGPR이 낮은 경우)이 성능에 부정적일 수 있는데.       
쉐이더가 많은 수의 메모리 읽기 명령어를 수행하는 경우 쉐이더 컴파일러는 최적화 차원에서 사용되는 VGPR(벡터 레지스터)의 수을 낮추기 위해(쉐이더 컴파일러는 VGPR을 낮추기 위한 코드 최적화를 수행한다.) 복수의 메모리 요청 명령어 연속적으로 배치하기도 하는데(ex. 메모리 읽기 명령어를 수행하고, 데이터를 기다리고, 도착한 데이터를 레지스터에 저장하고, 그것을 사용하고, 그리고 바로 다음 메모리 요청 동작으로 도착한 데이터를 다시 해당 레지스터에 저장해 레지스터를 재사용한다. -> 하나의 레지스터로 복수의 "메모리 읽기", "읽은 후 그 데이터로 연산을 하는 동작"을 처리함.).. 쉐이더 컴파일러가 레지스터를 재사용하여 VGPR을 낮추기 위해 메모리를 읽는 명령어을 연속되게 배치하지만 그 수가 많은 경우 메모리 Stall이 발생할 수 있다.               
            
**일반적으로는 쉐이더 Occupancy를 높이는 것이 메모리 Latency를 줄여주지만, 항상 그런 것은 아니니 프로파일링하고 검증하라!!**            
                  

reference : [https://interplayoflight.wordpress.com/2020/11/11/what-is-shader-occupancy-and-why-do-we-care-about-it/](https://interplayoflight.wordpress.com/2020/11/11/what-is-shader-occupancy-and-why-do-we-care-about-it/), [https://docs.nvidia.com/gameworks/content/developertools/desktop/analysis/report/cudaexperiments/kernellevel/achievedoccupancy.htm](https://docs.nvidia.com/gameworks/content/developertools/desktop/analysis/report/cudaexperiments/kernellevel/achievedoccupancy.htm), [https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline](https://developer.nvidia.com/content/life-triangle-nvidias-logical-pipeline)
