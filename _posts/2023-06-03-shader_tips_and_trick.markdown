---
layout: post
title:  "SHADER TIPS AND TRICKS(번역)"
date:   2023-06-03
tags: [ComputerGraphics, Recommend, UE]
---          
             
이 글은 [SHADER TIPS AND TRICKS](https://interplayoflight.wordpress.com/2022/01/22/shader-tips-and-tricks/)를 번역한 글입니다. 약간의 의역이 있을 수 있습니다.        
추가적으로 이해를 돕기 위해 필자가 코드블록에 각주를 추가해두었습니다.       
          
언리얼 엔진을 사용하는 경우 매터리얼 그래프 에디터를 사용해 쉐이더 코드를 작성하고, 쉐이더 코드를 직접 작성(.ush...)하는 경우에도 DXC/SPRIV 쉐이더 컴파일러를 거치며 컴파일러단에서 최적화되는 부분이 있기 때문에 직접 작성하는 쉐이더 코드와 실제 GPU에 전달되는 쉐이더 코드는 차이가 있을 수 있다. 그렇지만 아래에서 소개되는 여러 최적화 팁들을 알고 쉐이더 코드를 작성하거나 매터리얼 에디터를 사용하면, 명시적으로 최적화된 코드를 작성할 수 있기 때문에 비용이 매우 큰 것이 확인된 쉐이더들을 선별적으로 최적화할 때는 도움이 되지 않을까 싶다.                      
           
---------------------------------            

이 글에서 나는 여러 쉐이더 팁, 트릭들을 랜덤한 순서로 소개할 것이다. 대부분은 성능 개선과 관련된 것들이고, 약간은 GCN/RDNA 아키텍처, DirectX에 편중된 내용들이다. 본격적으로 시작하기 전 몇가지 주의해야할 것들을 얘기해보면..           
```
Radeon 계열 아키텍처를 기준으로 설명한 글이지만 NVIDIA쪽 아키텍처도 큰 틀에서는 비슷하지 않을까 싶다. 물론 디테일한 부분에서는 다르겠지만...       
```
    
### 주의할 점        
1. 성능 관련 팁, 조언들에 대해서는 항상 곧이곧대로 받아들이지 말고 약간의 의구심을 가져라. 그러한 팁들 대부분은 "모범 사례"는 맞지만, 그렇다고 무조건적으로 맹신해서는 안된다.         
2. 성능 향상 전/후 항상 프로파일링을 해라. 상식적으로 성능 향상이 당연해보이는 것들 실제로 프로파일링해보면 그렇지 않을 수도 있다.             
3. 타깃 플랫폼/하드웨어와 그것들의 특징을 이해하라. 예를 들면 노트북에 사용되는 GPU는 상당히 성능적으로 우수하지만, 데스크톱 GPU에 비해 메모리 대역폭이 낮다.        
             
### 팁/조언      
1. 만약 텍스처 필터링이 필요하지 않다면 텍스처나 렌더타겟을 읽을 때는 "포인트 샘플링을 수행하는 Sample()"보다는 "Load()"를 사용하라.      
```
Sample의 경우 Mipmapping 등 추가적인 동작들을 수행하기 떄문에 이러한 것들이 불필요한 경우 "Load"를 사용하는 것이 성능적으로 유리하다.
```
2. 텍스처 유닛은 텍스처 샘플링시 몇몇 [비교 연산자](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_filter)(EX. min/max)를 지원하는데 그것들은 성능 향상, VGPR 감소해 도움이 됩니다.
```
VGPR이란 : https://sungjjinkang.github.io/shader_occupancy
```
3. 더 많은 "Fixed Function Units"을 사용하기 위해서는, 쉐이더에서의 "linearisation"를 피하기 위해 텍스처를 sRGB로 생성하는 것이 좋다.       
4. 단일 채널 샘플링을 할 때는 "Sample()" 대신 "Gather4()"를 사용하라. 그것이 메모리 대역폭을 줄여주지는 않지만, 텍스처 유닛으로 전달되는 데이터량을 줄일 것이고 캐시 히트률을 높이고, 주소 참조에 필요한 VGPR을 줄일 것이다.
```
"Gather4()"는 4개의 텍셀의 R채널 값들을 float4로 반환한다. 이는 주로 "bilinear 필터링"에 사용된다. 만약 G채널의 값을 얻고 싶다면 "GatherGreen()을 사용하다 ( "GatherBlud(), "GatherAlpha"도 있다 )
```
5. Input, Output 명령어 Modifiers들을 사용하라. 쉐이더에서 "output에 대한 saturate(), x2, x4, /2, /4" 그리고 "Input에 대한 abs(), negate"와 같은 Modifier들은 비용이 거의 제로이다. 예를들면 "saturate(4*(x*-abs(z)-abs(y)))" GCN 아키텍처에서는 하나의 명령어에 불과하다. "v_fma_f32 v1, v2, -abs(v3), -abs(v0) mul:4 clamp"
6. "-4.0, -2.0, -1.0, 0.0, 1.0, 2.0, 4.0 and 1.0/(2.0*pi)"와 같은 상수들은 명령어 코드에 직접 쓰이기 때문에 추가적인 레지스터 할당이 없다.(GCN 아키텍처의 경우.)
```
아마 다른 대부분의 아키텍처에서도 동일하게 레지스터 할당이 없지 않을까 싶다.
```
7. 스케줄링에서의 성능 향상을 위해, 텍스처를 읽을 때 부분적으로 Loop Unroll을 고려하라.( EX. Loop 횟수를 절반으로 줄이는 대신 한 Iteration에서 두 번의 텍스처 샘플링을 수행하라 )
```
for(int i = 0 ; i < 10 ; i++)
{
    float3 Color = g_MeshTexture[i].Sample(MeshTextureSampler, In.TextureUV);
    DoSomething(Color);
}
위의 코드 대신 아래의 코드를 선호해라는 말이다.
for(int i = 0 ; i < 10 ; i += 2)
{
    float3 Color1 = g_MeshTexture[i].Sample(MeshTextureSampler, In.TextureUV);
    DoSomething(Color1);

    float3 Color2 = g_MeshTexture[i + 1].Sample(MeshTextureSampler, In.TextureUV);
    DoSomething(Color2);
}
```
8. 위의 7번 사례의 정반대로 하는 것은 쉐이더에서의 VGPR 감소에 도움을 주고, Shader Occupancy 향상에 도움을 줄 수도 있다.            
```
결국 프로파일링을 통해 둘을 비교/검증해보라는 얘기인 것 같다.
VGPR, Shader Occupancy이란 : https://sungjjinkang.github.io/shader_occupancy
```
9. Loop를 완전히 Unrolling하는 것은 쉐이더 명령어 개수를 높일 수 있고, 명령어 캐시 히트율을 줄일 수 있다.            
```
언리얼 엔진 쉐이더 코드를 보면 UNROLL 코드를 여러 곳에서 볼 수 있다.
EX)
UNROLL
for (uint VertexIndex = 0; VertexIndex < 3; VertexIndex++)
{
    Output.Vertex = Input[VertexIndex];
    OutStream.Append(Output);
}
```
10. Linear(선형) 컴퓨트 쉐이더 스레드 인덱싱시 인덱스가 텍스처 좌표로 사용되는 경우 캐시 동기화 측면에서 성능적으로 손해를 볼 수 있다. [스레드 ID를 Swizzling](https://developer.nvidia.com/blog/optimizing-compute-shaders-for-l2-locality-using-thread-group-id-swizzling/)하는 것이 텍스처 샘플링시 공강적 지역성을 높여 캐시 히트율을 높일 수 있다.
```
GPU가 텍스처를 저장할 때 메모리 상에서 왼쪽 끝에서 오른쪽 끝으로 차례대로 저장한다고 생각할 수 있지만, 실제로는 그렇지 않다. GPU는 텍스처를 저장할 떄 Sampling시 캐시 히트율을 높이기 위해 어떤 특정 방식으로 텍스처를 저장한다.
https://learn.microsoft.com/en-us/windows/win32/api/d3d12/ne-d3d12-d3d12_texture_layout, morton curve, 힐베르트 커브 등등의 방식이 있다.         
그래서 컴퓨터 쉐이더 Thread Group내의 Thread들간의 캐시 히트율을 높이기 위해 "왼쪽 끝에서 오른쪽 끝으로 차례대로" 텍스처를 읽는 것이 아닌 위의 "GPU가 텍스처를 저장하는 방식"에 최적화된 순서로 텍스처를 읽는 것이 좋다는 의미이다.       
```
11. 특정 GPU에 적절한 버퍼 타입(Constant, Structured, ByteAddressed 등등)을 선택하라. 예를 들면 [인텔 GPU는 Byte Addressed Buffer를 사용하는 것이 성능적으로 유리](https://www.intel.com/content/www/us/en/developer/articles/guide/developer-and-optimization-guide-for-intel-processor-graphics-gen11-api.html)하다.
12. 프로그램에 적절한 Input, Output 데이터 타입을 선택해라. ( EX. 렌더타겟에는 RGBA16 대신 R11G11B10를 사용하라 ) 이것이 대역폭, VGPR 사용을 줄여준다.             
13. 큰 데이터 타입들을 더 작은 데이터 타입(낮은 정밀도, EX. FP16)로 [교체하거나, 패킹](https://github.com/apitrace/dxsdk/blob/master/Include/d3dx_dxgiformatconvert.inl)하는 것이 좋다. ALU decompression이 메모리 대역폭보다 저렴하다.
```
Packing을 하거나 더 낮은 정밀도를 사용하는 경우 Decompression을 위해 추가적인 ALU 연산이 필요하지만, 높은 대역폭보다는 이것이 더 낫다.
```
14. [16비트 부동 소수점(FP16)](https://therealmjp.github.io/posts/shader-fp16/) VGPR 사용을 줄이는데 좋고, ALU Op 코드의 처리 속도를 높여준다. [FP16 사용시 실제로 성능적 이득을 얻을 수 있는지 여부](https://twitter.com/MartinJIFuller/status/1485181350881763330?s=20)는 단일 스레드내에서 2배의 데이터를 병렬로 처리할 수 있는지에 따라, 오랜 기간 동안 FP16 명령어 세트 내의 머물렀는지에 따라 다를 것이다. 또한 암시적으로 FP16에서 FP32로 변환되지 않는지 주의해야한다.
```
16비트 부동 소수점 타입을 사용한다고 항상 성능이 향상되는 것은 아니다. FP16 타입 사용시 쉐이더 코드내에서 2배의 데이터를 병렬처리할 수 있는 코드가 존재하는지 그리고 FP16 명령어를 연속적으로 사용("FP16으로 처리하다 FP32로 처리 다시 FP16으로 처리, 다시 FP32로 처리" 이렇게 FP16과 FP32를 왔다 갔다하지 않고 일관적으로 연속되게 FP16 명령어를 사용해야한다는 것이다. FP16과 FP32를 왔다갔다 하는 경우 중간 중간에 암시적으로 FP16이 FP32으로 변환되어야 하는 등등으로 인해 FP16이 저장된 레지스터를 그대로 활용하지 못하게되어 성능적으로 오히려 손해를 볼 수 있다.)하는지에 따라 성능 향상이 없을 수도 있다.
그리고 필자가 알기로는 기본적으로 PC GPU에서는 일부 최신 GPU를 제외하고는 FP16을 지원하지 않는 것으로 알고 있다. ( 쉐이더 코드에서 FP16 타입을 사용하여도 쉐이더 컴파일 과정에서 FP16으로 교체됨 )  
참고 : https://sungjjinkang.github.io/half_precision
```
15. "[Context rolls](https://gpuopen.com/learn/understanding-gpu-context-rolls/)"이 중요할 수 있다(특히 작은 드로우 콜들의 경우). 그러니 렌더 State를 기준으로 드로우 콜들을 Batch해라.
```
드로우 콜들 사이에 렌더 스테이트를 자주 바꾸는 것은 좋지 않다.
그래서 언리얼 엔진4를 보면 MeshDrawCommandSortKey 생성시 최대한 렌더 스테이트가 변경되지 않도록(PSO기준, Masekd 여부에 따라) Sort를 한다.
(EarlyZ를 높이기 위해서는 카메라와의 거리에 따라 Sort를 해주는 것이 좋지만, EarlyZ 증가로 오는 성능 개선보다는 "Context rolls"를 통한 성능 개선이 더 크다고 판단되어 이렇게 코드가 작성되어 있는 것으로 추정된다. 모바일은 거리에 따라 Sort를 했던 것으로 기억함.)
(그리고 언리얼이 이렇게 Sort를 하는 또 다른 이유는 인접한 MeshDrawCommand들을 기준으로 Auto Instancing을 수행하기 때문이기도 하다.)
아래는 PC 플랫폼 기준(다른 플랫폼은 다를 수 있음)
FMeshDrawCommandSortKey CalculateBasePassMeshStaticSortKey(EDepthDrawingMode EarlyZPassMode, EBlendMode BlendMode, const FMeshMaterialShader* VertexShader, const FMeshMaterialShader* PixelShader)
{
	FMeshDrawCommandSortKey SortKey;
	SortKey.BasePass.VertexShaderHash = PointerHash(VertexShader) & 0xFFFF;
	SortKey.BasePass.PixelShaderHash = PointerHash(PixelShader);
	if (EarlyZPassMode != DDM_None)
	{
		SortKey.BasePass.Masked = BlendMode == EBlendMode::BLEND_Masked ? 0 : 1;
	}
	else
	{
		SortKey.BasePass.Masked = BlendMode == EBlendMode::BLEND_Masked ? 1 : 0;
	}
	return SortKey;
}
```
16. 더 빠른 명령어, 데이터 접근을 위해 가능하다면 [Scalar, Wave Operation](https://twitter.com/KostasAAA/status/1063075770761916416)들을 사용하라.
```
Warp 내의 스레드들간의 통신을 활용하라는 의미이다.
```
17. 비록 하나의 연산이 완전히 "scalarised"될 수는 없지만 "Waterfall Loop"를 통해 부분적으로 Scalarise할 수는 있다.
```
참고 : https://gpuopen.com/wp-content/uploads/slides/GPUOpen_Let%E2%80%99sBuild2020_A%20Trip%20Down%20the%20GPU%20Compiler%20Pipeline.pdf 70P
```
18. 메모리 대역폭은 쉐이더에서 가장 큰 병목점이다. 캐시 미스를 최소화하고(특히 노이즈로 샘플링할 때), 공간적 지역성을 높이기 위해 [de-interleaving](https://developer.nvidia.com/sites/default/files/akamai/gameworks/samples/DeinterleavedTexturing.pdf)를 고려해라. 이는 레이트레이싱이나 레이마칭과 같이 스레드 divergent 작업시 도움이 될 수 있다.
```
```
19. Arrays of 텍스처를 인덱싱할 때 [NonUniformIndex](https://www.asawicki.info/news_1608_direct3d_12_-_watch_out_for_non-uniform_resource_index)를 사용하라.(비록 인덱스를 쉐이더 코드에서 생성하더라도 말이다.)      
```
최적화가 아닌 쉐이더 작성시 주의해야할 점이다.
기본적으로 텍스처들의 Array를 인덱싱할 때 인덱스는 Uniform 변수여야한다(한 드로우 콜내에서는 항상 불변하여야 한다.).
이는 GPU가 SIMD 프로세서들로 구성되어 있고, 단일 Scalar 값이 아닌 여러 Scalar 값들을 SIMD 스레드들을 통해 동시에 처리(Warp내의 여러 스레드들이 Lock Step으로 동시에 연산을 처리)하기 때문에, Warp내의 스레드들이 반드시 동일한 리소스를 참조해야(이 케이스의 경우 동일한 인덱스를 사용해 동일한 리소스에 접근해야) 성능적으로 더 유리하다. 그리고 D3D는 기본적으로 이를 강제하기 때문에(동일한 인덱스를 사용해야한다) "텍스처들의 Array를 인덱싱할 때 인덱스는 반드시 Uniform 변수"여야 한다.
만약 Non-Uniform(Divergent)한 인덱스를 사용하여 Warp내의 스레드들이 동시에 서로 다른 리소스에 접근한다면 이는 UB(Undefined Behaviour)로 (몇몇 케이스 혹은 일부 GPU에서는) 잘못된 결과를 낼 수도 있다.     
이러한 문제에도 불구하고 HLSL 컴파일러는 "텍스처들의 Array를 인덱싱할 때 Non-Uniform 인덱스"를 사용하여도 어떠한 Warning도 주지 않는다.
그래서 무심코 "Non-Uniform 인덱스"를 사용하는 경우 추적하기 어려운 버그에 직면할 가능성이 있다.    
그러니 반드시 "Non-Uniform 인덱스"를 사용하는 경우에는 NonUniformResourceIndex를 함께 사용해주어야 한다.(쉐이더 코드에서 인덱스를 생성해서 사용하는 경우에도 인덱스가 Divergent(Warp내 스레드들간의 인덱스가 서로 다른 경우)할 수 있는 경우에는 반드시 NonUniformResourceIndex를 사용해주어야 한다.)
EX) textures[NonUniformResourceIndex(materialId)].Sample(samp, texCoords);
```
20. 상수인 자주 접근되는 데이터는 루트 시그니쳐로 직접 지정하라. 다만 레지스터는 제한되어 있기 때문에 너무 과도하게 하는 경우 메인 메모리로 evict될 수 있다.
```
참고 : https://learn.microsoft.com/en-us/windows/win32/direct3d12/using-constants-directly-in-the-root-signature
```
21. 데이터 타입을 기준으로 수학 연산들을 그룹핑하라. dimension으로 그룹핑하지 않고 scalar, vector 타입들을 섞어서 사용하는 경우 불필요한 ALU 명령어 사용으로 이어질 수 있다. [예시를 하나 보면](https://shader-playground.timjones.io/78d778ede516a60737da592727a877ed) 연속된 곱셈 연산들의 순서를 어떻게 재배열하느냐에 따라 명령어 개수를 25%까지 감소시킬 수 있다는 것을 알 수 있다.
22. 쉐이더에서의 분기를 너무 두려워하지는 말되, 분기를 사용하는 타당한 이유가 있어야한다. ( 분기가 어떻게 나뉘어지고, uniform data를 기준으로 분기가 나뉘는 것은 괜찮지만, uniform data가 아닌 경우에는 문제가 될 수 있다.) 쉐이더 코드에서 큼지막한 분기를 제거하는 것은 전체 VGPR을 증가시키고 Occupancy에 부정적인 영향을 줄 수 있다.           
```
비슷하게 UNROLL의 경우에도 Iteration에 속한 코드가 큰 경우에는 UNROLL로 인해 VGPR이 증가하고 쉐이더 Occupancy가 낮아지는 문제를 유발할 수 있다.
참고 :
쉐이더 ( GPU )에서 분기점( if )이 나쁜(느린) 이유 : https://sungjjinkang.github.io/shader_if
Shader Occupancy : https://sungjjinkang.github.io/shader_occupancy
```
23. 작은 분기는 "Flattend"을 하는 것이 낫을 수도 있다. Small branches of code may perform better when “flattened”.
```
Flatten이란? : https://sungjjinkang.github.io/shader_if
언리얼 엔진 쉐이더 코드에서도 "FLATTEN"을 종종 볼 수 있다.
```
24. "Tiled"처리를 하는 것을 고려하라. 만약 쉐이더에서 Divergence의 수가 많다면, 전체 스크린을 여러 타일로 나누어 각 타일을 분류하고, 비슷한 속성의 타일들을 묶어서 별도 쉐이더로 처리하라.
```
"Tile Classify"라고 흔히 알려진 기법이다.
Runbleverse라는 게임에서 언리얼 엔진을 수정하여 Tile Classification 최적화를 한 사례가 있다. : https://youtu.be/ZkhIFcjZpac?t=1336
```
25. VGPR 개수, 로컬 변수 저장 할당은 Warp Occupancy에 영향을 줄 수 있으니 항상 주의 깊게 지속적으로 살펴봐야 한다. 
26. [낮은 Occupancy가 항상 나쁜 것은 아니다](https://interplayoflight.wordpress.com/2020/11/11/what-is-shader-occupancy-and-why-do-we-care-about-it/), 쉐이더 컴파일러는 메모리 Latency를 숨기기 위한 여러 최적화를 수행한다. 대표적으로는 "텍스처 데이터 요청"과 "텍스처 샘플링(실제 사용)"을 최대한 떨어트려두어 실제 텍스처 샘플링시 텍스처 데이터가 곧바로 접근가능하게 만드는 것이 있다.
```
낮은 쉐이더 Occupancy가 항상 나쁜 것일까? :
위의 컴파일된 쉐이더 코드에서도 보이듯이 쉐이더 컴파일러는 컴파일시 명령어의 순서를 재배치하여 “메모리에 데이터를 요청하는 명령어”와 “실제 데이터를 사용하는 명령어”를 최대한 떨어트려두어 최대한 실제 데이터를 사용할 때 데이터가 준비되어 있게끔 한다. 그래서 쉐이더 Occupancy가 낮다고 반드시 메모리 Stall이 발생하는 것은 아니다.(쉐이더 컴파일러의 노력 덕분에 데이터가 실제로 사용될 때 이미 해당 데이터가 준비되어 있을 가능성이 적지 않다는 것이다.) 그래서 쉐이더 Occupancy가 아니라 메모리 읽기(ex. 텍스쳐 읽기)로 인해 Stall이 실제 발생하였는지(SM에 다른 Warp를 배정하였음에도, 혹은 다른 Warp도 데이터가 준비되지 않아 GPU 코어가 Stall되었는지)를 보여주는 지표를 기준으로 최적화를 하여야 한다. 실제로 GPU 코어가 Stall되었다는 지표를 보인다면 쉐이더 Occupancy를 높이는 최적화(쉐이더 코드를 수정하여 VGPR을 줄여보는)를 수행한다면 성능 향상을 얻을 수 있을 것이다.
-> 실제로 GPU 코어가 Stall되었는지(GPU 코어가 놀고 있는지)를 보아야 한다
참고 : https://sungjjinkang.github.io/shader_occupancy
```
27. 높은 Occupancy/낮은 VGPR 개수가 항상 좋은 것은 아니다. 쉐이더 컴파일러는 VGPR 재사용율을 높이기 위해 메모리 Fetch 명령어를 연속해서 배치하는데, 이는 나쁜 스케줄링, 캐시 Trash(Warp의 교체가 잦아 캐싱해둔 데이터를 충분히 활용하지 못하는 것)를 이끌 수 있다.
```
 쉐이더가 많은 수의 메모리 읽기 명령어를 수행하는 경우 쉐이더 Occupancy가 높은 것(쉐이더의 VGPR이 낮은 경우)이 성능에 부정적일 수 있는데.
쉐이더가 많은 수의 메모리 읽기 명령어를 수행하는 경우 쉐이더 컴파일러는 최적화 차원에서 사용되는 VGPR(벡터 레지스터)의 수을 낮추기 위해(쉐이더 컴파일러는 VGPR을 낮추기 위한 코드 최적화를 수행한다.) 복수의 메모리 요청 명령어 연속적으로 배치하기도 하는데(ex. 메모리 읽기 명령어를 수행하고, 데이터를 기다리고, 도착한 데이터를 레지스터에 저장하고, 그것을 사용하고, 그리고 바로 다음 메모리 요청 동작으로 도착한 데이터를 다시 해당 레지스터에 저장해 레지스터를 재사용한다. -> 하나의 레지스터로 복수의 “메모리 읽기”, “읽은 후 그 데이터로 연산을 하는 동작”을 처리함.).. 쉐이더 컴파일러가 레지스터를 재사용하여 VGPR을 낮추기 위해 메모리를 읽는 명령어을 연속되게 배치하지만 그 수가 많은 경우 메모리 Stall이 발생할 수 있다.
참고 : https://sungjjinkang.github.io/shader_occupancy
```
28. [렌더타겟/스크린 밖의 영역에서는 연산을 수행하지 마라](https://interplayoflight.wordpress.com/2021/10/28/the-curious-case-of-slow-raytracing-on-a-high-end-gpu/)
29. 잠재적인 Export(픽셀 쉐이더 결과값을 렌더 타겟으로 쓰는 동작에서) Stall들을 제거하기 위해 픽셀 쉐이더에서 수행하는 작업을 컴퓨트 쉐이더로 옮겨라. ( 특히 픽셀 쉐이더가 짧은 경우에는 말이다. ) 비슷하게 이러한 최적화 작업은 Early out이 많거나, 분기 처리가 많은 쉐이더 코드에서는 성능 향상을 가져올 수 있다.
```
Export Stall은 처음 들어보는 용어라 ChatGPT에 물어봤다. 적당히 걸러서 보기 바란다.
In the context of pixel shaders, an "export stall" refers to a situation where the shader is unable to efficiently output the results of its computations to the next stage of the graphics pipeline. This can occur due to various reasons, such as dependencies between shader instructions or limitations in the hardware architecture.
Export stalls can happen when the pixel shader attempts to write data to a buffer or memory location but encounters a delay in the process. This delay can be caused by several factors:
Dependencies: If a pixel shader instruction depends on the results of a previous instruction that has not yet completed, it must wait for the previous instruction to finish before it can proceed. This dependency can lead to a stall in the export of the computed values.
Memory Access: If the pixel shader needs to read or write to memory, such as a texture lookup or a render target, it can introduce stalls. Memory accesses often have higher latency compared to other shader instructions, which can cause delays in exporting the results.
Bandwidth Limitations: The shader may be limited by the available memory bandwidth, especially when dealing with large amounts of data. When the shader is producing results faster than they can be transferred or written to memory, it can result in export stalls.
Resource Conflicts: If multiple shaders or threads are trying to access the same memory location simultaneously, conflicts can occur, leading to stalls. These conflicts need to be resolved before the shader can proceed with exporting the results.
To mitigate export stalls, shader compilers and hardware architectures employ various optimization techniques, such as instruction reordering, resource scheduling, and memory access optimizations. These techniques aim to minimize dependencies, maximize memory bandwidth utilization, and improve the overall efficiency of the shader execution.
```
30. [Early Z가 비활성화되는 조건을 알고 있어라.](https://developer.arm.com/documentation/102224/0200/Early-Z) ( 픽셀 쉐이더에서 Depth 혹은 UAV에 값 쓰기, Alpha Testing, Alpha To Coverage 등등...) 그것은 불필요한 픽셀 연산을 유발한다.
31. "Stencil Culling(스텐실 버퍼를 활용해서 Pixel Shading 연산을 생략하는 것)"은 일반적으로 "픽셀 쉐이더에서 픽셀을 버리는 것"보다 빠르다.
```
Early Stencil Culling : https://stackoverflow.com/questions/18740841/early-stencil-culling
```
32. [Bit Twiddling Hacks](http://graphics.stanford.edu/~seander/bithacks.html)을 사용하라. 그러한 Hack들은 쉐이더에서 매우 유용하다.
33. 컴퓨터 쉐이더 그룹은 최소한 하나의 Wavefronts/Warps를 채워야한다. 예를 들어 GCN 아키텍처의 경우 최소한 128/256 개수의 스레드만큼은 Issue를 해야한다. 그 숫자(최소한 Issue해줘야 해주는 것이 좋은 스레드 개수)는 높은 Occupancy를 위한 목적으로 스레드당 사용되는 레지스터의 수에 따라 다르기도하고, 스레드 그룹 내의 스레드들간의 데이터를 공유할 필요가 있는지 여부에 따라서도 다르다. ( [추가적으로 영향을 주는 것들](https://gpuopen.com/learn/optimizing-gpu-occupancy-resource-usage-large-thread-groups/) )
34. "atan"과 같은 non-native 명령어 사용을 피해라. 이러한 명령어들은 결국 내부적으로는 많은 수의 native 명령어 사용으로 동작한다.
35. Integer 나눗셈을 피해라. 그것 또한 non-native 연산이다. D3D12 디버그 레이어를 통해 확인해보면, 이러한 "Interger 나눗셈"에 대해 Warning을 주는 것을 알 수 있을 것이다.
36. ["trigonometric"](https://seblagarde.wordpress.com/2014/12/01/inverse-trigonometric-functions-gpu-optimization-for-amd-gcn-architecture/)와 같은 값 비싼 함수에 대한 [근사값을 제공하는 함수](https://github.com/michaldrobot/ShaderFastLibs)를 활용하라. 다만 그것이 진짜 성능적으로 유리한지는 프로파일링 해보아라.
37. ["InverseLerp"](https://gamedev.net/articles/programming/general-and-gameplay-programming/inverse-lerp-a-super-useful-yet-often-overlooked-function-r5230)는 거리, 범위를 기반으로 분수(fraction)을 얻는데 몇 없는 유용한 함수이다. 나는 자주 "smoothstep"의 값 싼 대체재로 이것을 사용한다.
38. 인덱스나 Loop 횟수를 float 대신 int로 쉐이더에 전달하는 것이 불필요한 변환 명령어, 레지스터 사용을 줄여준다는 측면에서 성능 향상에 도움이 된다. 쉐이더에서 뿐만아니라 양쪽 끝단에서 그것을 해라.
```
"Make sure that you do it on both ends and not only in the shader"에서 "on both ends"가 무엇을 의미하는지 모르겠다...
```
39. 픽셀 쉐이더에서 "SV_Position"를 컴퓨트 쉐이더에서 모방하기 위해 "SV_DispatchThreadID"를 사용할 때, 픽셀 사이즈의 절반만큼을 더해줘라. 그렇지 않으면 에러(오차)를 얻을 것이다. ( EX. TAA를 할 때 World Position reconstruction(재생성) mismatch(불일치) ) 픽셀 쉐이더에서 "SV_Position"은 이미 픽셀 사이즈의 절반만큼이 더해져서 픽셀 쉐이더에 전달된다.
40. 상수 버퍼에서 [Packoffsets](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-variable-packoffset)는 [연속적이거나 순서대로, 혹은 심지어 0부터 시작할 필요가 없다.](https://shader-playground.timjones.io/df2b371902ad92cb2f3aedc50cf772a1). 만약 너의 쉐이더 리플랙션 데이터가 이것(연속적이지 않거나, 순서대로 쓰지 않았거나, 0부터 시작하지 않은 경우)을 인지하지 못한다면, 디버깅할 때 고생을 좀 할 것이다.
41. LDS 읽기가 메모리 읽기 명령어들의 Latency를 높일 수 있는 경우 그것들을 연속되게 배치하기 때문에 ["Bank conflicts"](https://on-demand.gputechconf.com/gtc/2018/presentation/s81006-volta-architecture-and-performance-optimization.pdf)가 발생한다.
42. Vertex Position 중 하나를 NaN으로 셋팅하는 것은 버텍스 쉐이더에서 삼각형을 컬하는 좋은 방법이다. GPU에 따라  ["0/0 쓰기" 혹은 "asfloat(0X7fc00000)"](https://twitter.com/longbool/status/1484984001634971655?s=20)를 통해서도 버텍스 쉐이더에서 삼각형을 컬할 수 있다.
43. HLSL에서 "asfloat(0X7F800000)"는 "무한대"를 표현하는데 사용될 수 있다. 예를 들면 [Far plane을 셋팅하는 경우](https://link.springer.com/chapter/10.1007/978-1-4842-7185-8_3)에 말이다.
44. "GetDimensions"는 텍스처 명령어로 분류된다. 그냥 상수 버퍼로 텍스처 Dimension을 직접 전달하는 것이 성능상 유리하다.
45. 픽셀 쉐이더 작업을 버텍스 쉐이더로 옮기는 것을 고려하라. 다만 그것이 너무 많다면 버텍스 쉐이더에서 픽셀 쉐이더로 결과를 전달하는 곳에서 병목이 발생할 수 있고, 픽셀이 컬되거나 버려지는 경우에도 그 연산은 버텍스 쉐이더에서 불필요하게 항상 연산이 되는 문제가 있고, 몇몇 아키텍처의 경우 감소된 캐시 지역성으로 인해 버텍스 쉐이더에서의 텍스처 접근이 더 느릴 수 있는 문제도 있다.
```
결국 프로파일링해봐야 뭐가 더 나은지 알 수 있다.
```
46. NaN들은 여러 파이프라인에 걸쳐 전달이 될 수 있고 결국에는 최종 Output을 파괴한다. 디버깅을 편하게 하기 위해서는 [쉐이더에서 Nan을 잡고](https://sakibsaikia.github.io/graphics/2022/01/04/Nan-Checks-In-HLSL.html), 그것들을 시각화하는 것이 좋다.
47. 쉐이더를 디버깅할 때, 문제의 차원을 줄이려고 노력해라. 그러니깐 예를들면.. 텍스처 대신 Input으로 고정된 값을 전달해서 디버깅을 해라.
```
최대한 영향을 주는 조건들을 단순화시켜라는 말인 것 같다.
```
48. 쉐이더 디버깅을 시각화하는 것(EX. 각 경로별로 쉐이더 Output을 빨강색과 같은 특정 상수 값으로 만들어 분류.)는 때때로 뭐가 잘못되었는지 판단하는 가장 빠른 방법이다.
49. 버텍스 Attribute 보간(Interpolation)은 현대 GPU에서는 쉐이더에서 이루어진다. 그러니 보간이 필요 없는 경우에는 ["nointerpolation" modifier](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-struct)을 명시해주는 것이 ALU, VGPR 사용 감소 측면에서 비용 감소에 도움이 된다.
50. 몇몇 아키텍처에서는 [Depth 버퍼를 텍스처로 바인딩하는 것이 Depth 버퍼의 Decompress](http://amd-dev.wpengine.netdna-cdn.com/wordpress/media/2013/05/GCNPerformanceTweets.pdf)를 유발하기 때문에 이후 z 테스트 비용을 더 높일 수 있다. 그러니 Depth 버퍼를 텍스처로 바인딩하기 전에 지오메트리 렌더링을 위해 그것을 사용하는 것을 완전히 끝내었는지 확인하세요. (디퍼드 쉐이딩 엔진은 불투명 메시를 렌더링 하기 전 라이팅을 위해 Depth 버퍼를 바인딩하는 것이 필요하기 때문에 이를 제거하는 것이 아마 쉽지는 않을 것이다.)
```
God of War Ragnarok의 렌더링을 소개하는 GDC 강연(https://www.gdcvault.com/play/1028846/Rendering-God-of-War-Ragnarok 163P)에서도 이와(Depth 버퍼를 텍스처로 바인딩하는 것이 Depth 버퍼의 Decompress) 관련된 얘기가 나온다.
```
51. Z 테스트에 대해 [Reverse-Z](https://developer.nvidia.com/content/depth-precision-visualized)는 [적용하기 쉽고](https://github.com/sebbbi/rust_test/commit/d64119ce22a6a4972e97b8566e3bbd221123fcbb), Depth 버퍼에서 Depth 정밀도를 높이고, Z-Fighting을 상당히 줄여준다.
```
Reverse Depth Buffer : https://sungjjinkang.github.io/reverse_z
```
52. [쉐이더 어셈블리](https://interplayoflight.wordpress.com/2021/04/18/how-to-read-shader-assembly/)를 읽는 것을 배워두고, [Shader Playground](http://shader-playground.timjones.io/)와 같은 툴을 사용하여 몇몇 명령어들의 잠재적인 비용을 확인하여 쉐이더 컴파일러의 결과물을 살펴보는 습관을 가지는 것이 도움이 된다.
53. 성능 향상을 위해 GPU를 완전히 갈구는 것이 필요하다(ALU/Fixed Function Unit을 놀지 못하게 갈구는 것). 이를 위해 "어디가 병목인지를 알기 위해 각 케이스들을 프로파일링 하고", "쉐이더를 덜 효율적으로 만드는 것이 다른 작업들과 더 잘 overlap되게 만든다면 쉐이더를 덜 효율적으로 만드는 어찌보면 Best Practice와 반대로하는 것"이 도움이 될 수 있다. [이 발표자료](https://s3.amazonaws.com/nd.images/research/2020_siggraph/Low_Level_Optimizations_In_TLOU2.pptx)는 이를 잘 보여준다.
