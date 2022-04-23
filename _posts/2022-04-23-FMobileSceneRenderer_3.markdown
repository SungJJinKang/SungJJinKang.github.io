---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 4 ( FSceneRenderer::RenderCustomDepthPass )"
date:   2022-04-16
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
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
        
함께 보면 좋은 자료 : [Why Talking About UE4 Shaders](https://logins.github.io/graphics/2021/03/31/UE4ShadersIntroduction.html#rdg-dynamics), [Why Talking About Render Graphs](https://logins.github.io/graphics/2021/05/31/RenderGraphs.html)
                       
------------------------------         

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
	const bool bWritesCustomStencilValues = FSceneRenderTargets::IsCustomDepthPassWritingStencil(FeatureLevel);

	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(GraphBuilder.RHICmdList);
	const FCustomDepthTextures CustomDepthTextures = SceneContext.RequestCustomDepth(GraphBuilder, bPrimitives);

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

				if (bMobilePath)
				{
					checkSlow(CustomDepthTextures.MobileCustomDepth && CustomDepthTextures.MobileCustomStencil);

					PassParameters->RenderTargets[0] = FRenderTargetBinding(CustomDepthTextures.MobileCustomDepth, DepthLoadAction);
					PassParameters->RenderTargets[1] = FRenderTargetBinding(CustomDepthTextures.MobileCustomStencil, StencilLoadAction);

					PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
						CustomDepthTextures.CustomDepth,
						DepthLoadAction,
						StencilLoadAction,
						FExclusiveDepthStencil::DepthWrite_StencilWrite);
				}
				else
				{
					PassParameters->RenderTargets.DepthStencil = FDepthStencilBinding(
						CustomDepthTextures.CustomDepth,
						DepthLoadAction,
						StencilLoadAction,
						FExclusiveDepthStencil::DepthWrite_StencilWrite);
				}

				// Next pass, do not clear the shared target, only render into it
				DepthLoadAction = ERenderTargetLoadAction::ELoad;
				StencilLoadAction = ERenderTargetLoadAction::ELoad;

				GraphBuilder.AddPass(
					RDG_EVENT_NAME("CustomDepth"),
					PassParameters,
					ERDGPassFlags::Raster,
					[this, &View](FRHICommandListImmediate& RHICmdList)
				{
					FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);
					{
						const bool bMobilePath = FSceneInterface::GetShadingPath(View.GetFeatureLevel()) == EShadingPath::Mobile;

						static const auto MobileCustomDepthDownSampleLocalCVar = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("r.Mobile.CustomDepthDownSample"));
						const bool bMobileCustomDepthDownSample = bMobilePath && MobileCustomDepthDownSampleLocalCVar && MobileCustomDepthDownSampleLocalCVar->GetValueOnRenderThread() > 0;
						SetStereoViewport(RHICmdList, View, bMobileCustomDepthDownSample ? 0.5f : 1.0f);
					}

					if (CVarCustomDepthTemporalAAJitter.GetValueOnRenderThread() == 0 && View.AntiAliasingMethod == AAM_TemporalAA)
					{
						// Handle the "current" view the same, always.
						FViewUniformShaderParameters CustomDepthViewUniformBufferParameters = *View.CachedViewUniformShaderParameters;

						FBox VolumeBounds[TVC_MAX];

						FViewMatrices ModifiedViewMatrices = View.ViewMatrices;
						ModifiedViewMatrices.HackRemoveTemporalAAProjectionJitter();

						View.SetupUniformBufferParameters(
							SceneContext,
							ModifiedViewMatrices,
							ModifiedViewMatrices,
							VolumeBounds,
							TVC_MAX,
							CustomDepthViewUniformBufferParameters);

						Scene->UniformBuffers.CustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(CustomDepthViewUniformBufferParameters);

						if ((View.IsInstancedStereoPass() || View.bIsMobileMultiViewEnabled) && View.Family->Views.Num() > 0)
						{
							// When drawing the left eye in a stereo scene, set up the instanced custom depth uniform buffer with the right-eye data,
							// with the TAA jitter removed.
							const EStereoscopicPass StereoPassIndex = IStereoRendering::IsStereoEyeView(View) ? eSSP_RIGHT_EYE : eSSP_FULL;

							const FViewInfo& InstancedView = static_cast<const FViewInfo&>(View.Family->GetStereoEyeView(StereoPassIndex));

							CustomDepthViewUniformBufferParameters = *InstancedView.CachedViewUniformShaderParameters;
							ModifiedViewMatrices = InstancedView.ViewMatrices;
							ModifiedViewMatrices.HackRemoveTemporalAAProjectionJitter();

							InstancedView.SetupUniformBufferParameters(
								SceneContext,
								ModifiedViewMatrices,
								ModifiedViewMatrices,
								VolumeBounds,
								TVC_MAX,
								CustomDepthViewUniformBufferParameters);

							Scene->UniformBuffers.InstancedCustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(reinterpret_cast<FInstancedViewUniformShaderParameters&>(CustomDepthViewUniformBufferParameters));
						}
					}
					else
					{
						Scene->UniformBuffers.CustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(*View.CachedViewUniformShaderParameters);
						if ((View.IsInstancedStereoPass() || View.bIsMobileMultiViewEnabled) && View.Family->Views.Num() > 0)
						{
							const FViewInfo& InstancedView = Scene->UniformBuffers.GetInstancedView(View);
							Scene->UniformBuffers.InstancedCustomDepthViewUniformBuffer.UpdateUniformBufferImmediate(reinterpret_cast<FInstancedViewUniformShaderParameters&>(*InstancedView.CachedViewUniformShaderParameters));
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

					View.ParallelMeshDrawCommandPasses[EMeshPass::CustomDepth].DispatchDraw(nullptr, RHICmdList);
				});

				SceneContext.bCustomDepthIsValid = true;
			}
		}
	}
}
```