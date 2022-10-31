---
layout: post
title:  "언리얼 엔진4 FMobileSceneRenderer 분석 - 3 ( FSceneRenderer::RenderCustomDepthPass )"
date:   2022-04-16
categories: UE UnrealEngine ComputerScience ComputerGraphics
---

중요도가 떨어지는 코드는        
```

...
...
...

```
으로 처리해두었으니 확인하고 싶은 경우에는 UE4 소스코드를 확인해주세요.           

------------------------         

**FMobileSceneRenderer**는 언리얼 엔진4의 모바일용 그래픽스 파이프라인이다.                
이 글은 **FMobileSceneRenderer::Render** 함수를 위주로 FMobileSceneRenderer의 소스 코드를 분석할 것이다.      
          
UE4 4.27 버전을 기준으로 작성하였다.      
4.27 버전이 언리얼 엔진 4세대의 마지막 Release 버전이다.             
          
----------------------------              
             
이번 챕터에는 **FSceneRenderer::RenderCustomDepthPass**에 대해 분석해볼 것이다.       
        
함께 보면 좋은 자료 : [Why Talking About Render Graphs](https://logins.github.io/graphics/2021/05/31/RenderGraphs.html)
                       
------------------------------         

```cpp
void FMobileSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
{

	// ...
	// ...
	// ...

	if (bShouldRenderCustomDepth)
	{
		FRDGBuilder GraphBuilder(RHICmdList);
		FSceneTextureShaderParameters SceneTextures = CreateSceneTextureShaderParameters(GraphBuilder, Views[0].GetFeatureLevel(), ESceneTextureSetupMode::None);
		RenderCustomDepthPass(GraphBuilder, SceneTextures);
		GraphBuilder.Execute();
	}

	// ...
	// ...
	// ...

}
```


```cpp
void FSceneRenderer::RenderCustomDepthPass(FRDGBuilder& GraphBuilder, const FSceneTextureShaderParameters& SceneTextures)
{
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderCustomDepthPass);

	// do we have primitives in this pass?
	bool bPrimitives = false;

	if (!Scene->World || (Scene->World->WorldType != EWorldType::EditorPreview && Scene->World->WorldType != EWorldType::Inactive))
	{
		for(int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
		{
			const FViewInfo& View = Views[ViewIndex];
			if (View.bHasCustomDepthPrimitives)
			{
				bPrimitives = true;
				break;
			}
		}
	}

	
	const bool bMobilePath = (FeatureLevel <= ERHIFeatureLevel::ES3_1);

	// ⭐
	// Stencil 버퍼에 쓰기 동작을 수행할지 여부
	const bool bWritesCustomStencilValues = FSceneRenderTargets::IsCustomDepthPassWritingStencil(FeatureLevel);
	// ⭐

	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(GraphBuilder.RHICmdList);

	// ⭐
	// 아래로 내려가 분석한 것을 보세요.
	const FCustomDepthTextures CustomDepthTextures = SceneContext.RequestCustomDepth(GraphBuilder, bPrimitives);
	// ⭐

	if (CustomDepthTextures.CustomDepth)
	{
		RDG_GPU_STAT_SCOPE(GraphBuilder, CustomDepth);

		// Only clear once the target for every views it contains.
		ERenderTargetLoadAction DepthLoadAction = ERenderTargetLoadAction::EClear;
		ERenderTargetLoadAction StencilLoadAction = bWritesCustomStencilValues ? ERenderTargetLoadAction::EClear : ERenderTargetLoadAction::ENoAction;

		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);

			FViewInfo& View = Views[ViewIndex];

			if (View.ShouldRenderView())
			{
				FCustomDepthPassParameters* PassParameters = GraphBuilder.AllocParameters<FCustomDepthPassParameters>();
				PassParameters->SceneTextures = SceneTextures;

				// ⭐
				// Render Graph에 전달할 Parameter를 셋팅한다.
				if (bMobilePath)
				{
					checkSlow(CustomDepthTextures.MobileCustomDepth && CustomDepthTextures.MobileCustomStencil);

					// ⭐
					// Render Target에 Depth Buffer Texture를 셋팅한다.
					PassParameters->RenderTargets[0] = FRenderTargetBinding(CustomDepthTextures.MobileCustomDepth, DepthLoadAction);
					// ⭐

					// ⭐
					// Render Target에 Stencil Buffer Texture를 셋팅한다.
					PassParameters->RenderTargets[1] = FRenderTargetBinding(CustomDepthTextures.MobileCustomStencil, StencilLoadAction);
					// ⭐

					// ⭐
					// FDepthStencilBinding : Depth, Stencil 렌더 타겟을 어떻게 바인드 할지에 대한 Render Graph 정보를 담은 구조체
					PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
						CustomDepthTextures.CustomDepth,
						DepthLoadAction,
						StencilLoadAction,
						FExclusiveDepthStencil::DepthWrite_StencilWrite);
					// ⭐
				}
				else
				{
					PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
						CustomDepthTextures.CustomDepth,
						DepthLoadAction,
						StencilLoadAction,
						FExclusiveDepthStencil::DepthWrite_StencilWrite);
				}
				// ⭐

				// Next pass, do not clear the shared target, only render into it
				DepthLoadAction = ERenderTargetLoadAction::ELoad;
				StencilLoadAction = ERenderTargetLoadAction::ELoad;

				// ⭐⭐⭐⭐⭐⭐⭐
				//
				// Render Graph에 "Custom Depth" Pass를 추가한다.
				//
				// 참고 자료 : https://logins.github.io/graphics/2021/05/31/RenderGraphs.html
				GraphBuilder.AddPass(
					RDG_EVENT_NAME("CustomDepth"),
					PassParameters,
					ERDGPassFlags::Raster,
					[this, &View](FRHICommandListImmediate& RHICmdList)
				{
					FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);
					{
						const bool bMobilePath = FSceneInterface::GetShadingPath(View.GetFeatureLevel()) == EShadingPath::Mobile;

						// ⭐
						// 모바일의 경우 CustomDepthDownSample 옵션이 Enabled 되어 있는 경우 Depth, Stencil Buffer를 다운 샘플링 ( 축소해서 Write한 후 늘린다 )해서 Write한다.
						static const auto MobileCustomDepthDownSampleLocalCVar = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.Mobile.CustomDepthDownSample"));
						const bool bMobileCustomDepthDownSample = bMobilePath && MobileCustomDepthDownSampleLocalCVar && MobileCustomDepthDownSampleLocalCVar->GetValueOnRenderThread() > 0;
						SetStereoViewport(RHICmdList, View, bMobileCustomDepthDownSample ? 0.5f : 1.0f);
						// ⭐
					}

					if (CVarCustomDepthTemporalAAJitter.GetValueOnRenderThread() == 0 && View.AntiAliasingMethod == AAM_TemporalAA)
					{
						
						// ...
						// ... CVarCustomDepthTemporalAAJitter의 기본 값은 1이다.
						// ... -> 생략
						// ...
						
					}
					else
					{
						// ⭐
						// Uniform Buffer를 업데이트한다.
						Scene->UniformBuffers.CustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(*View.CachedViewUniformShaderParameters);
						// ⭐

						if ((View.IsInstancedStereoPass() || View.bIsMobileMultiViewEnabled) && View.Family->Views.Num() > 0)
						{
							// ...
							// ...
							// ...
						}
						else
						{
							// If we don't render this pass in stereo we simply update the buffer with the same view uniform parameters.
							Scene->UniformBuffers.InstancedCustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(reinterpret_cast<FInstancedViewUniformShaderParameters&>(*View.CachedViewUniformShaderParameters));
						}
					}

					extern TSet<IPersistentViewUniformBufferExtension*> PersistentViewUniformBufferExtensions;

					for (IPersistentViewUniformBufferExtension* Extension : PersistentViewUniformBufferExtensions)
					{
						Extension->BeginRenderView(&View);
					}

					// ⭐
					//
					// /**
					// * Parallel mesh draw command processing and rendering. 
					// * Encapsulates two parallel tasks - mesh command setup task and drawing task.
					// *
					// * Mesh Draw Command를 병렬로 처리하고, 렌더링을 수행한다.
					// * Mesh Draw Command를 셋팅하고, Draw하는 두 동작을 수행한다.
					// */
					// class FParallelMeshDrawCommandPass
					//
					// 가시성 테스트를 통과한 Mesh Draw Command에 대한 Draw 작업을 전송한다.
					// FParallelMeshDrawCommandPass::DispatchDraw(FParallelCommandListSet* ParallelCommandListSet, FRHICommandList& RHICmdList) const;
					//
					View.ParallelMeshDrawCommandPasses[EMeshPass::CustomDepth].DispatchDraw(nullptr, RHICmdList);
					// ⭐

				});
				// ⭐⭐⭐⭐⭐⭐⭐

				SceneContext.bCustomDepthIsValid = true;
			}
		}
	}
}
```


```cpp
FCustomDepthTextures FSceneRenderTargets::RequestCustomDepth(FRDGBuilder& GraphBuilder, bool bPrimitives)
{
	FCustomDepthTextures CustomDepthTextures{};

	// ⭐
	const int CustomDepthValue = CVarCustomDepth.GetValueOnRenderThread();
	const bool bWritesCustomStencilValues = IsCustomDepthPassWritingStencil(CurrentFeatureLevel);

	const bool bMobilePath = (CurrentFeatureLevel <= ERHIFeatureLevel::ES3_1);
	const int32 DownsampleFactor = bMobilePath && CVarMobileCustomDepthDownSample.GetValueOnRenderThread() > 0 ? 2 : 1;
	// ⭐

	if ((CustomDepthValue == 1 && bPrimitives) || CustomDepthValue == 2 || bWritesCustomStencilValues)
	{
		// ⭐
		// 생명 주기가 Render Graph로부터 독립되어 있는 External Texture
		// 대표적으로는 Window API의 Swap Chain 백버퍼가 있다.
		CustomDepthTextures.CustomDepth = TryRegisterExternalTexture(GraphBuilder, CustomDepth);
		if (bMobilePath)
		{
			CustomDepthTextures.MobileCustomDepth = TryRegisterExternalTexture(GraphBuilder, MobileCustomDepth);
			CustomDepthTextures.MobileCustomStencil = TryRegisterExternalTexture(GraphBuilder, MobileCustomStencil);
		}
		// ⭐

		const FIntPoint CustomDepthBufferSize = FIntPoint::DivideAndRoundUp(BufferSize, DownsampleFactor);

		bool bHasValidCustomDepth = (CustomDepthTextures.CustomDepth && CustomDepthBufferSize == CustomDepthTextures.CustomDepth->Desc.Extent && !GFastVRamConfig.bDirty);
		bool bHasValidCustomStencil;
		if (bMobilePath)
		{
			bHasValidCustomStencil = (CustomDepthTextures.MobileCustomStencil && CustomDepthBufferSize == CustomDepthTextures.MobileCustomStencil->Desc.Extent) &&
			                         // Use memory less when stencil writing is disabled and vice versa
			                         bWritesCustomStencilValues == ((CustomDepthTextures.MobileCustomStencil->Desc.Flags & TexCreate_Memoryless) == 0);
		}
		else
		{
			bHasValidCustomStencil = CustomStencilSRV.IsValid();
		}

		if (!(bHasValidCustomDepth && bHasValidCustomStencil))
		{
			// Skip depth decompression, custom depth doesn't benefit from it
			// Also disables fast clears, but typically only a small portion of custom depth is written to anyway
			ETextureCreateFlags CustomDepthFlags = GFastVRamConfig.CustomDepth | TexCreate_NoFastClear | TexCreate_DepthStencilTargetable | TexCreate_ShaderResource;
			if (bMobilePath)
			{
				CustomDepthFlags |= TexCreate_Memoryless;
			}

			// Todo: Could check if writes stencil here and create min viable target
			const FRDGTextureDesc CustomDepthDesc = FRDGTextureDesc::Create2D(CustomDepthBufferSize, PF_DepthStencil, FClearValueBinding::DepthFar, CustomDepthFlags);

			CustomDepthTextures.CustomDepth = GraphBuilder.CreateTexture(CustomDepthDesc, TEXT("CustomDepth"));
			ConvertToExternalTexture(GraphBuilder, CustomDepthTextures.CustomDepth, CustomDepth);
			
			if (bMobilePath)
			{
				const float DepthFar = (float)ERHIZBuffer::FarPlane;
				const FClearValueBinding DepthFarColor = FClearValueBinding(FLinearColor(DepthFar, DepthFar, DepthFar, DepthFar));

				ETextureCreateFlags MobileCustomDepthFlags = TexCreate_RenderTargetable | TexCreate_ShaderResource;
				ETextureCreateFlags MobileCustomStencilFlags = MobileCustomDepthFlags;
				if (!bWritesCustomStencilValues)
				{
					MobileCustomStencilFlags |= TexCreate_Memoryless;
				}

				const FRDGTextureDesc MobileCustomDepthDesc = FRDGTextureDesc::Create2D(CustomDepthBufferSize, PF_R16F, DepthFarColor, MobileCustomDepthFlags);
				const FRDGTextureDesc MobileCustomStencilDesc = FRDGTextureDesc::Create2D(CustomDepthBufferSize, PF_G8, FClearValueBinding::Transparent, MobileCustomStencilFlags);

				CustomDepthTextures.MobileCustomDepth = GraphBuilder.CreateTexture(MobileCustomDepthDesc, TEXT("MobileCustomDepth"));
				CustomDepthTextures.MobileCustomStencil = GraphBuilder.CreateTexture(MobileCustomStencilDesc, TEXT("MobileCustomStencil"));

				ConvertToExternalTexture(GraphBuilder, CustomDepthTextures.MobileCustomDepth, MobileCustomDepth);
				ConvertToExternalTexture(GraphBuilder, CustomDepthTextures.MobileCustomStencil, MobileCustomStencil);

				CustomStencilSRV.SafeRelease();
			}
			else
			{
				CustomStencilSRV = RHICreateShaderResourceView((FTexture2DRHIRef&)CustomDepth->GetRenderTargetItem().TargetableTexture, 0, 1, PF_X24_G8);
			}
		}
	}

	return CustomDepthTextures;
}
```