---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 3 ( FMobileSceneRenderer::InitViews )"
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
             
이번 챕터에는 **FMobileSceneRenderer::InitViews**에 대해 분석해볼 것이다.              
                    
------------------------------         

```cpp
/**
 * Initialize scene's views.
 * Check visibility, sort translucent items, etc.
 */
void FMobileSceneRenderer::InitViews(FRHICommandListImmediate& RHICmdList)
{
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_InitViews));

	SCOPED_DRAW_EVENT(RHICmdList, InitViews);

	SCOPE_CYCLE_COUNTER(STAT_InitViewsTime);
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(InitViews_Scene);

	check(Scene);

	// ⭐
	// 가상 텍스쳐링 수행을 위한 공간을 확보한다.
	if (bUseVirtualTexturing)
	{
		SCOPED_GPU_STAT(RHICmdList, VirtualTextureUpdate);
		// AllocateResources needs to be called before RHIBeginScene
		FVirtualTextureSystem::Get().AllocateResources(RHICmdList, FeatureLevel);
		FVirtualTextureSystem::Get().CallPendingCallbacks();
	}
	// ⭐

	FILCUpdatePrimTaskData ILCTaskData;
	FViewVisibleCommandsPerView ViewCommandsPerView;
	ViewCommandsPerView.SetNum(Views.Num());

	const FExclusiveDepthStencil::Type BasePassDepthStencilAccess = FExclusiveDepthStencil::DepthWrite_StencilWrite;

	// ⭐
	// 가시성 결정 전 한번 수행한다.
	// 가시성 처리 연산을 위한 각종 준비들을 수행한다.
	PreVisibilityFrameSetup(RHICmdList);
	// ⭐

	// ⭐⭐⭐⭐⭐⭐⭐
	// 가시성 결정 연산을 수행한다.
	// 아래로 내려가서 자세한 분석을 읽자.
	ComputeViewVisibility(RHICmdList, BasePassDepthStencilAccess, ViewCommandsPerView, DynamicIndexBuffer, DynamicVertexBuffer, DynamicReadBuffer);
	// ⭐⭐⭐⭐⭐⭐⭐

	PostVisibilityFrameSetup(ILCTaskData);

	const FIntPoint RenderTargetSize = (ViewFamily.RenderTarget->GetRenderTargetTexture().IsValid()) ? ViewFamily.RenderTarget->GetRenderTargetTexture()->GetSizeXY() : ViewFamily.RenderTarget->GetSizeXY();
	const bool bRequiresUpscale = 
		((int32)RenderTargetSize.X > FamilySize.X || (int32)RenderTargetSize.Y > FamilySize.Y) 
		// in the editor color surface and backbuffer could have a different pixel formats and size, 
		// so we always run upscale pass to blit content from scene color to backbuffer 
		|| (GIsEditor && !IsMobileHDR() && NumMSAASamples > 1);
	// ES requires that the back buffer and depth match dimensions.
	// For the most part this is not the case when using scene captures. Thus scene captures always render to scene color target.
	const bool bStereoRenderingAndHMD = ViewFamily.EngineShowFlags.StereoRendering && ViewFamily.EngineShowFlags.HMDDistortion;
	bRenderToSceneColor = !bGammaSpace || bStereoRenderingAndHMD || bRequiresUpscale || FSceneRenderer::ShouldCompositeEditorPrimitives(Views[0]) || Views[0].bIsSceneCapture || Views[0].bIsReflectionCapture;
	const FPlanarReflectionSceneProxy* PlanarReflectionSceneProxy = Scene ? Scene->GetForwardPassGlobalPlanarReflection() : nullptr;

	bRequiresPixelProjectedPlanarRelfectionPass = IsUsingMobilePixelProjectedReflection(ShaderPlatform)
		&& PlanarReflectionSceneProxy != nullptr
		&& PlanarReflectionSceneProxy->RenderTarget != nullptr
		&& !Views[0].bIsReflectionCapture
		&& !ViewFamily.EngineShowFlags.HitProxies
		&& ViewFamily.EngineShowFlags.Lighting
		&& !ViewFamily.EngineShowFlags.VisualizeLightCulling
		&& !ViewFamily.UseDebugViewPS()
		// Only support forward shading, we don't want to break tiled deferred shading.
		&& !bDeferredShading;

	bRequiresAmbientOcclusionPass = IsUsingMobileAmbientOcclusion(ShaderPlatform)
		&& Views[0].FinalPostProcessSettings.AmbientOcclusionIntensity > 0
		&& (Views[0].FinalPostProcessSettings.AmbientOcclusionStaticFraction >= 1 / 100.0f || (Scene && Scene->SkyLight && Scene->SkyLight->ProcessedTexture && Views[0].Family->EngineShowFlags.SkyLighting))
		&& ViewFamily.EngineShowFlags.Lighting
		&& !Views[0].bIsReflectionCapture
		&& !Views[0].bIsPlanarReflection
		&& !ViewFamily.EngineShowFlags.HitProxies
		&& !ViewFamily.EngineShowFlags.VisualizeLightCulling
		&& !ViewFamily.UseDebugViewPS();

	bRequiresDistanceField = IsMobileDistanceFieldEnabled(ShaderPlatform)
		&& ViewFamily.EngineShowFlags.Lighting
		&& !Views[0].bIsReflectionCapture
		&& !Views[0].bIsPlanarReflection
		&& !ViewFamily.EngineShowFlags.HitProxies
		&& !ViewFamily.EngineShowFlags.VisualizeLightCulling
		&& !ViewFamily.UseDebugViewPS();

	bRequiresDistanceFieldShadowingPass = bRequiresDistanceField && IsMobileDistanceFieldShadowingEnabled(ShaderPlatform);
		
	bShouldRenderVelocities = ShouldRenderVelocities();

	bShouldRenderHZB = ShouldRenderHZB();

	// Whether we need to store depth for post-processing
	// On PowerVR we see flickering of shadows and depths not updating correctly if targets are discarded.
	// See CVarMobileForceDepthResolve use in ConditionalResolveSceneDepth.
	const bool bForceDepthResolve = (CVarMobileForceDepthResolve.GetValueOnRenderThread() == 1);
	const bool bSeparateTranslucencyActive = IsMobileSeparateTranslucencyActive(Views.GetData(), Views.Num()); 
	const bool bPostProcessUsesSceneDepth = PostProcessUsesSceneDepth(Views[0]);
	bRequiresMultiPass = RequiresMultiPass(RHICmdList, Views[0]);
	bKeepDepthContent = 
		bRequiresMultiPass || 
		bForceDepthResolve ||
		bRequiresPixelProjectedPlanarRelfectionPass ||
		bSeparateTranslucencyActive ||
		Views[0].bIsReflectionCapture ||
		(bDeferredShading && bPostProcessUsesSceneDepth) ||
		bShouldRenderVelocities ||
		bIsFullPrepassEnabled;
    
	// never keep MSAA depth
	bKeepDepthContent = (NumMSAASamples > 1 ? false : bKeepDepthContent);

	// In the editor RHIs may split a render-pass into several cmd buffer submissions, so all targets need to Store
	if (IsSimulatedPlatform(ShaderPlatform))
	{
		bKeepDepthContent = true;
	}
    
	// Initialize global system textures (pass-through if already initialized).
	GSystemTextures.InitializeTextures(RHICmdList, FeatureLevel);
	FSceneRenderTargets& SceneContext = FSceneRenderTargets::Get(RHICmdList);

	// Allocate the maximum scene render target space for the current view family.
	SceneContext.SetKeepDepthContent(bKeepDepthContent);
	SceneContext.Allocate(RHICmdList, this);
	if (bDeferredShading)
	{
		ETextureCreateFlags AddFlags = bRequiresMultiPass ? TexCreate_InputAttachmentRead : (TexCreate_InputAttachmentRead | TexCreate_Memoryless);
		SceneContext.AllocGBufferTargets(RHICmdList, AddFlags);
	}

	// Initialise Sky/View resources before the view global uniform buffer is built.
	if (ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags))
	{
		InitSkyAtmosphereForViews(RHICmdList);
	}
	if (bRequiresPixelProjectedPlanarRelfectionPass)
	{
		InitPixelProjectedReflectionOutputs(RHICmdList, PlanarReflectionSceneProxy->RenderTarget->GetSizeXY());
	}
	else
	{
		ReleasePixelProjectedReflectionOutputs();
	}

	if (bRequiresAmbientOcclusionPass)
	{
		InitAmbientOcclusionOutputs(RHICmdList, SceneContext.SceneDepthZ);
	}
	else
	{
		ReleaseAmbientOcclusionOutputs();
	}

	if(bRequiresDistanceFieldShadowingPass)
	{
		InitSDFShadowingOutputs(RHICmdList, SceneContext.SceneDepthZ);
	}
	else
	{
		ReleaseSDFShadowingOutputs();
	}

	//make sure all the targets we're going to use will be safely writable.
	GRenderTargetPool.TransitionTargetsWritable(RHICmdList);

	// Find out whether custom depth pass should be rendered.
	{
		bool bCouldUseCustomDepthStencil = !bGammaSpace && (!Scene->World || (Scene->World->WorldType != EWorldType::EditorPreview && Scene->World->WorldType != EWorldType::Inactive));
		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			Views[ViewIndex].bCustomDepthStencilValid = bCouldUseCustomDepthStencil && UsesCustomDepthStencilLookup(Views[ViewIndex]);
			bShouldRenderCustomDepth |= Views[ViewIndex].bCustomDepthStencilValid;
		}
	}

#if PLATFORM_HOLOLENS
	// Check if any material renders depth to translucent materials.
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		bShouldRenderDepthToTranslucency |= Views[ViewIndex].bShouldRenderDepthToTranslucency;
	}
#endif
	
	const bool bDynamicShadows = ViewFamily.EngineShowFlags.DynamicShadows;
	
	if (bDynamicShadows && !IsSimpleForwardShadingEnabled(ShaderPlatform))
	{
		// Setup dynamic shadows.
		InitDynamicShadows(RHICmdList);		
	}
	else
	{
		// TODO: only do this when CSM + static is required.
		PrepareViewVisibilityLists();
	}

	/** Before SetupMobileBasePassAfterShadowInit, we need to update the uniform buffer and shadow info for all movable point lights.*/
	UpdateMovablePointLightUniformBufferAndShadowInfo();

	SetupMobileBasePassAfterShadowInit(BasePassDepthStencilAccess, ViewCommandsPerView);

	// if we kicked off ILC update via task, wait and finalize.
	if (ILCTaskData.TaskRef.IsValid())
	{
		Scene->IndirectLightingCache.FinalizeCacheUpdates(Scene, *this, ILCTaskData);
	}

	// initialize per-view uniform buffer.  Pass in shadow info as necessary.
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		FViewInfo& View = Views[ViewIndex];
		
		if (bDeferredShading)
		{
			if (View.ViewState)
			{
				if (!View.ViewState->ForwardLightingResources)
				{
					View.ViewState->ForwardLightingResources.Reset(new FForwardLightingViewResources());
				}
				View.ForwardLightingResources = View.ViewState->ForwardLightingResources.Get();
			}
			else
			{
				View.ForwardLightingResourcesStorage.Reset(new FForwardLightingViewResources());
				View.ForwardLightingResources = View.ForwardLightingResourcesStorage.Get();
			}
		}
	
		if (View.ViewState)
		{
			View.ViewState->UpdatePreExposure(View);
		}

		// Initialize the view's RHI resources.
		View.InitRHIResources();

		// TODO: remove when old path is removed
		// Create the directional light uniform buffers
		CreateDirectionalLightUniformBuffers(View);

		// Get the custom 1x1 target used to store exposure value and Toggle the two render targets used to store new and old.
		if (IsMobileEyeAdaptationEnabled(View))
		{
			View.SwapEyeAdaptationBuffers();
		}
	}

	UpdateGPUScene(RHICmdList, *Scene);
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		UploadDynamicPrimitiveShaderDataForView(RHICmdList, *Scene, Views[ViewIndex]);
	}

	if (bRequiresDistanceField)
	{
		PrepareDistanceFieldScene(RHICmdList, false);
	}

	extern TSet<IPersistentViewUniformBufferExtension*> PersistentViewUniformBufferExtensions;

	for (IPersistentViewUniformBufferExtension* Extension : PersistentViewUniformBufferExtensions)
	{
		Extension->BeginFrame();

		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			// Must happen before RHI thread flush so any tasks we dispatch here can land in the idle gap during the flush
			Extension->PrepareView(&Views[ViewIndex]);
		}
	}

	// update buffers used in cached mesh path
	// in case there are multiple views, these buffers will be updated before rendering each view
	if (Views.Num() > 0)
	{
		const FViewInfo& View = Views[0];
		// We want to wait for the extension jobs only when the view is being actually rendered for the first time
		Scene->UniformBuffers.UpdateViewUniformBuffer(View, false);
		UpdateOpaqueBasePassUniformBuffer(RHICmdList, View);
		UpdateTranslucentBasePassUniformBuffer(RHICmdList, View);
		UpdateDirectionalLightUniformBuffers(RHICmdList, View);
	}
	if (bDeferredShading)
	{
		SetupSceneReflectionCaptureBuffer(RHICmdList);
	}
	UpdateSkyReflectionUniformBuffer();

	// Now that the indirect lighting cache is updated, we can update the uniform buffers.
	UpdatePrimitiveIndirectLightingCacheBuffers();
	
	OnStartRender(RHICmdList);

	// Whether to submit cmdbuffer with offscreen rendering before doing post-processing
	bSubmitOffscreenRendering = (!bGammaSpace || bRenderToSceneColor) && CVarMobileFlushSceneColorRendering.GetValueOnAnyThread() != 0;
}
```
               



```cpp
void FSceneRenderer::ComputeViewVisibility(FRHICommandListImmediate& RHICmdList, FExclusiveDepthStencil::Type BasePassDepthStencilAccess, FViewVisibleCommandsPerView& ViewCommandsPerView, 
	FGlobalDynamicIndexBuffer& DynamicIndexBuffer, FGlobalDynamicVertexBuffer& DynamicVertexBuffer, FGlobalDynamicReadBuffer& DynamicReadBuffer)
{
	SCOPE_CYCLE_COUNTER(STAT_ViewVisibilityTime);
	SCOPED_NAMED_EVENT(FSceneRenderer_ComputeViewVisibility, FColor::Magenta);

	STAT(int32 NumProcessedPrimitives = 0);
	STAT(int32 NumCulledPrimitives = 0);
	STAT(int32 NumOccludedPrimitives = 0);

	// Allocate the visible light info.
	if (Scene->Lights.GetMaxIndex() > 0)
	{
		VisibleLightInfos.AddZeroed(Scene->Lights.GetMaxIndex());
	}

	int32 NumPrimitives = Scene->Primitives.Num();
	float CurrentRealTime = ViewFamily.CurrentRealTime;

	FPrimitiveViewMasks HasDynamicMeshElementsMasks;
	HasDynamicMeshElementsMasks.AddZeroed(NumPrimitives);

	FPrimitiveViewMasks HasDynamicEditorMeshElementsMasks;

	if (GIsEditor)
	{
		HasDynamicEditorMeshElementsMasks.AddZeroed(NumPrimitives);
	}

	const bool bIsInstancedStereo = (Views.Num() > 0) ? (Views[0].IsInstancedStereoPass() || Views[0].bIsMobileMultiViewEnabled) : false;
	UpdateReflectionSceneData(Scene);

	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_ViewVisibilityTime_ConditionalUpdateStaticMeshesWithoutVisibilityCheck);
		SCOPED_NAMED_EVENT(FSceneRenderer_ConditionalUpdateStaticMeshes, FColor::Red);

		Scene->ConditionalMarkStaticMeshElementsForUpdate();

		TArray<FPrimitiveSceneInfo*> UpdatedSceneInfos;
		for (TSet<FPrimitiveSceneInfo*>::TIterator It(Scene->PrimitivesNeedingStaticMeshUpdateWithoutVisibilityCheck); It; ++It)
		{
			FPrimitiveSceneInfo* Primitive = *It;
			if (Primitive->NeedsUpdateStaticMeshes())
			{
				UpdatedSceneInfos.Add(Primitive);
			}
		}
		if (UpdatedSceneInfos.Num() > 0)
		{
			FPrimitiveSceneInfo::UpdateStaticMeshes(RHICmdList, Scene, UpdatedSceneInfos);
		}
		Scene->PrimitivesNeedingStaticMeshUpdateWithoutVisibilityCheck.Reset();
	}

	uint8 ViewBit = 0x1;
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex, ViewBit <<= 1)
	{
		STAT(NumProcessedPrimitives += NumPrimitives);

		FViewInfo& View = Views[ViewIndex];
		FViewCommands& ViewCommands = ViewCommandsPerView[ViewIndex];
		FSceneViewState* ViewState = (FSceneViewState*)View.State;

		// ⭐⭐⭐⭐⭐⭐⭐
		// Primitive들의 Visibility 여부를 담은 BitArray를 초기화한다.
		//
		// Allocate the view's visibility maps.
		View.PrimitiveVisibilityMap.Init(false,Scene->Primitives.Num());
		// ⭐⭐⭐⭐⭐⭐⭐

		// we don't initialized as we overwrite the whole array (in GatherDynamicMeshElements)
		View.DynamicMeshEndIndices.SetNumUninitialized(Scene->Primitives.Num());
		View.PrimitiveDefinitelyUnoccludedMap.Init(false,Scene->Primitives.Num());
		View.PotentiallyFadingPrimitiveMap.Init(false,Scene->Primitives.Num());
		View.PrimitiveFadeUniformBuffers.AddZeroed(Scene->Primitives.Num());
		View.PrimitiveFadeUniformBufferMap.Init(false, Scene->Primitives.Num());
		View.StaticMeshVisibilityMap.Init(false,Scene->StaticMeshes.GetMaxIndex());
		View.StaticMeshFadeOutDitheredLODMap.Init(false,Scene->StaticMeshes.GetMaxIndex());
		View.StaticMeshFadeInDitheredLODMap.Init(false,Scene->StaticMeshes.GetMaxIndex());
		View.PrimitivesLODMask.Init(FLODMask(), Scene->Primitives.Num());
		View.DistanceCullingPrimitiveMap.Init(false, Scene->Primitives.Num());

		View.VisibleLightInfos.Empty(Scene->Lights.GetMaxIndex());

		// The dirty list allocation must take into account the max possible size because when GILCUpdatePrimTaskEnabled is true,
		// the indirect lighting cache will be update on by threaded job, which can not do reallocs on the buffer (since it uses the SceneRenderingAllocator).
		View.DirtyIndirectLightingCacheBufferPrimitives.Reserve(Scene->Primitives.Num());

		for(int32 LightIndex = 0;LightIndex < Scene->Lights.GetMaxIndex();LightIndex++)
		{
			if( LightIndex+2 < Scene->Lights.GetMaxIndex() )
			{
				if (LightIndex > 2)
				{
					FLUSH_CACHE_LINE(&View.VisibleLightInfos(LightIndex-2));
				}
				// @todo optimization These prefetches cause asserts since LightIndex > View.VisibleLightInfos.Num() - 1
				//FPlatformMisc::Prefetch(&View.VisibleLightInfos[LightIndex+2]);
				//FPlatformMisc::Prefetch(&View.VisibleLightInfos[LightIndex+1]);
			}
			new(View.VisibleLightInfos) FVisibleLightViewInfo();
		}

		View.PrimitiveViewRelevanceMap.Empty(Scene->Primitives.Num());
		View.PrimitiveViewRelevanceMap.AddZeroed(Scene->Primitives.Num());

		// If this is the visibility-parent of other views, reset its ParentPrimitives list.
		const bool bIsParent = ViewState && ViewState->IsViewParent();
		if ( bIsParent )
		{
			// PVS-Studio does not understand the validation of ViewState above, so we're disabling
			// its warning that ViewState may be null:
			ViewState->ParentPrimitives.Empty(); //-V595
		}


		// ⭐
		// Precomputed Visbility 데이터를 가져온다.
		if (ViewState)
		{	
			SCOPE_CYCLE_COUNTER(STAT_DecompressPrecomputedOcclusion);
			View.PrecomputedVisibilityData = ViewState->GetPrecomputedVisibilityData(View, Scene);
		}
		else
		{
			View.PrecomputedVisibilityData = NULL;
		}

		if (View.PrecomputedVisibilityData)
		{
			bUsedPrecomputedVisibility = true;
		}
		// ⭐

		bool bNeedsFrustumCulling = true;

		// ...
		// ... 디버깅 관련, 에디터 전용 처리들
		// ...

		// ⭐⭐⭐⭐⭐⭐⭐
		// Frustum Culling을 수행한다.
		// UE4에서는 Frustum Culling을 병렬로 처리한다.
		//
		// Most views use standard frustum culling.
		if (bNeedsFrustumCulling)
		{
			// Update HLOD transition/visibility states to allow use during distance culling
			FLODSceneTree& HLODTree = Scene->SceneLODHierarchy;
			if (HLODTree.IsActive())
			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_ViewVisibilityTime_HLODUpdate);
				HLODTree.UpdateVisibilityStates(View);
			}
			else
			{
				HLODTree.ClearVisibilityState(View);
			}

			int32 NumCulledPrimitivesForView;
			const bool bUseFastIntersect = (View.ViewFrustum.PermutedPlanes.Num() == 8) && CVarUseFastIntersect.GetValueOnRenderThread();
			if (View.CustomVisibilityQuery && View.CustomVisibilityQuery->Prepare())
			{
				if (CVarAlsoUseSphereForFrustumCull.GetValueOnRenderThread())
				{
					NumCulledPrimitivesForView = bUseFastIntersect ? FrustumCull<true, true, true>(Scene, View) : FrustumCull<true, true, false>(Scene, View);
				}
				else
				{
					NumCulledPrimitivesForView = bUseFastIntersect ? FrustumCull<true, false, true>(Scene, View) : FrustumCull<true, false, false>(Scene, View);
				}
			}
			else
			{
				if (CVarAlsoUseSphereForFrustumCull.GetValueOnRenderThread())
				{
					NumCulledPrimitivesForView = bUseFastIntersect ? FrustumCull<false, true, true>(Scene, View) : FrustumCull<false, true, false>(Scene, View);
				}
				else
				{
					NumCulledPrimitivesForView = bUseFastIntersect ? FrustumCull<false, false, true>(Scene, View) : FrustumCull<false, false, false>(Scene, View);
				}
			}
			STAT(NumCulledPrimitives += NumCulledPrimitivesForView);
			UpdatePrimitiveFading(Scene, View);			
		}
		// ⭐⭐⭐⭐⭐⭐⭐


		// ⭐
		// 이 View에 대해 Primitive가 숨겨짐 처리가 되어 있다면 그 Primitive도 PrimitiveVisibilityMap에서 숨겨짐 처리를 한다.
		//
		// If any primitives are explicitly hidden, remove them now.
		if (View.HiddenPrimitives.Num())
		{
			for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
			{
				if (View.HiddenPrimitives.Contains(Scene->PrimitiveComponentIds[BitIt.GetIndex()]))
				{
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
				}
			}
		}
		// ⭐


		// If the view has any show only primitives, hide everything else
		if (View.ShowOnlyPrimitives.IsSet())
		{
			View.bHasNoVisiblePrimitive = View.ShowOnlyPrimitives->Num() == 0;
			for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
			{
				if (!View.ShowOnlyPrimitives->Contains(Scene->PrimitiveComponentIds[BitIt.GetIndex()]))
				{
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
				}
			}
		}

		if (View.bStaticSceneOnly)
		{
			for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
			{
				// Reflection captures should only capture objects that won't move, since reflection captures won't update at runtime
				if (!Scene->Primitives[BitIt.GetIndex()]->Proxy->HasStaticLighting())
				{
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
				}
			}
		}

		// ...
		// ... 디버깅 관련, 에디터 전용 처리들
		// ...

		// ⭐⭐⭐⭐⭐⭐⭐
		// Occlusion Culling을 수행한다.
		//
		// Occlusion cull for all primitives in the view frustum, but not in wireframe.
		if (!View.Family->EngineShowFlags.Wireframe)
		{
			int32 NumOccludedPrimitivesInView = OcclusionCull(RHICmdList, Scene, View, DynamicVertexBuffer);
			STAT(NumOccludedPrimitives += NumOccludedPrimitivesInView);
		}
		// ⭐⭐⭐⭐⭐⭐⭐

		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_ViewVisibilityTime_ConditionalUpdateStaticMeshes);
			SCOPED_NAMED_EVENT(FSceneRenderer_UpdateStaticMeshes, FColor::Red);

			TArray<FPrimitiveSceneInfo*> AddedSceneInfos;
			for (TConstDualSetBitIterator<SceneRenderingBitArrayAllocator, FDefaultBitArrayAllocator> BitIt(View.PrimitiveVisibilityMap, Scene->PrimitivesNeedingStaticMeshUpdate); BitIt; ++BitIt)
			{
				int32 PrimitiveIndex = BitIt.GetIndex();
				AddedSceneInfos.Add(Scene->Primitives[PrimitiveIndex]);
			}

			if (AddedSceneInfos.Num() > 0)
			{
				FPrimitiveSceneInfo::UpdateStaticMeshes(RHICmdList, Scene, AddedSceneInfos);
			}
		}

		// ...
		// ... 디버깅 관련, 에디터 전용 처리들
		// ...

		// TODO: right now decals visibility computed right before rendering them, ideally it should be done in InitViews and this flag should be replaced with list of visible decals  
	    // Currently used to disable stencil operations in forward base pass when scene has no any decals
		View.bSceneHasDecals = (Scene->Decals.Num() > 0) || (GForceSceneHasDecals != 0);
	}

	// ...
	// ... VR용 렌더링 관련 처리
	// ...
	
	ViewBit = 0x1;
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
	{
		FViewInfo& View = Views[ViewIndex];
		FViewCommands& ViewCommands = ViewCommandsPerView[ViewIndex];
		
		if (bIsInstancedStereo)
		{
			SCOPE_CYCLE_COUNTER(STAT_ViewRelevance);
			ComputeAndMarkRelevanceForViewParallel(RHICmdList, Scene, View, ViewCommands, ViewBit, HasDynamicMeshElementsMasks, HasDynamicEditorMeshElementsMasks);
		}
		ViewBit <<= 1;
	}


	// ⭐⭐⭐⭐⭐⭐⭐
	// Dynamic Mesh들에 대한 MeshBatch, FMeshDrawCommand 생성한다.
	// 참고 자료 : https://scahp.tistory.com/75?category=848072
	{
		SCOPED_NAMED_EVENT(FSceneRenderer_GatherDynamicMeshElements, FColor::Yellow);
		// Gather FMeshBatches from scene proxies
		GatherDynamicMeshElements(Views, Scene, ViewFamily, DynamicIndexBuffer, DynamicVertexBuffer, DynamicReadBuffer,
			HasDynamicMeshElementsMasks, HasDynamicEditorMeshElementsMasks, MeshCollector);
	}
	// ⭐⭐⭐⭐⭐⭐⭐


	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		FViewInfo& View = Views[ViewIndex];
		if (!View.ShouldRenderView())
		{
			continue;
		}

		FViewCommands& ViewCommands = ViewCommandsPerView[ViewIndex];

#if !UE_BUILD_SHIPPING
		DumpPrimitives(ViewCommands);
#endif

		SetupMeshPass(View, BasePassDepthStencilAccess, ViewCommands);
	}

	INC_DWORD_STAT_BY(STAT_ProcessedPrimitives,NumProcessedPrimitives);
	INC_DWORD_STAT_BY(STAT_CulledPrimitives,NumCulledPrimitives);
	INC_DWORD_STAT_BY(STAT_OccludedPrimitives,NumOccludedPrimitives);
}
```


------------------------------         

references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
