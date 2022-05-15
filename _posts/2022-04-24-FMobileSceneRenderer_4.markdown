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

	// ⭐
	// 기본적으로 false
	static const auto CVarMobileMultiView = IConsoleManager::Get().FindTConsoleVariableDataInt(TEXT("vr.MobileMultiView"));
	const bool bIsMultiViewApplication = (CVarMobileMultiView && CVarMobileMultiView->GetValueOnAnyThread() != 0);
	// ⭐
	
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
		// ⭐
		// Color를 Write할 텍스쳐
		// 렌더 타겟으로 사용될 Color 텍스쳐
		SceneColor = SceneContext.GetSceneColorSurface();
		// ⭐
		if (bMobileMSAA)
		{
			// ⭐
			// MSAA를 적용하는 경우,
			// 위의 SceneColor는 멀티샘플링된다.
			// 그럼 그 멀티샘플링된 SceneColor를 멀티샘플링되지 않는 텍스쳐로 옮기는데 그 대상이 되는 텍스쳐가 이 텍스쳐이다.
			//
			// Resolve : 멀티샘플링된 텍스쳐를 멀티샘플링 되지 않는 텍스쳐로 Blend한다.
			SceneColorResolve = SceneContext.GetSceneColorTexture();
			ColorTargetAction = ERenderTargetActions::Clear_Resolve;
			RHICmdList.Transition(FRHITransitionInfo(SceneColorResolve, ERHIAccess::Unknown, ERHIAccess::RTV | ERHIAccess::ResolveDst));
			// ⭐
		}
		else
		{
			SceneColorResolve = nullptr;
			// ⭐
			// 렌더타겟으로 사용될 SceneColor 텍스쳐를 Clear한 후 색을 Write한다.
			ColorTargetAction = ERenderTargetActions::Clear_Store;
			// ⭐
		}

		// ⭐
		// Depth 텍스쳐
		SceneDepth = SceneContext.GetSceneDepthSurface();
		// ⭐

		// ⭐
		// bool FMobileSceneRenderer::RequiresMultiPass(FRHICommandListImmediate& RHICmdList, const FViewInfo& View) const
		// {
		// 		// Vulkan uses subpasses
		// 		if (IsVulkanPlatform(ShaderPlatform))
		// 		{
		// 			return false;
		// 		}
		// 
		// 		// All iOS support frame_buffer_fetch
		// 		if (IsMetalMobilePlatform(ShaderPlatform))
		// 		{
		//			return false;
		//		}
		//
		//		if (IsMobileDeferredShadingEnabled(ShaderPlatform))
		//		{
		//			// TODO: add GL support
		//			return true;
		//		}
		//
		//	 	// Some Androids support frame_buffer_fetch
		//		if (IsAndroidOpenGLESPlatform(ShaderPlatform) && (GSupportsShaderFramebufferFetch || GSupportsShaderDepthStencilFetch))
		//		{
		//			return false;
		//		}
		//	
		// 		// Always render reflection capture in single pass
		//		if (View.bIsPlanarReflection || View.bIsSceneCapture)
		//		{
		//			return false;
		//		}
		//
		//		// Always render LDR in single pass
		//		if (!IsMobileHDR())
		//		{
		//			return false;
		//		}
		//
		// 		// MSAA depth can't be sampled or resolved, unless we are on PC (no vulkan)
		//		if (NumMSAASamples > 1 && !IsSimulatedPlatform(ShaderPlatform))
		//		{
		//			return false;
		//		}
		//
		//		return true;
		//	}
		//
		// 멀티패스가 필요한 경우
		//
		if (bRequiresMultiPass)
		// ⭐
		{	
			// ⭐
			// store targets after opaque so translucency render pass can be restarted
			//
			// Color 텍스쳐를 보존해둔다. 이후 투명한 오브젝트를 별개 Pass에서 처리할 때 렌더타겟으로 다시 사용한다.
			//
			ColorTargetAction = ERenderTargetActions::Clear_Store;
			// ⭐
			DepthTargetAction = EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil;
		}
		
		//
		// bKeepDepthContent = 
		// 		bRequiresMultiPass || 
		// 		bForceDepthResolve ||
		// 		bRequiresPixelProjectedPlanarRelfectionPass ||
		// 		bSeparateTranslucencyActive ||
		// 		Views[0].bIsReflectionCapture ||
		// 		(bDeferredShading && bPostProcessUsesSceneDepth) ||
		// 		bShouldRenderVelocities ||
		// 		bIsFullPrepassEnabled;
		if (bKeepDepthContent)
		{
			// ⭐
			// store depth if post-processing/capture needs it
			//
			// 포스트프로세싱이나 캡쳐를 하는 경우 Depth Buffer를 보존 ( StoreDepthStencil )해두어야한다.
			//
			DepthTargetAction = EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil;
			// ⭐
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

	// ⭐⭐⭐⭐⭐⭐⭐
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

	// ⭐
	// Prepass를 사용하지 않는 것으로 앞의 글에서 분석을 했다.
	// 앞서 Depth Prepass 단계를 수행하지 않았으니, 여기서 Occlusion Query를 수행한다.
	if (!bIsFullPrepassEnabled)
	{
		SceneColorRenderPassInfo.NumOcclusionQueries = ComputeNumOcclusionQueriesToBatch();
		SceneColorRenderPassInfo.bOcclusionQueries = SceneColorRenderPassInfo.NumOcclusionQueries != 0;
	}
	// ⭐

	//if the scenecolor isn't multiview but the app is, need to render as a single-view multiview due to shaders
	SceneColorRenderPassInfo.MultiViewCount = View.bIsMobileMultiViewEnabled ? 2 : (bIsMultiViewApplication ? 1 : 0);

	RHICmdList.BeginRenderPass(SceneColorRenderPassInfo, TEXT("SceneColorRendering"));
	// ⭐⭐⭐⭐⭐⭐⭐
	
	if (GIsEditor && !View.bIsSceneCapture)
	{
		DrawClearQuad(RHICmdList, Views[0].BackgroundColor);
	}

	if (!bIsFullPrepassEnabled)
	{
		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLM_MobilePrePass));

		// ⭐
		//
		// Depth pre-pass
		//
		// Depth Pre-Pass를 수행한다.
		// 월드의 Opaque한 오브젝트들에 대해 픽셀 쉐이딩이 생략된(간소화된) Draw를 수행한다. ( 그냥 Depth 버퍼만 채우는 것이다. )
		// 이는 아래 BasePass에서 오브젝트들에 대한 렌더링을 수행할 때 Early-Z를 극대화하기 위함이다.
		//
		// 여기서 RenderPrePass 함수를 호출하지만 기본 셋팅으로는 아무 동작도 하지 않는다.
		// 아래의 조건을 충족해야한다.
		//
		// void FMobileSceneRenderer::RenderPrePass(FRHICommandListImmediate& RHICmdList)
		// {
		//		...
		//		...
		//		...
		//		if (Scene->EarlyZPassMode == DDM_MaskedOnly || Scene->EarlyZPassMode == DDM_AllOpaque)
		//		{
		//			...
		//			...
		//			...
		//		}
		// }
		// 
		RenderPrePass(RHICmdList);
		// ⭐
	}
	
	// Opaque and masked
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Opaque));

	// ⭐⭐⭐⭐⭐⭐⭐
	// 아래쪽으로 내려가 자세한 분석을 보세요.
	RenderMobileBasePass(RHICmdList, ViewList);
	// ⭐⭐⭐⭐⭐⭐⭐

	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
	// ...
	// ... 디버깅용 코드
	// ...
#endif // !(UE_BUILD_SHIPPING || UE_BUILD_TEST)

	// ⭐
	// 기본 값은 false
	// 
	// static TAutoConsoleVariable<int32> CVarMobileAdrenoOcclusionMode(
	// TEXT("r.Mobile.AdrenoOcclusionMode"),
	// 0,
	// TEXT("0: Render occlusion queries after the base pass (default).\n")
	// TEXT("1: Render occlusion queries after translucency and a flush, which can help Adreno devices in GL mode."),
	// ECVF_RenderThreadSafe);
	//
	const bool bAdrenoOcclusionMode = CVarMobileAdrenoOcclusionMode.GetValueOnRenderThread() != 0;
	// ⭐

	// ⭐
	// 일반적인 셋팅의 경우 여기서 Occlusion Query 수행 커맨드가 RHI 스레드로 전송된다.
	if (!bIsFullPrepassEnabled)
	{
		
		if (!bAdrenoOcclusionMode)
		{
			// Issue occlusion queries
			RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Occlusion));

			// ⭐⭐⭐⭐⭐⭐⭐
			//
			// BasePass 후 Occlusion Query를 발행한다.
			// 기본 프로젝트 셋팅 값으로는 여기서 Occlusion Query를 날린다.
			// 위의 BasePass에서 Depth Buffer를 채웠다.
			//
			// 아래쪽으로 내려가 자세한 분석을 보세요.
			RenderOcclusion(RHICmdList);
			// ⭐⭐⭐⭐⭐⭐⭐

			RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
		}
	}
	// ⭐

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
	
	// ⭐⭐⭐⭐⭐⭐⭐
	// 투명, 반투명한 오브젝트를 렌더링한다.
	//
	// Draw translucency.
	if (ViewFamily.EngineShowFlags.Translucency)
	{
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderTranslucency);
		SCOPE_CYCLE_COUNTER(STAT_TranslucencyDrawTime);
		RenderTranslucency(RHICmdList, ViewList);
		FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
		RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	}
	// ⭐⭐⭐⭐⭐⭐⭐

	if (!bIsFullPrepassEnabled)
	{
		// ⭐
		// 기본 값은 false
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
		// ⭐
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

```cpp
void FMobileSceneRenderer::RenderMobileBasePass(FRHICommandListImmediate& RHICmdList, const TArrayView<const FViewInfo*> PassViews)
{
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderBasePass);
	SCOPED_DRAW_EVENT(RHICmdList, MobileBasePass);
	SCOPE_CYCLE_COUNTER(STAT_BasePassDrawTime);
	SCOPED_GPU_STAT(RHICmdList, Basepass);

	for (int32 ViewIndex = 0; ViewIndex < PassViews.Num(); ViewIndex++)
	{
		SCOPED_CONDITIONAL_DRAW_EVENTF(RHICmdList, EventView, Views.Num() > 1, TEXT("View%d"), ViewIndex);
		const FViewInfo& View = *PassViews[ViewIndex];
		if (!View.ShouldRenderView())
		{
			continue;
		}

		// ⭐
		// Basepass 렌더링에 필요한 UniformBuffer를 업데이트한다.
		if (Scene->UniformBuffers.UpdateViewUniformBuffer(View))
		{
			// ⭐
			// Opaque Base Pass를 렌더링하는데 필요한 GPU Uniform Buffer를 업데이트한다.
			//
			// 아래쪽으로 내려가 자세한 분석을 읽으세요.
			UpdateOpaqueBasePassUniformBuffer(RHICmdList, View);
			// ⭐

			// ⭐
			// Directional Light를 렌더링하는데 필요한 GPU Uniform Buffer를 업데이트한다.
			UpdateDirectionalLightUniformBuffers(RHICmdList, View);
			// ⭐
		}
		// ⭐
		
		// ⭐⭐⭐⭐⭐⭐⭐
		// Viewport를 셋팅하고 BasePass의 MeshDrawCommnad에 대한 Draw 커맨드를 보낸다.
		RHICmdList.SetViewport(View.ViewRect.Min.X, View.ViewRect.Min.Y, 0, View.ViewRect.Max.X, View.ViewRect.Max.Y, 1);
		View.ParallelMeshDrawCommandPasses[EMeshPass::BasePass].DispatchDraw(nullptr, RHICmdList);
		// ⭐⭐⭐⭐⭐⭐⭐
		
		if (View.Family->EngineShowFlags.Atmosphere)
		{
			View.ParallelMeshDrawCommandPasses[EMeshPass::SkyPass].DispatchDraw(nullptr, RHICmdList);
		}

		// ⭐
		// editor primitives
		//
		// 에디터용 Primitive를 렌더링한다.
		{
			FMeshPassProcessorRenderState DrawRenderState(View, Scene->UniformBuffers.MobileOpaqueBasePassUniformBuffer);
			DrawRenderState.SetBlendState(TStaticBlendStateWriteMask<CW_RGBA>::GetRHI());
			DrawRenderState.SetDepthStencilAccess(Scene->DefaultBasePassDepthStencilAccess);
			DrawRenderState.SetDepthStencilState(TStaticDepthStencilState<true, CF_DepthNearOrEqual>::GetRHI());
			RenderMobileEditorPrimitives(RHICmdList, View, DrawRenderState);
		}
		// ⭐
	}
}

void FMobileSceneRenderer::UpdateOpaqueBasePassUniformBuffer(FRHICommandListImmediate& RHICmdList, const FViewInfo& View)
{
	FMobileBasePassUniformParameters Parameters;

	// ⭐
	// Mobile Base Pass를 그리기 위한 Uniform Buffer Parameter를 셋팅한다.
	SetupMobileBasePassUniformParameters(RHICmdList, View, false /* bTranslucentPass */, false /* bCanUseCSM */, Parameters);
	// ⭐

	// ⭐
	// Uniform Buffer Parameter를 가지고 MobileOpaqueBasePass Uniform Buffer를 업데이트한다.
	Scene->UniformBuffers.MobileOpaqueBasePassUniformBuffer.UpdateUniformBufferImmediate(Parameters);
	// ⭐

	// ⭐
	// Cascade Shadow Map을 그리기 위한 Uniform Buffer Parameter를 셋팅한다.
	// 
	// Shadow Map을 그린다의 의미는? : 그냥 Direction Light에 카메라를 두고 방향을 바라보게 하여 월드를 한번더 렌더링하는 것이다.
	// 그 이상도 그 이하도 아니다. 대신 Pixel Shading 단계에서 그냥 점만 찍는 것이다. Shadow Map은 Depth 값만 알면 되니깐 말이다... 
	//
	// 그렇기 때문에 위에 Opaque Base Pass 렌더링을 위한 Uniform Buffer Parameter 업데이트 함수와 같은 함수를 사용한다.
	// Base Pass를 렌더링에 사용되는 대부분의 Uniform Buffer 데이터 값이 Shadow Map을 그릴 때도 동일하게 필요하기 때문이다.
	SetupMobileBasePassUniformParameters(RHICmdList, View, false /* bTranslucentPass */, true /* bCanUseCSM */, Parameters);
	// ⭐

	// ⭐
	// Uniform Buffer Parameter를 가지고 MobileCSMOpaqueBasePass Uniform Buffer를 업데이트한다.
	Scene->UniformBuffers.MobileCSMOpaqueBasePassUniformBuffer.UpdateUniformBufferImmediate(Parameters);	
	// ⭐
}

// ⭐⭐⭐⭐⭐⭐⭐
// Mobile Base Pass에 필요한 Uniform Buffer 데이터를 매개변수 BasePassParameters에 저장한다.
void SetupMobileBasePassUniformParameters(
// ⭐⭐⭐⭐⭐⭐⭐
	FRHICommandListImmediate& RHICmdList, 
	const FViewInfo& View, 
	bool bTranslucentPass, 
	bool bCanUseCSM,
	FMobileBasePassUniformParameters& BasePassParameters)
{
	SetupFogUniformParameters(nullptr, View, BasePassParameters.Fog);

	const FScene* Scene = View.Family->Scene ? View.Family->Scene->GetRenderScene() : nullptr;
	const FPlanarReflectionSceneProxy* ReflectionSceneProxy = Scene ? Scene->GetForwardPassGlobalPlanarReflection() : nullptr;
	SetupPlanarReflectionUniformParameters(View, ReflectionSceneProxy, BasePassParameters.PlanarReflection);
	BasePassParameters.UseCSM = bCanUseCSM ? 1 : 0;

	EMobileSceneTextureSetupMode SetupMode = EMobileSceneTextureSetupMode::None;
	if (bTranslucentPass)
	{
		// ⭐
		// 이 함수를 호출한 Pass가 TranslucentPass인 경우, SceneTexture에 Scene
		// 아래 FMobileSceneRenderer::RenderTranslucency 함수 분석에서 볼 수 있다.
		//
		// 아래 SetupMobileSceneTextureUniformParameters 함수에서 아래와 같이 셋팅된다.
		// SceneTextureParameters.SceneColorTexture = GetRDG(SceneContext.GetSceneColor());
		SetupMode |= EMobileSceneTextureSetupMode::SceneColor;
		// ⭐
	}
	if (View.bCustomDepthStencilValid)
	{
		// ⭐
		// SceneTextureParameters.CustomDepthTexture = GetRDG(SceneContext.MobileCustomDepth);
		// 
		// 아래 SetupMobileSceneTextureUniformParameters 함수에서 아래와 같이 셋팅된다.
		SetupMode |= EMobileSceneTextureSetupMode::CustomDepth;
		// ⭐
	}

	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);

	// ⭐
	// BasePassParameters.SceneTextures에 알맞은 값들을 셋팅한다.
	// ex)
	SetupMobileSceneTextureUniformParameters(SceneContext, SetupMode, BasePassParameters.SceneTextures);
	// ⭐

	BasePassParameters.PreIntegratedGFTexture = GSystemTextures.PreintegratedGF->GetRenderTargetItem().ShaderResourceTexture;
	BasePassParameters.PreIntegratedGFSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();

	if (GPixelProjectedReflectionMobileOutputs.IsValid())
	{
		if (bTranslucentPass)
		{
			BasePassParameters.PlanarReflection.PlanarReflectionTexture = GPixelProjectedReflectionMobileOutputs.PixelProjectedReflectionTexture->GetRenderTargetItem().ShaderResourceTexture;
			if (GetMobilePixelProjectedReflectionQuality() <= EMobilePixelProjectedReflectionQuality::BestPerformance)
			{
				// We only render the meshes used for pixel projected reflection once and it could cause color bleeding artifact if we use bilinear filter.
				BasePassParameters.PlanarReflection.PlanarReflectionSampler = TStaticSamplerState<SF_Point, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
			}
			else
			{
				// We render the meshes used for pixel projected reflection twice, so we could use bilinear filter.
				BasePassParameters.PlanarReflection.PlanarReflectionSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
			}
		}
		else
		{
			// Clear the ReflectionPlane to skip planar reflection on opaque mesh when the PPR is enabled because we render the reflection meshes used for pixel projected reflection in the translucent pass.
			BasePassParameters.PlanarReflection.ReflectionPlane.Set(0.0f, 0.0f, 0.0f, 0.0f);
		}
	}

	BasePassParameters.EyeAdaptationBuffer = GetEyeAdaptationBuffer(View);

	// ⭐
	// BasePass 렌더링에 사용될 Ambient Occlusion 관련 데이터를 Base Pass Uniform Buffer Parameter를 셋팅한다.
	if (!bTranslucentPass && GAmbientOcclusionMobileOutputs.IsValid() && IsUsingMobileAmbientOcclusion(View.GetShaderPlatform()))
	{
		BasePassParameters.AmbientOcclusionTexture = GAmbientOcclusionMobileOutputs.AmbientOcclusionTexture->GetRenderTargetItem().ShaderResourceTexture;
	}
	else
	{
		BasePassParameters.AmbientOcclusionTexture = GSystemTextures.WhiteDummy->GetRenderTargetItem().ShaderResourceTexture;
	}
	BasePassParameters.AmbientOcclusionSampler = TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
	BasePassParameters.AmbientOcclusionStaticFraction = FMath::Clamp(View.FinalPostProcessSettings.AmbientOcclusionStaticFraction, 0.0f, 1.0f);
	// ⭐


	// ⭐
	// BasePass 렌더링에 사용될 Screen Space Shadow 관련 데이터를 Base Pass Uniform Buffer Parameter를 셋팅한다.
	bool bRequiresDistanceFieldShadowingPass = IsMobileDistanceFieldShadowingEnabled(View.GetShaderPlatform());
	if (bRequiresDistanceFieldShadowingPass && GScreenSpaceShadowMaskTextureMobileOutputs.ScreenSpaceShadowMaskTextureMobile.IsValid())
	{
		BasePassParameters.ScreenSpaceShadowMaskTexture = GScreenSpaceShadowMaskTextureMobileOutputs.ScreenSpaceShadowMaskTextureMobile->GetRenderTargetItem().ShaderResourceTexture;
		BasePassParameters.ScreenSpaceShadowMaskSampler = TStaticSamplerState<SF_Point, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
	}
	else
	{
		BasePassParameters.ScreenSpaceShadowMaskTexture = GSystemTextures.WhiteDummy->GetRenderTargetItem().ShaderResourceTexture;
		BasePassParameters.ScreenSpaceShadowMaskSampler = TStaticSamplerState<SF_Point, AM_Clamp, AM_Clamp, AM_Clamp>::GetRHI();
	}
	// ⭐

}
```

```cpp
void FMobileSceneRenderer::RenderOcclusion(FRHICommandListImmediate& RHICmdList)
{
	if (!DoOcclusionQueries(FeatureLevel))
	{
		return;
	}

	{
		SCOPED_NAMED_EVENT(FMobileSceneRenderer_BeginOcclusionTests, FColor::Emerald);

		// ⭐
		// Projected Shadow, PlanarReflection들에 대한 Occlusion Query를 생성한다.
		const FViewOcclusionQueriesPerView QueriesPerView = AllocateOcclusionTests(Scene, VisibleLightInfos, Views);
		// ⭐

		if (QueriesPerView.Num())
		{
			// ⭐⭐⭐⭐⭐⭐⭐
			// RHI 스레드에 Occlusion Query 커맨드를 보낸다.
			BeginOcclusionTests(RHICmdList, Views, FeatureLevel, QueriesPerView, 1.0f);
			// ⭐⭐⭐⭐⭐⭐⭐
		}
	}

	// ⭐
	// RHI 스레드가 별개로 존재하는 경우,
	// 렌더 스레드 커맨트 큐에 쌓여있는 Occlusion Query 수행 커맨드들을 곧바로 RHI 스레드로 전송한다.
	// 
	// RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	// RHICmdList.PollRenderQueryResults();
	FRDGBuilder GraphBuilder(RHICmdList);
	FenceOcclusionTests(GraphBuilder);
	GraphBuilder.Execute();
	// ⭐
}


static FViewOcclusionQueriesPerView AllocateOcclusionTests(const FScene* Scene, TArrayView<const FVisibleLightInfo> VisibleLightInfos, TArrayView<FViewInfo> Views)
{
	SCOPED_NAMED_EVENT(FSceneRenderer_AllocateOcclusionTestsOcclusionTests, FColor::Emerald);

	const ERHIFeatureLevel::Type FeatureLevel = Scene->GetFeatureLevel();
	const int32 NumBufferedFrames = FOcclusionQueryHelpers::GetNumBufferedFrames(FeatureLevel);

	bool bBatchedQueries = false;

	FViewOcclusionQueriesPerView QueriesPerView;
	QueriesPerView.AddDefaulted(Views.Num());

	// Perform occlusion queries for each view
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		FViewInfo& View = Views[ViewIndex];
		FViewOcclusionQueries& ViewQuery = QueriesPerView[ViewIndex];
		FSceneViewState* ViewState = (FSceneViewState*)View.State;
		const FSceneViewFamily& ViewFamily = *View.Family;

		if (ViewState && !View.bDisableQuerySubmissions)
		{
			// Issue this frame's occlusion queries (occlusion queries from last frame may still be in flight)
			const uint32 QueryIndex = FOcclusionQueryHelpers::GetQueryIssueIndex(ViewState->PendingPrevFrameNumber, NumBufferedFrames);
			FSceneViewState::ShadowKeyOcclusionQueryMap& ShadowOcclusionQueryMap = ViewState->ShadowOcclusionQueryMaps[QueryIndex];

			// Clear primitives which haven't been visible recently out of the occlusion history, and reset old pending occlusion queries.
			ViewState->TrimOcclusionHistory(ViewFamily.CurrentRealTime, ViewFamily.CurrentRealTime - GEngine->PrimitiveProbablyVisibleTime, ViewFamily.CurrentRealTime, ViewState->OcclusionFrameCounter);

			// Give back all these occlusion queries to the pool.
			ShadowOcclusionQueryMap.Reset();

			if (FeatureLevel > ERHIFeatureLevel::ES3_1)
			{
				for (TSparseArray<FLightSceneInfoCompact>::TConstIterator LightIt(Scene->Lights); LightIt; ++LightIt)
				{
					const FVisibleLightInfo& VisibleLightInfo = VisibleLightInfos[LightIt.GetIndex()];

					for (int32 ShadowIndex = 0; ShadowIndex < VisibleLightInfo.AllProjectedShadows.Num(); ShadowIndex++)
					{
						const FProjectedShadowInfo& ProjectedShadowInfo = *VisibleLightInfo.AllProjectedShadows[ShadowIndex];

						if (ProjectedShadowInfo.DependentView && ProjectedShadowInfo.DependentView != &View)
						{
							continue;
						}

						if (!IsShadowCacheModeOcclusionQueryable(ProjectedShadowInfo.CacheMode))
						{
							// Only query one of the cache modes for each shadow
							continue;
						}

						if (ProjectedShadowInfo.bOnePassPointLightShadow)
						{
							FRHIRenderQuery* ShadowOcclusionQuery;
							if (AllocateProjectedShadowOcclusionQuery(View, ProjectedShadowInfo, NumBufferedFrames, SOQ_LightInfluenceSphere, ShadowOcclusionQuery))
							{
								ViewQuery.PointLightQueryInfos.Add(&ProjectedShadowInfo);
								ViewQuery.PointLightQueries.Add(ShadowOcclusionQuery);
								checkSlow(ViewQuery.PointLightQueryInfos.Num() == ViewQuery.PointLightQueries.Num());
								bBatchedQueries = true;
							}
						}
						else if (ProjectedShadowInfo.IsWholeSceneDirectionalShadow())
						{
							// Don't query the first cascade, it is always visible
							if (GOcclusionCullCascadedShadowMaps && ProjectedShadowInfo.CascadeSettings.ShadowSplitIndex > 0)
							{
								FRHIRenderQuery* ShadowOcclusionQuery;
								if (AllocateProjectedShadowOcclusionQuery(View, ProjectedShadowInfo, NumBufferedFrames, SOQ_None, ShadowOcclusionQuery))
								{
									ViewQuery.CSMQueryInfos.Add(&ProjectedShadowInfo);
									ViewQuery.CSMQueries.Add(ShadowOcclusionQuery);
									checkSlow(ViewQuery.CSMQueryInfos.Num() == ViewQuery.CSMQueries.Num());
									bBatchedQueries = true;
								}
							}
						}
						else if (
							// Don't query preshadows, since they are culled if their subject is occluded.
							!ProjectedShadowInfo.bPreShadow
							// Don't query if any subjects are visible because the shadow frustum will be definitely unoccluded
							&& !ProjectedShadowInfo.SubjectsVisible(View))
						{
							FRHIRenderQuery* ShadowOcclusionQuery;
							if (AllocateProjectedShadowOcclusionQuery(View, ProjectedShadowInfo, NumBufferedFrames, SOQ_NearPlaneVsShadowFrustum, ShadowOcclusionQuery))
							{
								ViewQuery.ShadowQuerieInfos.Add(&ProjectedShadowInfo);
								ViewQuery.ShadowQueries.Add(ShadowOcclusionQuery);
								checkSlow(ViewQuery.ShadowQuerieInfos.Num() == ViewQuery.ShadowQueries.Num());
								bBatchedQueries = true;
							}
						}
					}

					// Issue occlusion queries for all per-object projected shadows that we would have rendered but were occluded last frame.
					for (int32 ShadowIndex = 0; ShadowIndex < VisibleLightInfo.OccludedPerObjectShadows.Num(); ShadowIndex++)
					{
						const FProjectedShadowInfo& ProjectedShadowInfo = *VisibleLightInfo.OccludedPerObjectShadows[ShadowIndex];
						FRHIRenderQuery* ShadowOcclusionQuery;
						if (AllocateProjectedShadowOcclusionQuery(View, ProjectedShadowInfo, NumBufferedFrames, SOQ_NearPlaneVsShadowFrustum, ShadowOcclusionQuery))
						{
							ViewQuery.ShadowQuerieInfos.Add(&ProjectedShadowInfo);
							ViewQuery.ShadowQueries.Add(ShadowOcclusionQuery);
							checkSlow(ViewQuery.ShadowQuerieInfos.Num() == ViewQuery.ShadowQueries.Num());
							bBatchedQueries = true;
						}
					}
				}
			}

			if (FeatureLevel > ERHIFeatureLevel::ES3_1 &&
				!View.bIsPlanarReflection &&
				!View.bIsSceneCapture &&
				!View.bIsReflectionCapture)
			{
				// +1 to buffered frames because the query is submitted late into the main frame, but read at the beginning of a frame
				const int32 NumReflectionBufferedFrames = NumBufferedFrames + 1;

				for (int32 ReflectionIndex = 0; ReflectionIndex < Scene->PlanarReflections.Num(); ReflectionIndex++)
				{
					FPlanarReflectionSceneProxy* SceneProxy = Scene->PlanarReflections[ReflectionIndex];
					FRHIRenderQuery* ShadowOcclusionQuery;
					if (AllocatePlanarReflectionOcclusionQuery(View, SceneProxy, NumReflectionBufferedFrames, ShadowOcclusionQuery))
					{
						ViewQuery.ReflectionQuerieInfos.Add(SceneProxy);
						ViewQuery.ReflectionQueries.Add(ShadowOcclusionQuery);
						checkSlow(ViewQuery.ReflectionQuerieInfos.Num() == ViewQuery.ReflectionQueries.Num());
						bBatchedQueries = true;
					}
				}
			}

			// Don't do primitive occlusion if we have a view parent or are frozen - only applicable to Debug & Development.
#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
			ViewQuery.bFlushQueries &= (!ViewState->HasViewParent() && !ViewState->bIsFrozen);
#endif

			bBatchedQueries |= (View.IndividualOcclusionQueries.HasBatches() || View.GroupedOcclusionQueries.HasBatches() || ViewQuery.bFlushQueries);
		}
	}

	// Return an empty array if no queries exist.
	if (!bBatchedQueries)
	{
		QueriesPerView.Empty();
	}
	return MoveTemp(QueriesPerView);
}

static void BeginOcclusionTests(
	FRHICommandListImmediate& RHICmdList,
	TArrayView<FViewInfo> Views,
	ERHIFeatureLevel::Type FeatureLevel,
	const FViewOcclusionQueriesPerView& QueriesPerView,
	uint32 DownsampleFactor)
{
	check(RHICmdList.IsInsideRenderPass());
	check(QueriesPerView.Num() == Views.Num());

	SCOPE_CYCLE_COUNTER(STAT_BeginOcclusionTestsTime);
	SCOPED_DRAW_EVENT(RHICmdList, BeginOcclusionTests);

	FGraphicsPipelineStateInitializer GraphicsPSOInit;
	RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);
	GraphicsPSOInit.PrimitiveType = PT_TriangleList;
	GraphicsPSOInit.BlendState = TStaticBlendStateWriteMask<CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE, CW_NONE>::GetRHI();
	// Depth tests, no depth writes, no color writes, opaque
	GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_DepthNearOrEqual>::GetRHI();
	GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GetVertexDeclarationFVector3();

	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		SCOPED_DRAW_EVENTF(RHICmdList, ViewOcclusionTests, TEXT("ViewOcclusionTests %d"), ViewIndex);

		FViewInfo& View = Views[ViewIndex];
		const FViewOcclusionQueries& ViewQuery = QueriesPerView[ViewIndex];
		FSceneViewState* ViewState = (FSceneViewState*)View.State;
		SCOPED_GPU_MASK(RHICmdList, View.GPUMask);

		// ⭐⭐⭐⭐⭐⭐⭐
		// Occlusion Query를 수행하는데 필요한 PSO를 셋팅한다.
		//
		
		// ⭐
		// 당연히 Front Face만 렌더링하면 된다.
		// We only need to render the front-faces of the culling geometry (this halves the amount of pixels we touch)
		GraphicsPSOInit.RasterizerState = View.bReverseCulling ? TStaticRasterizerState<FM_Solid, CM_CCW>::GetRHI() : TStaticRasterizerState<FM_Solid, CM_CW>::GetRHI();
		// ⭐


		const FIntRect ViewRect = GetDownscaledRect(View.ViewRect, DownsampleFactor);
		RHICmdList.SetViewport(ViewRect.Min.X, ViewRect.Min.Y, 0.0f, ViewRect.Max.X, ViewRect.Max.Y, 1.0f);

		// Lookup the vertex shader.
		TShaderMapRef<FOcclusionQueryVS> VertexShader(View.ShaderMap);
		GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();

		if (View.Family->EngineShowFlags.OcclusionMeshes)
		{
			TShaderMapRef<FOcclusionQueryPS> PixelShader(View.ShaderMap);
			GraphicsPSOInit.BoundShaderState.PixelShaderRHI = PixelShader.GetPixelShader();
			GraphicsPSOInit.BlendState = TStaticBlendState<CW_RGBA>::GetRHI();
		}

		SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit);
		// ⭐⭐⭐⭐⭐⭐⭐

		// ⭐
		// 각종 Shadow, Reflection들에 대한 Occlusion Query 수행 커맨드를 RHI 스레드로 전송한다.
		// 쉐도우 맵도 결국에는 월드를 렌더링하는 것이다. ( 다른 것은 Depth 값만 취한다는 것 )
		// 그러니 쉐도우 맵을 그릴 필요가 있는지에 대해서도 Occlusion Query를 수행하여 확인하다.
		if (FeatureLevel > ERHIFeatureLevel::ES3_1)
		{
			SCOPED_DRAW_EVENT(RHICmdList, ShadowFrustumQueries);
			for (int i = 0; i < ViewQuery.PointLightQueries.Num(); i++)
			{
				ExecutePointLightShadowOcclusionQuery(RHICmdList, View, *ViewQuery.PointLightQueryInfos[i], VertexShader, ViewQuery.PointLightQueries[i]);
			}
		}

		uint32 NumVertices = ViewQuery.CSMQueries.Num() * 6 // Plane 
			+ ViewQuery.ShadowQueries.Num() * 8 // Cube
			+ ViewQuery.ReflectionQueries.Num() * 8; // Cube

		if (NumVertices > 0)
		{
			uint32 BaseVertexOffset = 0;
			FRHIResourceCreateInfo CreateInfo;
			FVertexBufferRHIRef VertexBufferRHI = RHICreateVertexBuffer(sizeof(FVector) * NumVertices, BUF_Volatile, CreateInfo);
			void* VoidPtr = RHILockVertexBuffer(VertexBufferRHI, 0, sizeof(FVector) * NumVertices, RLM_WriteOnly);

			{
				FVector* Vertices = reinterpret_cast<FVector*>(VoidPtr);
				for (FProjectedShadowInfo const* Query : ViewQuery.CSMQueryInfos)
				{
					PrepareDirectionalLightShadowOcclusionQuery(BaseVertexOffset, Vertices, View, *Query);
					checkSlow(BaseVertexOffset <= NumVertices);
				}

				for (FProjectedShadowInfo const* Query : ViewQuery.ShadowQuerieInfos)
				{
					PrepareProjectedShadowOcclusionQuery(BaseVertexOffset, Vertices, View, *Query);
					checkSlow(BaseVertexOffset <= NumVertices);
				}

				for (FPlanarReflectionSceneProxy const* Query : ViewQuery.ReflectionQuerieInfos)
				{
					PreparePlanarReflectionOcclusionQuery(BaseVertexOffset, Vertices, View, Query);
					checkSlow(BaseVertexOffset <= NumVertices);
				}
			}

			RHIUnlockVertexBuffer(VertexBufferRHI);

			{
				SCOPED_DRAW_EVENT(RHICmdList, ShadowFrustumQueries);
				VertexShader->SetParameters(RHICmdList, View);
				RHICmdList.SetStreamSource(0, VertexBufferRHI, 0);
				BaseVertexOffset = 0;

				for (FRHIRenderQuery* const& Query : ViewQuery.CSMQueries)
				{
					ExecuteDirectionalLightShadowOcclusionQuery(RHICmdList, BaseVertexOffset, Query);
					checkSlow(BaseVertexOffset <= NumVertices);
				}

				for (FRHIRenderQuery* const& Query : ViewQuery.ShadowQueries)
				{
					ExecuteProjectedShadowOcclusionQuery(RHICmdList, BaseVertexOffset, Query);
					checkSlow(BaseVertexOffset <= NumVertices);
				}
			}

			if (FeatureLevel > ERHIFeatureLevel::ES3_1)
			{
				SCOPED_DRAW_EVENT(RHICmdList, PlanarReflectionQueries);
				for (FRHIRenderQuery* const& Query : ViewQuery.ReflectionQueries)
				{
					ExecutePlanarReflectionOcclusionQuery(RHICmdList, BaseVertexOffset, Query);
					check(BaseVertexOffset <= NumVertices);
				}
			}

			VertexBufferRHI.SafeRelease();
		}
		// ⭐

		// 미리 저장해둔 Occlusion Query들에 대한 Query 수행 커맨드를 RHI 스레드로 전송한다.
		//
		// GroupedOcclusionQueries와 IndividualOcclusionQueries는 이전 InitViews 단계에서 저장을 해두었었다.      
		// 참고 : https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/04/16/FMobileSceneRenderer_2.html
		if (ViewQuery.bFlushQueries)
		{
			VertexShader->SetParameters(RHICmdList, View);

			{
				SCOPED_DRAW_EVENT(RHICmdList, GroupedQueries);
				View.GroupedOcclusionQueries.Flush(RHICmdList);
			}
			{
				SCOPED_DRAW_EVENT(RHICmdList, IndividualQueries);
				View.IndividualOcclusionQueries.Flush(RHICmdList);
			}
		}
	}
}
```

```cpp
void FMobileSceneRenderer::RenderTranslucency(FRHICommandListImmediate& RHICmdList, const TArrayView<const FViewInfo*> PassViews)
{
	ETranslucencyPass::Type TranslucencyPass = 
		ViewFamily.AllowTranslucencyAfterDOF() ? ETranslucencyPass::TPT_StandardTranslucency : ETranslucencyPass::TPT_AllTranslucency;
		
	if (ShouldRenderTranslucency(TranslucencyPass))
	{
		SCOPED_DRAW_EVENT(RHICmdList, Translucency);
		SCOPED_GPU_STAT(RHICmdList, Translucency);

		for (int32 ViewIndex = 0; ViewIndex < PassViews.Num(); ViewIndex++)
		{
			SCOPED_CONDITIONAL_DRAW_EVENTF(RHICmdList, EventView, Views.Num() > 1, TEXT("View%d"), ViewIndex);

			const FViewInfo& View = *PassViews[ViewIndex];
			if (!View.ShouldRenderView())
			{
				continue;
			}

			RHICmdList.SetViewport(View.ViewRect.Min.X, View.ViewRect.Min.Y, 0.0f, View.ViewRect.Max.X, View.ViewRect.Max.Y, 1.0f);

			if (!View.Family->UseDebugViewPS())
			{
				// ⭐
				// 투명한 Primitive를 렌더링하기 위해 관련된 UniformBuffer를 업데이트한다.
				if (Scene->UniformBuffers.UpdateViewUniformBuffer(View))
				{
					UpdateTranslucentBasePassUniformBuffer(RHICmdList, View);
					UpdateDirectionalLightUniformBuffers(RHICmdList, View);
				}
				// ⭐
		
				// ⭐
				// 적절한 Translucency Pass를 계산한다.
				const EMeshPass::Type MeshPass = TranslucencyPassToMeshPass(TranslucencyPass);
				//

				// ⭐⭐⭐⭐⭐⭐⭐
				// Translucency Pass의 MeshDrawCommand들에 대한 Draw Command를 GPU로 날린다.
				View.ParallelMeshDrawCommandPasses[MeshPass].DispatchDraw(nullptr, RHICmdList);
				// ⭐⭐⭐⭐⭐⭐⭐
			}
		}
	}
}

void FMobileSceneRenderer::UpdateTranslucentBasePassUniformBuffer(FRHICommandListImmediate& RHICmdList, const FViewInfo& View)
{
	FMobileBasePassUniformParameters Parameters;

	// ⭐
	// 위에서 분석한 부분을 다시 읽어보자.
	SetupMobileBasePassUniformParameters(RHICmdList, View, true, false, Parameters);
	// ⭐

	Scene->UniformBuffers.MobileTranslucentBasePassUniformBuffer.UpdateUniformBufferImmediate(Parameters);
}

EMeshPass::Type TranslucencyPassToMeshPass(ETranslucencyPass::Type TranslucencyPass)
{
	EMeshPass::Type TranslucencyMeshPass = EMeshPass::Num;

	switch (TranslucencyPass)
	{
	case ETranslucencyPass::TPT_StandardTranslucency: TranslucencyMeshPass = EMeshPass::TranslucencyStandard; break;
	case ETranslucencyPass::TPT_TranslucencyAfterDOF: TranslucencyMeshPass = EMeshPass::TranslucencyAfterDOF; break;
	case ETranslucencyPass::TPT_TranslucencyAfterDOFModulate: TranslucencyMeshPass = EMeshPass::TranslucencyAfterDOFModulate; break;
	case ETranslucencyPass::TPT_AllTranslucency: TranslucencyMeshPass = EMeshPass::TranslucencyAll; break;
	}

	check(TranslucencyMeshPass != EMeshPass::Num);

	return TranslucencyMeshPass;
}
```
