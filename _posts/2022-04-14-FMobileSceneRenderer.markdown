---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 1"
date:   2022-04-14
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
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

- 1. [FScene::UpdateAllPrimitiveSceneInfos](https://sungjjinkang.github.io/unrealengine4/ue4/computerscience/computergraphics/2022/04/16/FMobileSceneRenderer_1.html)               


```cpp
void FMobileSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
{
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_SceneStart));

	SCOPED_DRAW_EVENT(RHICmdList, MobileSceneRender);
	SCOPED_GPU_STAT(RHICmdList, MobileSceneRender);

    // ⭐⭐⭐⭐⭐⭐⭐
	// 1.
	Scene->UpdateAllPrimitiveSceneInfos(RHICmdList);
	// ⭐⭐⭐⭐⭐⭐⭐

	PrepareViewRectsForRendering(RHICmdList);

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

	WaitOcclusionTests(RHICmdList);
	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	RHICmdList.ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);

	// Find the visible primitives and prepare targets and buffers for rendering
	InitViews(RHICmdList);
	
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
	if (bDeferredShading)
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

	// Custom depth
	// bShouldRenderCustomDepth has been initialized in InitViews on mobile platform
	if (bShouldRenderCustomDepth)
	{
		FRDGBuilder GraphBuilder(RHICmdList);
		FSceneTextureShaderParameters SceneTextures = CreateSceneTextureShaderParameters(GraphBuilder, Views[0].GetFeatureLevel(), ESceneTextureSetupMode::None);
		RenderCustomDepthPass(GraphBuilder, SceneTextures);
		GraphBuilder.Execute();
	}

	if (bIsFullPrepassEnabled)
	{
		//SDF and AO require full depth prepass

		FRHIRenderPassInfo DepthPrePassRenderPassInfo(
			SceneContext.GetSceneDepthSurface(),
			EDepthStencilTargetActions::ClearDepthStencil_StoreDepthStencil);

		DepthPrePassRenderPassInfo.NumOcclusionQueries = ComputeNumOcclusionQueriesToBatch();
		DepthPrePassRenderPassInfo.bOcclusionQueries = DepthPrePassRenderPassInfo.NumOcclusionQueries != 0;

		RHICmdList.BeginRenderPass(DepthPrePassRenderPassInfo, TEXT("DepthPrepass"));

		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLM_MobilePrePass));
		// Full Depth pre-pass
		RenderPrePass(RHICmdList);

		// Issue occlusion queries
		RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_Occlusion));
		RenderOcclusion(RHICmdList);

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
	if (bDeferredShading)
	{
		SceneColor = RenderDeferred(RHICmdList, ViewList, SortedLightSet);
	}
	else
	{
		SceneColor = RenderForward(RHICmdList, ViewList);
	}
	

	if (bShouldRenderVelocities)
	{
		FRDGBuilder GraphBuilder(RHICmdList);

		FRDGTextureMSAA SceneDepthTexture = RegisterExternalTextureMSAA(GraphBuilder, SceneContext.SceneDepthZ);
		FRDGTextureRef VelocityTexture = TryRegisterExternalTexture(GraphBuilder, SceneContext.SceneVelocity);

		if (VelocityTexture != nullptr)
		{
			AddClearRenderTargetPass(GraphBuilder, VelocityTexture);
		}

		// Render the velocities of movable objects
		AddSetCurrentStatPass(GraphBuilder, GET_STATID(STAT_CLMM_Velocity));
		RenderVelocities(GraphBuilder, SceneDepthTexture.Resolve, VelocityTexture, FSceneTextureShaderParameters(), EVelocityPass::Opaque, false);
		AddSetCurrentStatPass(GraphBuilder, GET_STATID(STAT_CLMM_AfterVelocity));

		AddSetCurrentStatPass(GraphBuilder, GET_STATID(STAT_CLMM_TranslucentVelocity));
		RenderVelocities(GraphBuilder, SceneDepthTexture.Resolve, VelocityTexture, GetSceneTextureShaderParameters(CreateMobileSceneTextureUniformBuffer(GraphBuilder, EMobileSceneTextureSetupMode::SceneColor)), EVelocityPass::Translucent, false);

		GraphBuilder.Execute();
	}

	{
		FRendererModule& RendererModule = static_cast<FRendererModule&>(GetRendererModule());
		FRDGBuilder GraphBuilder(RHICmdList);
		RendererModule.RenderPostOpaqueExtensions(GraphBuilder, Views, SceneContext);

		if (FXSystem && Views.IsValidIndex(0))
		{
			AddUntrackedAccessPass(GraphBuilder, [this](FRHICommandListImmediate& RHICmdList)
			{
				check(RHICmdList.IsOutsideRenderPass());

				FXSystem->PostRenderOpaque(
					RHICmdList,
					Views[0].ViewUniformBuffer,
					nullptr,
					nullptr,
					Views[0].AllowGPUParticleUpdate()
				);
				if (FGPUSortManager* GPUSortManager = FXSystem->GetGPUSortManager())
				{
					GPUSortManager->OnPostRenderOpaque(RHICmdList);
				}
			});
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
	
	if (ViewFamily.bResolveScene)
	{
		if (!bGammaSpace || bRenderToSceneColor)
		{
			// Finish rendering for each view, or the full stereo buffer if enabled
			{
				RDG_EVENT_SCOPE(GraphBuilder, "PostProcessing");
				SCOPE_CYCLE_COUNTER(STAT_FinishRenderViewTargetTime);

				// Note that we should move this uniform buffer set up process right after the InitView to avoid any uniform buffer creation during the rendering after we porting all the passes to the RDG.
				// We couldn't do it right now because the ResolveSceneDepth has another GraphicBuilder and it will re-register SceneDepthZ and that will cause crash.
				TArray<TRDGUniformBufferRef<FMobileSceneTextureUniformParameters>, TInlineAllocator<1, SceneRenderingAllocator>> MobileSceneTexturesPerView;
				MobileSceneTexturesPerView.SetNumZeroed(Views.Num());

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

				SetupMobileSceneTexturesPerView();

				FMobilePostProcessingInputs PostProcessingInputs;
				PostProcessingInputs.ViewFamilyTexture = ViewFamilyTexture;

				for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
				{
					RDG_EVENT_SCOPE_CONDITIONAL(GraphBuilder, Views.Num() > 1, "View%d", ViewIndex);
					PostProcessingInputs.SceneTextures = MobileSceneTexturesPerView[ViewIndex];
					AddMobilePostProcessingPasses(GraphBuilder, Views[ViewIndex], PostProcessingInputs, NumMSAASamples > 1);
				}
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

	RenderFinish(GraphBuilder, ViewFamilyTexture);
	GraphBuilder.Execute();

	FRHICommandListExecutor::GetImmediateCommandList().PollOcclusionQueries();
	FRHICommandListExecutor::GetImmediateCommandList().ImmediateFlush(EImmediateFlushType::DispatchToRHIThread);
}
```



------------------------------         

references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
