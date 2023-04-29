---
layout: post
title:  "언리얼 엔진4 FMobileSceneRenderer 분석 - 0 ( FMobileSceneRenderer::Render )"
date:   2022-04-14
tags: [UE, Recommend]
---

**FMobileSceneRenderer**는 언리얼 엔진4의 모바일용 그래픽스 파이프라인이다.                
이 글은 **FMobileSceneRenderer::Render** 함수를 위주로 FMobileSceneRenderer의 소스 코드를 분석할 것이다.      
          
UE4 4.27 버전을 기준으로 작성하였다.      
4.27 버전이 언리얼 엔진 4세대의 마지막 Release 버전이다.             
          
------------------------------        

**FMobileSceneRenderer::Render** 함수는 렌더 스레드에서 매틱마다 도는 함수로 렌더링에 필요한 여러 동작을 수행한다.         
FMobileSceneRenderer는 모바일에서만 사용되는 렌더러로 ( 아마 안드로이드와 일부 IOS기기에서 사용되는 것으로 보임 ), 기존의 다른 플랫폼에서 사용되는 렌더러와는 달리 모바일 플랫폼의 특수성으로 인해 Forward 렌더링으로 렌더링을 수행한다. 이는 모바일 환경에서 Deffered 렌더링 수행에 필요한 G버퍼의를 가지고 있기에는 모바일 GPU 메모리의 제약이 많기 때문이다.      

FMobileSceneRenderer::Render 함수의 호출 경로를 보고 싶다면        
UGameViewportClient::Draw -> FRendererModule::BeginRenderingViewFamily -> RenderViewFamily_RenderThread -> FMobileSceneRenderer::Render 순으로 따라 가면 된다.     

----------------------------------------------

주요 코드들 위주로 챕터를 나누어서 소개할 것이다.         

1. [FScene::UpdateAllPrimitiveSceneInfos](https://sungjjinkang.github.io/FMobileSceneRenderer_1)            
            
2. [FMobileSceneRenderer::InitViews](https://sungjjinkang.github.io/FMobileSceneRenderer_2)          
          
3. [FSceneRenderer::RenderCustomDepthPass](https://sungjjinkang.github.io/FMobileSceneRenderer_3)         
        
4. [FSceneRenderer::RenderForward](https://sungjjinkang.github.io/FMobileSceneRenderer_4)             




```cpp
void FMobileSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
{
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_SceneStart));

	SCOPED_DRAW_EVENT(RHICmdList, MobileSceneRender);
	SCOPED_GPU_STAT(RHICmdList, MobileSceneRender);

    // ⭐⭐⭐⭐⭐⭐⭐
	// 1. PrimitiveSceneInfo를 업데이트해주고 필요한 FMeshBatch를 생성해서 캐싱해둔다.
	//    추가되거나 ( 갱싱된 ) StaticMesh에 대한 FMeshBatch, FMeshDrawCommand를 생성하고 캐싱한다.
	Scene->UpdateAllPrimitiveSceneInfos(RHICmdList);
	// ⭐⭐⭐⭐⭐⭐⭐

	// ⭐
	// FViewInfo를 설정한다.
	// 해상도, 렌더링 수행할 영역... 등등을 셋팅한다.
	PrepareViewRectsForRendering(RHICmdList);
	// ⭐

	if (ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags))
	{
		for (int32 LightIndex = 0; LightIndex < NUM_ATMOSPHERE_LIGHTS; ++LightIndex)
		{
			if (Scene->AtmosphereLights[LightIndex])
			{
				PrepareSunLightProxy(*Scene->GetSkyAtmosphereSceneInfo(), LightIndex, *Scene->AtmosphereLights[LightIndex]);
			}
		}
	}
	else
	{
		Scene->ResetAtmosphereLightsProperties();
	}

	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderOther);
	QUICK_SCOPE_CYCLE_COUNTER(STAT_FMobileSceneRenderer_Render);
	//FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

	if(!ViewFamily.EngineShowFlags.Rendering)
	{
		return;
	}

	SCOPED_DRAW_EVENT(RHICmdList, MobileSceneRender);
	SCOPED_GPU_STAT(RHICmdList, MobileSceneRender);

	// ⭐
	// RHI 스레드가 렌더 스레드와 분리되어 있는 경우 이전 프레임의 Occlusion Query 결과를 기다린다.
	// 여기서 Occlusion Query란 : 
	// Occlude 될(!) 오브젝트들을 픽셀 쉐이딩 없이 ( 그냥 Depth 값만 얻으면 되니 ) 그려본 후 ( Occluder와 Depth 값만 비교하는 용도로 ),
	// 그려졌다면 ( Depth Test를 통과했다면 ) 해당 오브젝트는 Occlude되지 않은 오브젝트이니 제대로 그린다.        
	// GPU를 사용하여 수행하는 Occlusion Culling의 일종이다.
	// Occlusion Query 결과 ( Occlude될 오브젝트가 Depth Test를 통과했는지 여부 )를 시스템 메모리 ( 메인 메모리 )로 다시 읽어와야하기 때문에
	// 성능 문제 ( reference : https://megayuchi.com/2021/06/06/ddraw-surface-d3d-dynamic-buffer-%EC%97%90%EC%84%9C%EC%9D%98-write-combine-memory/ )가 있다.
	//  
	WaitOcclusionTests(RHICmdList);
	// ⭐

	
	// ⭐
	// 이전 프레임 ( 3 ~ 4 프레임 전 )의 Occlusion Query의 결과가 나왔는지 확인한다.
	// GPU에서 아직 Occlusion Query를 처리하지 못했다면 그냥 빠져나온다. ( Stall되지 않는다. )
	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	// ⭐

	// ⭐⭐⭐⭐⭐⭐⭐
	// 렌더스레드에서 RHI 스레드로 전송 대기 중인 커맨드들을 즉시 처리한다.
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	// ⭐⭐⭐⭐⭐⭐⭐


    // ⭐⭐⭐⭐⭐⭐⭐
	// 2. Primitive들의 가시성 연산,  반투명 오브젝트들에 대한 Sorting 작업등을 수행한다. ( 대부분 CPU에서 처리 )
	//    또한 View에 그려질 Static Mesh, Dynamic Mesh에 대한 FMeshBatch, FMeshDrawCommand를 생성하고 모읍니다. ( 매우 중요!! )
	//    그리고 각종 Pass들에 필요한 사전 작업들을 수행한다.
	//
	// Find the visible primitives and prepare targets and buffers for rendering
	InitViews(RHICmdList);
    // ⭐⭐⭐⭐⭐⭐⭐
	
	if (GRHINeedsExtraDeletionLatency || !GRHICommandList.Bypass())
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_FMobileSceneRenderer_PostInitViewsFlushDel);
		// we will probably stall on occlusion queries, so might as well have the RHI thread and GPU work while we wait.
		// Also when doing RHI thread this is the only spot that will process pending deletes
		FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
		FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(EImmediateFlushType::FlushRHIThreadFlushResources);
	}

	GEngine->GetPreRenderDelegate().Broadcast();

	// Global dynamic buffers need to be committed before rendering.
	DynamicIndexBuffer.Commit();
	DynamicVertexBuffer.Commit();
	DynamicReadBuffer.Commit();
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_SceneSim));

	if (ViewFamily.bLateLatchingEnabled)
	{
		BeginLateLatching(RHICmdList);
	}

	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);

	if (bUseVirtualTexturing)
	{
		SCOPED_GPU_STAT(RHICmdList, VirtualTextureUpdate);
		FVirtualTextureSystem::Get().Update(RHICmdList, FeatureLevel, Scene);
		// Clear virtual texture feedback to default value
		FUnorderedAccessViewRHIRef FeedbackUAV = SceneContext.GetVirtualTextureFeedbackUAV();
		RHICmdList.Transition(FRHITransitionInfo(FeedbackUAV, ERHIAccess::SRVMask, ERHIAccess::UAVMask));
		RHICmdList.ClearUAVUint(FeedbackUAV, FUintVector4(~0u, ~0u, ~0u, ~0u));
		RHICmdList.Transition(FRHITransitionInfo(FeedbackUAV, ERHIAccess::UAVMask, ERHIAccess::UAVMask));
		RHICmdList.BeginUAVOverlap(FeedbackUAV);
	}
	
	FSortedLightSetSceneInfo SortedLightSet;

	// ⭐
	// 모바일의 경우 G버퍼도 저장할만큼 메모리가 충분치 않아,
	// 기본적으로 Forward 렌더링으로 렌더링한다.
	if (bDeferredShading)
	// ⭐
	{
		GatherAndSortLights(SortedLightSet);
		int32 NumReflectionCaptures = Views[0].NumBoxReflectionCaptures + Views[0].NumSphereReflectionCaptures;
		bool bCullLightsToGrid = (NumReflectionCaptures > 0 || GMobileUseClusteredDeferredShading != 0);
		FRDGBuilder GraphBuilder(RHICmdList);
		ComputeLightGrid(GraphBuilder, bCullLightsToGrid, SortedLightSet);
		GraphBuilder.Execute();
	}

	// Generate the Sky/Atmosphere look up tables
	const bool bShouldRenderSkyAtmosphere = ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags);
	if (bShouldRenderSkyAtmosphere)
	{
		FRDGBuilder GraphBuilder(RHICmdList);
		RenderSkyAtmosphereLookUpTables(GraphBuilder);
		GraphBuilder.Execute();
	}

	// Notify the FX system that the scene is about to be rendered.
	if (FXSystem && ViewFamily.EngineShowFlags.Particles)
	{
		FXSystem->PreRender(RHICmdList, NULL, !Views[0].bIsPlanarReflection);
		if (FGPUSortManager* GPUSortManager = FXSystem->GetGPUSortManager())
		{
			GPUSortManager->OnPreRender(RHICmdList);
		}
	}
	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Shadows));

	RenderShadowDepthMaps(RHICmdList);
	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

	// Default view list
	TArray<const FViewInfo*> ViewList;
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++) 
	{
		ViewList.Add(&Views[ViewIndex]);
	}

    // ⭐⭐⭐⭐⭐⭐⭐
	// 3. Custom Depth Pass
	// bShouldRenderCustomDepth has been initialized in InitViews on mobile platform
	//
	// Custom Depth 옵션이 활성화되어 있는 경우 Custom Depth Pass를 렌더링한다.
	if (bShouldRenderCustomDepth)
	{
		FRDGBuilder GraphBuilder(RHICmdList);
		FSceneTextureShaderParameters SceneTextures = CreateSceneTextureShaderParameters(GraphBuilder, Views[0].GetFeatureLevel(), ESceneTextureSetupMode::None);
		RenderCustomDepthPass(GraphBuilder, SceneTextures);
		GraphBuilder.Execute();
	}
    // ⭐⭐⭐⭐⭐⭐⭐

	// ⭐
	// Depth Buffer를 그릴지에 대한 옵션 enum
	// enum EDepthDrawingMode
	// { 
	//   // tested at a higher level
	// 	 DDM_None			= 0,
	//
	// 	 // Opaque 매터리얼의 경우에만
	// 	 DDM_NonMaskedOnly	= 1,
	//
	// 	 // Opaque, Masked 매터리얼의 경우, 단 bUseAsOccluder 옵션이 disabled되어 있는 오브젝트는 제외하고
	// 	 DDM_AllOccluders	= 2,
	//
	// 	 // Full prepass, 모든 오브젝트의 Depth가 그려져야하고, 모든 픽셀은 Base Pass의 Depth와 일치해야한다.
	// 	 DDM_AllOpaque		= 3,
	//
	// 	 // Masked materials만
	// 	 DDM_MaskedOnly = 4,
	// };
	//
	//
	// void FScene::UpdateEarlyZPassMode()
	// {
	//   DefaultBasePassDepthStencilAccess = FExclusiveDepthStencil::DepthWrite_StencilWrite;
    //	 EarlyZPassMode = DDM_NonMaskedOnly;
	//	 bEarlyZPassMovable = false;
	//
	//   if (GetShadingPath(GetFeatureLevel()) == EShadingPath::Deferred)
	//   {
	//   	...
	//      ... 모바일의 경우 G버퍼도 저장할만큼 메모리가 충분치 않아, 기본적으로 Forward 렌더링으로 렌더링한다.
	//		...
	//	 }
	//   else if (GetShadingPath(GetFeatureLevel()) == EShadingPath::Mobile)
	//	 {
	//		 EarlyZPassMode = DDM_None;
	//
	//		 if (MaskedInEarlyPass(GetFeatureLevelShaderPlatform(FeatureLevel)))
	//		 {
	//			 EarlyZPassMode = DDM_MaskedOnly;
	//		 }
	//
	//		 if (IsMobileDistanceFieldEnabled(GetShaderPlatform()) || IsMobileAmbientOcclusionEnabled(GetShaderPlatform()))
	//		 {
	//			 EarlyZPassMode = DDM_AllOpaque;
	//		 }
	//	 }
	// }
	//
	//
	// 모든 오브젝트에 대해 Depth Prepass를 수행하는 경우.
	// Depth Only Pass ( 픽셀 쉐이더 단계에서 점만 찍는다, 그냥 Depth 값만 취한다. )의 경우,
	// Base Pass에서 Primitive를 렌더링 할 때 Early Z를 극대화하여 오버드로우를 최소화, 픽셀 쉐이딩 연산을 최소화하기 위함이다.
	// ( 값 비싼 픽셀 쉐이딩을 최소화하겠다! )
	// 결과적으로 한 Fragment에 한번만 Pixel 쉐이딩을 수행하고자 ( 오버드로우를 최소화 ) 하는 목적으로 사용한다.
	// ( 디퍼드 렌더링의 경우 어차피 Pixel 쉐이더가 가벼운데 왜 Depth Only Pass를 수행하느냐는 논쟁도 있더라..... )
	//
	// 모바일 Forward 렌더링의 경우에는 Sigend Distance Field나 Ambient Occlusion이 Enabled되어 있는 경우에만 Full Depth Pre Pass를 수행한다.
	//
	// 반면 Deffered 렌더링의 경우 EarlyZPass 프로젝트 설정에 따라 달라진다.
	//
	//
	// 모바일 GPU의 경우에는 Depth Pre Pass가 불필요 하다고 한다.
	// 오히려 성능에 해가된다고....
	// reference : 
	// https://sungjjinkang.github.io/mobile_gpu_hidden_surface_removal
	//
	// EarlyZPassMode가 DDM_AllOpaque인 경우. 
	if (bIsFullPrepassEnabled)
	// ⭐
	{
		//SDF and AO require full depth prepass

		FRHIRenderPassInfo DepthPrePassRenderPassInfo(
			SceneContext.GetSceneDepthSurface(),
			EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil);

		DepthPrePassRenderPassInfo.NumOcclusionQueries = ComputeNumOcclusionQueriesToBatch();
		DepthPrePassRenderPassInfo.bOcclusionQueries = DepthPrePassRenderPassInfo.NumOcclusionQueries != 0;

		RHICmdList.BeginRenderPass(DepthPrePassRenderPassInfo, TEXT("DepthPrepass"));

		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLM_MobilePrePass));

		// ⭐
		// Full Depth pre-pass
		//
		// Depth Pre-Pass를 수행한다.
		// 월드의 Opaque한 오브젝트들에 대해 픽셀 쉐이딩이 생략된(간소화된) Draw를 수행한다. ( 그냥 Depth 버퍼만 채우는 것이다. )
		RenderPrePass(RHICmdList);
		// ⭐

		// ⭐
		// Issue occlusion queries
		//
		// 바로 위 RenderPrePass에서 채운 Depth Buffer를 가지고 Occlusion Query를 수행한다.
		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Occlusion));
		RenderOcclusion(RHICmdList);
		// ⭐

		RHICmdList.EndRenderPass();

		if (bRequiresDistanceFieldShadowingPass)
		{
			CSV_SCOPED_TIMING_STAT_EXCLUSIVE(RenderSDFShadowing);
			RenderSDFShadowing(RHICmdList);
		}

		if (bShouldRenderHZB)
		{
			RenderHZB(RHICmdList, SceneContext.SceneDepthZ);
		}

		if (bRequiresAmbientOcclusionPass)
		{
			RenderAmbientOcclusion(RHICmdList, SceneContext.SceneDepthZ);
		}
	}

	FRHITexture* SceneColor = nullptr;
	// ⭐
	// 모바일의 경우 G버퍼도 저장할만큼 메모리가 충분치 않아,
	// 기본적으로 Forward 렌더링으로 렌더링한다.
	if (bDeferredShading)
	// ⭐
	{
		SceneColor = RenderDeferred(RHICmdList, ViewList, SortedLightSet);	
	}
	else
	{
		// ⭐⭐⭐⭐⭐⭐⭐
		// 4. 모바일의 경우 기본적으로 Forward 렌더링으로 수행한다.
		SceneColor = RenderForward(RHICmdList, ViewList);
		// ⭐⭐⭐⭐⭐⭐⭐
	}
	

	// ⭐
	// Temporal Anti Aliasing을 수행하는 경우에만 해당.
	// references : https://scahp.tistory.com/77
	if (bShouldRenderVelocities)
	// ⭐
	{
		// ...
		// ...
		// ...
	}

	{
		FRendererModule& RendererModule = static_cast<FRendererModule&>(GetRendererModule());
		FRDGBuilder GraphBuilder(RHICmdList);
		RendererModule.RenderPostOpaqueExtensions(GraphBuilder, Views, SceneContext);

		if (FXSystem && Views.IsValidIndex(0))
		{
			// ...
			// ... /** The cached FXSystem which could be released while we are rendering. */
			// ...
		}
		GraphBuilder.Execute();
	}

	// Flush / submit cmdbuffer
	if (bSubmitOffscreenRendering)
	{
		RHICmdList.SubmitCommandsHint();
		RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
	}
	
	if (!bGammaSpace || bRenderToSceneColor)
	{
		RHICmdList.Transition(FRHITransitionInfo(SceneColor, ERHIAccess::Unknown, ERHIAccess::SRVMask));
	}

	if (bDeferredShading)
	{
		// Release the original reference on the scene render targets
		SceneContext.AdjustGBufferRefCount(RHICmdList, -1);
	}

	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Post));

	if (bUseVirtualTexturing)
	{	
		SCOPED_GPU_STAT(RHICmdList, VirtualTextureUpdate);

		// No pass after this should make VT page requests
		RHICmdList.EndUAVOverlap(SceneContext.VirtualTextureFeedbackUAV);
		RHICmdList.Transition(FRHITransitionInfo(SceneContext.VirtualTextureFeedbackUAV, ERHIAccess::UAVMask, ERHIAccess::SRVMask));

		TArray<FIntRect, TInlineAllocator<4>> ViewRects;
		ViewRects.AddUninitialized(Views.Num());
		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
		{
			ViewRects[ViewIndex] = Views[ViewIndex].ViewRect;
		}
		
		FVirtualTextureFeedbackBufferDesc Desc;
		Desc.Init2D(SceneContext.GetBufferSizeXY(), ViewRects, SceneContext.GetVirtualTextureFeedbackScale());

		SubmitVirtualTextureFeedbackBuffer(RHICmdList, SceneContext.VirtualTextureFeedback, Desc);
	}

	FMemMark Mark(FMemStack::Get());
	FRDGBuilder GraphBuilder(RHICmdList);

	FRDGTextureRef ViewFamilyTexture = TryCreateViewFamilyTexture(GraphBuilder, ViewFamily);
	
	// ⭐
	// bRenderToSceneColor = 
	//		!bGammaSpace || 
	//		bStereoRenderingAndHMD || 
	//		bRequiresUpscale || 
	//		FSceneRenderer::ShouldCompositeEditorPrimitives(Views[0]) || 
	//		Views[0].bIsSceneCapture || 
	//		Views[0].bIsReflectionCapture;
	if (ViewFamily.bResolveScene)
	// ⭐
	{
		if (!bGammaSpace || bRenderToSceneColor)
		{
			// Finish rendering for each view, or the full stereo buffer if enabled
			{
				// ⭐⭐⭐⭐⭐⭐⭐ 
				// Post Process Pass

				RDG_EVENT_SCOPE(GraphBuilder, "PostProcessing");
				SCOPE_CYCLE_COUNTER(STAT_FinishRenderViewTargetTime);

				// Note that we should move this uniform buffer set up process right after the InitView to avoid any uniform buffer creation during the rendering after we porting all the passes to the RDG.
				// We couldn't do it right now because the ResolveSceneDepth has another GraphicBuilder and it will re-register SceneDepthZ and that will cause crash.
				TArray<TRDGUniformBufferRef<FMobileSceneTextureUniformParameters>, TInlineAllocator<1, SceneRenderingAllocator>> MobileSceneTexturesPerView;
				MobileSceneTexturesPerView.SetNumZeroed(Views.Num());

				// ⭐
				// Post Process에 사용할 Scene Texture를 지정할 Uniform Buffer를 생성한다.
				const auto SetupMobileSceneTexturesPerView = [&]()
				{
					for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
					{
						EMobileSceneTextureSetupMode SetupMode = EMobileSceneTextureSetupMode::SceneColor;
						if (Views[ViewIndex].bCustomDepthStencilValid)
						{
							SetupMode |= EMobileSceneTextureSetupMode::CustomDepth;
						}

						if (bShouldRenderVelocities)
						{
							SetupMode |= EMobileSceneTextureSetupMode::SceneVelocity;
						}

						MobileSceneTexturesPerView[ViewIndex] = CreateMobileSceneTextureUniformBuffer(GraphBuilder, SetupMode);
					}
				};
				// ⭐

				SetupMobileSceneTexturesPerView();

				FMobilePostProcessingInputs PostProcessingInputs;
				PostProcessingInputs.ViewFamilyTexture = ViewFamilyTexture;

				for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
				{
					RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);
					PostProcessingInputs.SceneTextures = MobileSceneTexturesPerView[ViewIndex];

					// ⭐
					// AddMobilePostProcessingPasses 함수를 확인해보면 수 많은 PostProcess 효과들을 만들어내는 동작을 수행한다.
					AddMobilePostProcessingPasses(GraphBuilder, Views[ViewIndex], PostProcessingInputs, NumMSAASamples > 1);
					// ⭐
				}
				
				// ⭐⭐⭐⭐⭐⭐⭐ 
			}
		}
	}

	GEngine->GetPostRenderDelegate().Broadcast();

	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_SceneEnd));

	if (bShouldRenderVelocities)
	{
		SceneContext.SceneVelocity.SafeRelease();
	}
	
	if (ViewFamily.bLateLatchingEnabled)
	{
		// LateLatching is only enabled with multiview
		EndLateLatching(RHICmdList, Views[0]);
	}

	// ⭐
	// 디버깅용 여러 동작들을 수행한다.
	RenderFinish(GraphBuilder, ViewFamilyTexture);
	// ⭐

	GraphBuilder.Execute();

	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
}
```



------------------------------         

