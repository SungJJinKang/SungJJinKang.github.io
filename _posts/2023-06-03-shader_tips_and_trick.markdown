---
layout: post
title:  "SHADER TIPS AND TRICKS(번역) (작성 중)"
date:   2023-06-03
tags: [ComputerGraphics]
---          
             
이 글은 [SHADER TIPS AND TRICKS](https://interplayoflight.wordpress.com/2022/01/22/shader-tips-and-tricks/)를 번역한 글입니다.        
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
7. 스케줄링에서의 성능 향상을 위해, 텍스처를 읽을 때 부분적으로 Loop Unroll을 고려하라. ( EX. Loop 횟수를 절반으로 줄이는 대신 한 Iteration에서 두 번의 텍스처 샘플링을 수행하라)
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
10.  Linear(선형) 컴퓨트 쉐이더 스레드 인덱싱시 인덱스가 텍스처 좌표로 사용되는 경우 캐시 동기화 측면에서 성능적으로 손해를 볼 수 있다. [스레드 ID를 Swizzling](https://developer.nvidia.com/blog/optimizing-compute-shaders-for-l2-locality-using-thread-group-id-swizzling/)하는 것이 텍스처 샘플링시 공강적 지역성을 높여 캐시 히트율을 높일 수 있다.
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
14.  [16비트 부동 소수점(FP16)](https://therealmjp.github.io/posts/shader-fp16/) VGPR 사용을 줄이는데 좋고, ALU Op 코드의 처리 속도를 높여준다. [FP16 사용시 실제로 성능적 이득을 얻을 수 있는지 여부](https://twitter.com/MartinJIFuller/status/1485181350881763330?s=20)는 단일 스레드내에서 2배의 데이터를 병렬로 처리할 수 있는지에 따라, 오랜 기간 동안 FP16 명령어 세트 내의 머물렀는지에 따라 다를 것이다. 또한 암시적으로 FP16에서 FP32로 변환되지 않는지 주의해야한다.
```
16비트 부동 소수점 타입을 사용한다고 항상 성능이 향상되는 것은 아니다. FP16 타입 사용시 쉐이더 코드내에서 2배의 데이터를 병렬처리할 수 있는 코드가 존재하는지 그리고 FP16 명령어를 연속적으로 사용("FP16으로 처리하다 FP32로 처리 다시 FP16으로 처리, 다시 FP32로 처리" 이렇게 FP16과 FP32를 왔다 갔다하지 않고 일관적으로 연속되게 FP16 명령어를 사용해야한다는 것이다. FP16과 FP32를 왔다갔다 하는 경우 중간 중간에 암시적으로 FP16이 FP32으로 변환되어야 하는 등등으로 인해 FP16이 저장된 레지스터를 그대로 활용하지 못하게되어 성능적으로 오히려 손해를 볼 수 있다.)하는지에 따라 성능 향상이 없을 수도 있다.

그리고 필자가 알기로는 기본적으로 PC GPU에서는 일부 최신 GPU를 제외하고는 FP16을 지원하지 않는 것으로 알고 있다. ( 쉐이더 코드에서 FP16 타입을 사용하여도 쉐이더 컴파일 과정에서 FP16으로 교체됨 )

references : https://sungjjinkang.github.io/half_precision
```
15.  "[Context rolls](https://gpuopen.com/learn/understanding-gpu-context-rolls/)"이 중요할 수 있다(특히 작은 드로우 콜들의 경우). 그러니 렌더 State를 기준으로 드로우 콜들을 Batch해라.       
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
