---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 4 ( FMobileSceneRenderer::RenderForward )"
date:   2022-04-24
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
             
이번 챕터에는 **FMobileSceneRenderer::RenderForward**에 대해 분석해볼 것이다.       
        
                       
------------------------------         

```cpp
FRHITexture* FMobileSceneRenderer::RenderForward(FRHICommandListImmediate& RHICmdList, const TArrayView<const FViewInfo*> ViewList)
{
	const FViewInfo& View = *ViewList[0];
	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);
				
	FRHITexture* SceneColor = nullptr;
	FRHITexture* SceneColorResolve = nullptr;
	FRHITexture* SceneDepth = nullptr;
	ERenderTargetActions ColorTargetAction = ERenderTargetActions::Clear_Store;
	EDepthStencilTargetActions DepthTargetAction = EDepthStencilTargetActions::ClearDepthStencil_DontStoreDepthStencil;

	// Verify using both MSAA sample count AND the scene color surface sample count, since on GLES you can't have MSAA color targets,
	// so the color target would be created without MSAA, and MSAA is achieved through magical means (the framebuffer, being MSAA,
	// tells the GPU "execute this renderpass as MSAA, and when you're done, automatically resolve and copy into this non-MSAA texture").
	bool bMobileMSAA = NumMSAASamples > 1 && SceneContext.GetSceneColorSurface()->GetNumSamples() > 1;

	static const auto CVarMobileMultiView = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("vr.MobileMultiView"));
	const bool bIsMultiViewApplication = (CVarMobileMultiView && CVarMobileMultiView->GetValueOnAnyThread() != 0);
	
	if (bGammaSpace && !bRenderToSceneColor)
	{
		if (bMobileMSAA)
		{
			SceneColor = SceneContext.GetSceneColorSurface();
			SceneColorResolve = ViewFamily.RenderTarget->GetRenderTargetTexture();
			ColorTargetAction = ERenderTargetActions::Clear_Resolve;
			RHICmdList.Transition(FRHITransitionInfo(SceneColorResolve, ERHIAccess::Unknown, ERHIAccess::RTV | ERHIAccess::ResolveDst));
		}
		else
		{
			SceneColor = ViewFamily.RenderTarget->GetRenderTargetTexture();
			RHICmdList.Transition(FRHITransitionInfo(SceneColor, ERHIAccess::Unknown, ERHIAccess::RTV));
		}
		SceneDepth = SceneContext.GetSceneDepthSurface();
	}
	else
	{
		SceneColor = SceneContext.GetSceneColorSurface();
		if (bMobileMSAA)
		{
			SceneColorResolve = SceneContext.GetSceneColorTexture();
			ColorTargetAction = ERenderTargetActions::Clear_Resolve;
			RHICmdList.Transition(FRHITransitionInfo(SceneColorResolve, ERHIAccess::Unknown, ERHIAccess::RTV | ERHIAccess::ResolveDst));
		}
		else
		{
			SceneColorResolve = nullptr;
			ColorTargetAction = ERenderTargetActions::Clear_Store;
		}

		SceneDepth = SceneContext.GetSceneDepthSurface();
				
		if (bRequiresMultiPass)
		{	
			// store targets after opaque so translucency render pass can be restarted
			ColorTargetAction = ERenderTargetActions::Clear_Store;
			DepthTargetAction = EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil;
		}
						
		if (bKeepDepthContent)
		{
			// store depth if post-processing/capture needs it
			DepthTargetAction = EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil;
		}
	}

	if (bIsFullPrepassEnabled)
	{
		ERenderTargetActions DepthTarget = MakeRenderTargetActions(ERenderTargetLoadAction::ELoad, GetStoreAction(GetDepthActions(DepthTargetAction)));
		ERenderTargetActions StencilTarget = MakeRenderTargetActions(ERenderTargetLoadAction::ELoad, GetStoreAction(GetStencilActions(DepthTargetAction)));
		DepthTargetAction = MakeDepthStencilTargetActions(DepthTarget, StencilTarget);
	}

	FRHITexture* ShadingRateTexture = nullptr;
	
	if (!View.bIsSceneCapture && !View.bIsReflectionCapture)
	{
		TRefCountPtr<IPooledRenderTarget> ShadingRateTarget = GVRSImageManager.GetMobileVariableRateShadingImage(ViewFamily);
		if (ShadingRateTarget.IsValid())
		{
			ShadingRateTexture = ShadingRateTarget->GetRenderTargetItem().ShaderResourceTexture;
		}
	}

	FRHIRenderPassInfo SceneColorRenderPassInfo(
		SceneColor,
		ColorTargetAction,
		SceneColorResolve,
		SceneDepth,
		DepthTargetAction,
		nullptr, // we never resolve scene depth on mobile
		ShadingRateTexture,
		VRSRB_Sum,
		FExclusiveDepthStencil::DepthWrite_StencilWrite
	);
	SceneColorRenderPassInfo.SubpassHint = ESubpassHint::DepthReadSubpass;
	if (!bIsFullPrepassEnabled)
	{
		SceneColorRenderPassInfo.NumOcclusionQueries = ComputeNumOcclusionQueriesToBatch();
		SceneColorRenderPassInfo.bOcclusionQueries = SceneColorRenderPassInfo.NumOcclusionQueries != 0;
	}
	//if the scenecolor isn't multiview but the app is, need to render as a single-view multiview due to shaders
	SceneColorRenderPassInfo.MultiViewCount = View.bIsMobileMultiViewEnabled ? 2 : (bIsMultiViewApplication ? 1 : 0);

	RHICmdList.BeginRenderPass(SceneColorRenderPassInfo, TEXT("SceneColorRendering"));
	
	if (GIsEditor && !View.bIsSceneCapture)
	{
		DrawClearQuad(RHICmdList, Views[0].BackgroundColor);
	}

	if (!bIsFullPrepassEnabled)
	{
		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLM_MobilePrePass));
		// Depth pre-pass
		RenderPrePass(RHICmdList);
	}
	
	// Opaque and masked
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Opaque));
	RenderMobileBasePass(RHICmdList, ViewList);
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
	if (ViewFamily.UseDebugViewPS())
	{
		// Here we use the base pass depth result to get z culling for opaque and masque.
		// The color needs to be cleared at this point since shader complexity renders in additive.
		DrawClearQuad(RHICmdList, FLinearColor::Black);
		RenderMobileDebugView(RHICmdList, ViewList);
		RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	}
#endif // !(UE_BUILD_SHIPPING || UE_BUILD_TEST)

	const bool bAdrenoOcclusionMode = CVarMobileAdrenoOcclusionMode.GetValueOnRenderThread() != 0;
	if (!bIsFullPrepassEnabled)
	{
		
		if (!bAdrenoOcclusionMode)
		{
			// Issue occlusion queries
			RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Occlusion));
			RenderOcclusion(RHICmdList);
			RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
		}
	}

	{
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(ViewExtensionPostRenderBasePass);
		QUICK_SCOPE_CYCLE_COUNTER(STAT_FMobileSceneRenderer_ViewExtensionPostRenderBasePass);
		for (int32 ViewExt = 0; ViewExt < ViewFamily.ViewExtensions.Num(); ++ViewExt)
		{
			for (int32 ViewIndex = 0; ViewIndex < ViewFamily.Views.Num(); ++ViewIndex)
			{
				ViewFamily.ViewExtensions[ViewExt]->PostRenderBasePass_RenderThread(RHICmdList, Views[ViewIndex]);
			}
		}
	}
		
	// Split if we need to render translucency in a separate render pass
	// Split if we need to render pixel projected reflection
	if (bRequiresMultiPass || bRequiresPixelProjectedPlanarRelfectionPass)
	{
		RHICmdList.EndRenderPass();
	}
	   
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Translucency));

		
	// Restart translucency render pass if needed
	if (bRequiresMultiPass || bRequiresPixelProjectedPlanarRelfectionPass)
	{
		check(RHICmdList.IsOutsideRenderPass());

		// Make a copy of the scene depth if the current hardware doesn't support reading and writing to the same depth buffer
		ConditionalResolveSceneDepth(RHICmdList, View);

		if (bRequiresPixelProjectedPlanarRelfectionPass)
		{
			const FPlanarReflectionSceneProxy* PlanarReflectionSceneProxy = Scene ? Scene->GetForwardPassGlobalPlanarReflection() : nullptr;
			RenderPixelProjectedReflection(RHICmdList, SceneContext, PlanarReflectionSceneProxy);

			FRHITransitionInfo TranslucentRenderPassTransitions[] = {
			FRHITransitionInfo(SceneColor, ERHIAccess::SRVMask, ERHIAccess::RTV),
			FRHITransitionInfo(SceneDepth, ERHIAccess::SRVMask, ERHIAccess::DSVWrite)
			};
			RHICmdList.Transition(MakeArrayView(TranslucentRenderPassTransitions, UE_ARRAY_COUNT(TranslucentRenderPassTransitions)));
		}

		DepthTargetAction = EDepthStencilTargetActions::LoadDepthStencil_DontStoreDepthStencil;
		FExclusiveDepthStencil::Type ExclusiveDepthStencil = FExclusiveDepthStencil::DepthRead_StencilRead;
		if (bModulatedShadowsInUse)
		{
			// FIXME: modulated shadows write to stencil
			ExclusiveDepthStencil = FExclusiveDepthStencil::DepthRead_StencilWrite;
		}

		// The opaque meshes used for mobile pixel projected reflection have to write depth to depth RT, since we only render the meshes once if the quality level is less or equal to BestPerformance 
		if (IsMobilePixelProjectedReflectionEnabled(View.GetShaderPlatform())
			&& GetMobilePixelProjectedReflectionQuality() == EMobilePixelProjectedReflectionQuality::BestPerformance)
		{
			ExclusiveDepthStencil = FExclusiveDepthStencil::DepthWrite_StencilWrite;
		}

		if (bKeepDepthContent && !bMobileMSAA)
		{
			DepthTargetAction = EDepthStencilTargetActions::LoadDepthStencil_StoreDepthStencil;
		}

#if PLATFORM_HOLOLENS
		if (bShouldRenderDepthToTranslucency)
		{
			ExclusiveDepthStencil = FExclusiveDepthStencil::DepthWrite_StencilWrite;
		}
#endif

		FRHIRenderPassInfo TranslucentRenderPassInfo(
			SceneColor,
			SceneColorResolve ? ERenderTargetActions::Load_Resolve : ERenderTargetActions::Load_Store,
			SceneColorResolve,
			SceneDepth,
			DepthTargetAction, 
			nullptr,
			ShadingRateTexture,
			VRSRB_Sum,
			ExclusiveDepthStencil
		);
		TranslucentRenderPassInfo.NumOcclusionQueries = 0;
		TranslucentRenderPassInfo.bOcclusionQueries = false;
		TranslucentRenderPassInfo.SubpassHint = ESubpassHint::DepthReadSubpass;
		RHICmdList.BeginRenderPass(TranslucentRenderPassInfo, TEXT("SceneColorTranslucencyRendering"));
	}

	// scene depth is read only and can be fetched
	RHICmdList.NextSubpass();

	if (!View.bIsPlanarReflection)
	{
		if (ViewFamily.EngineShowFlags.Decals)
		{
			CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderDecals);
			RenderDecals(RHICmdList);
		}

		if (ViewFamily.EngineShowFlags.DynamicShadows)
		{
			CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderShadowProjections);
			RenderModulatedShadowProjections(RHICmdList);
		}
	}
	
	// Draw translucency.
	if (ViewFamily.EngineShowFlags.Translucency)
	{
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderTranslucency);
		SCOPE_CYCLE_COUNTER(STAT_TranslucencyDrawTime);
		RenderTranslucency(RHICmdList, ViewList);
		FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
		RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	}

	if (!bIsFullPrepassEnabled)
	{
		if (bAdrenoOcclusionMode)
		{
			RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Occlusion));
			// flush
			RHICmdList.SubmitCommandsHint();
			bSubmitOffscreenRendering = false; // submit once
			// Issue occlusion queries
			RenderOcclusion(RHICmdList);
			RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
		}
	}

	// Pre-tonemap before MSAA resolve (iOS only)
	if (!bGammaSpace)
	{
		PreTonemapMSAA(RHICmdList);
	}

	// End of scene color rendering
	RHICmdList.EndRenderPass();

	return SceneColorResolve ? SceneColorResolve : SceneColor;
}
```