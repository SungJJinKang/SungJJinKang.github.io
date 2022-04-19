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
	// 아래로 내려가서 자세한 분석을 읽으세요.
	ComputeViewVisibility(RHICmdList, BasePassDepthStencilAccess, ViewCommandsPerView, DynamicIndexBuffer, DynamicVertexBuffer, DynamicReadBuffer);
	// ⭐⭐⭐⭐⭐⭐⭐

	PostVisibilityFrameSetup(ILCTaskData);

	const FIntPoint RenderTargetSize = (ViewFamily.RenderTarget->GetRenderTargetTexture().IsValid()) ? ViewFamily.RenderTarget->GetRenderTargetTexture()->GetSizeXY() : ViewFamily.RenderTarget->GetSizeXY();

	// ⭐
	// 업스케일링이 필요한지 여부.
	const bool bRequiresUpscale = 
		((int32)RenderTargetSize.X > FamilySize.X || (int32)RenderTargetSize.Y > FamilySize.Y) 
		// in the editor color surface and backbuffer could have a different pixel formats and size, 
		// so we always run upscale pass to blit content from scene color to backbuffer 
		|| (GIsEditor && !IsMobileHDR() && NumMSAASamples > 1);
	// ⭐

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

	// ⭐
	// 포스트 프로세싱을 위해 Depth Buffer를 저장해둘지 여부. 
	bKeepDepthContent = 
		bRequiresMultiPass || 
		bForceDepthResolve ||
		bRequiresPixelProjectedPlanarRelfectionPass ||
		bSeparateTranslucencyActive ||
		Views[0].bIsReflectionCapture ||
		(bDeferredShading && bPostProcessUsesSceneDepth) ||
		bShouldRenderVelocities ||
		bIsFullPrepassEnabled;
	//
    
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
		// ⭐
		// Ambient Occlusion을 위한 사전 작업
		InitAmbientOcclusionOutputs(RHICmdList, SceneContext.SceneDepthZ);
		// ⭐
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

	
	// ⭐
	// CustomDepthStencil Pass를 그릴지 여부.
	//
	// Find out whether custom depth pass should be rendered.
	{
		bool bCouldUseCustomDepthStencil = !bGammaSpace && (!Scene->World || (Scene->World->WorldType != EWorldType::EditorPreview && Scene->World->WorldType != EWorldType::Inactive));
		for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
		{
			Views[ViewIndex].bCustomDepthStencilValid = bCouldUseCustomDepthStencil && UsesCustomDepthStencilLookup(Views[ViewIndex]);
			bShouldRenderCustomDepth |= Views[ViewIndex].bCustomDepthStencilValid;
		}
	}
	// ⭐

	// ...
	// ... 홀로렌즈 관련 처리
	// ...
	
	const bool bDynamicShadows = ViewFamily.EngineShowFlags.DynamicShadows;
	
	if (bDynamicShadows && !IsSimpleForwardShadingEnabled(ShaderPlatform))
	{
		// ⭐
		// Dynamic Shadow를 위한 사전 작업
		//
		// Setup dynamic shadows.
		InitDynamicShadows(RHICmdList);		
		// ⭐
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

		// ⭐
		// 이 View에서 사용할 RHI 리소스들을 초기화한다.
		// ex) View와 관련된 각종 Uniform Buffer 데이터들을 초기화한다.
		//
		// Initialize the view's RHI resources.
		View.InitRHIResources();
		// ⭐

		// TODO: remove when old path is removed
		// Create the directional light uniform buffers
		CreateDirectionalLightUniformBuffers(View);

		// Get the custom 1x1 target used to store exposure value and Toggle the two render targets used to store new and old.
		if (IsMobileEyeAdaptationEnabled(View))
		{
			View.SwapEyeAdaptationBuffers();
		}
	}

	//
	// In order to have different primitives in the same instanced draw with primitive-specific parameters, 
	// supporting platforms (UseGPUScene) upload them to a scene-wide buffer (UpdateGPUScene) and index into it with a PrimitiveId. 
	// For FLocalVertexFactory, PrimitiveId comes from a instance-frequency vertex input stream. 
	// This must be passed to the pixel shader, which must use GetPrimitiveData(Parameters.PrimitiveId).
	// Member to access Primitive shader parameters, instead of accessing the primitive uniform buffer directly (Primitive.Member).
	//
	// https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/#gpuscene
	//
	UpdateGPUScene(RHICmdList, *Scene);
	//

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
               

**FSceneRenderer::ComputeViewVisibility** 함수는 Primitive에 대한 가시성 테스트 뿐만아니라 Dynamic Primitive에 대한 FMeshBatch, FMeshDrawCommand를 생성하는 매우 중요한 역할을 하는 함수이다.        

```cpp
void FSceneRenderer::ComputeViewVisibility
(
	FRHICommandListImmediate& RHICmdList, 
	FExclusiveDepthStencil::Type BasePassDepthStencilAccess, 

	// ⭐⭐⭐⭐⭐⭐⭐
	// 매우 중요한 Parameter 입니다.
	// 최종적으로 View에 그려질 Static, Dynamic Mesh들에 대한 FMeshDrawCommand들이 이 함수에서 이 Parameter에 저장될 것 입니다. 
	FViewVisibleCommandsPerView& ViewCommandsPerView, 
	// ⭐⭐⭐⭐⭐⭐⭐

	FGlobalDynamicIndexBuffer& DynamicIndexBuffer, 
	FGlobalDynamicVertexBuffer& DynamicVertexBuffer,
	FGlobalDynamicReadBuffer& DynamicReadBuffer
)
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

	// ⭐⭐⭐⭐⭐⭐⭐
	// HasDynamicMeshElementsMasks는 FScene->Primitives와 같은 개수로 초기화되며, 
	// 아래 ComputeAndMarkRelevanceForViewParallel 함수에서 DynamicMesh 중 렌더링 될 Index에 1이 셋팅될 것 입니다.. 
	// 즉, DynamicMesh 용 VisibilityMap입니다.
	// 참고 자료 : https://scahp.tistory.com/75?category=848072
	FPrimitiveViewMasks HasDynamicMeshElementsMasks;
	HasDynamicMeshElementsMasks.AddZeroed(NumPrimitives);
	// ⭐⭐⭐⭐⭐⭐⭐

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

		// ⭐⭐⭐⭐⭐⭐⭐
		// 현재 View에서 렌더링 될 FMeshDrawCommand들이 저장될 것입니다.
		// Visibility 연산 결과에 따라 필요한 FMeshDrawCommand도 달라진다.
		// 참고 자료 : https://scahp.tistory.com/75?category=848072
		FViewInfo& View = Views[ViewIndex];
		// ⭐⭐⭐⭐⭐⭐⭐

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
		// 이 View에 대해 어떤 Primitive가 숨겨짐 처리가 되어 있다면 그 Primitive도 PrimitiveVisibilityMap에서 숨겨짐 처리를 한다.
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
		// Precomputed Visibility를 통한 Culling도 여기서 수행한다.
		// 아래로 내려가서 자세한 분석을 보시기 바랍니다.
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

	
	// ⭐⭐⭐⭐⭐⭐⭐
	// 여기까지 왔다면 월드 상의 모든 Primitive들에 대한 가시성 테스트는 끝났습니다.
	// 이제 그 가시성 테스트 결과를 바탕으로 화면 상에 보이는 Primitive들에 대한 FMeshDrawCommand들을 모을 것 입니다.
	// ⭐⭐⭐⭐⭐⭐⭐
	

	// ⭐⭐⭐⭐⭐⭐⭐
	// ComputeAndMarkRelevanceForViewParallel 함수는 매우 중요한 역할을 담당한다.
	//
	// 1. 위의 가시성 테스트 결과를 가지고 렌더링 될 StaticMesh의 FMeshDrawCommand를 모읍니다.
	// 2. 렌더링 될 ( 화면상에 보이는 ) DynamicMesh들의 가시성 정보인 HasDynamicMeshElementsMasks를 채웁니다. DynamicMesh의 경우 해당 Index의 위치에 1이 쓰입니다.
	// 
	// 그리고 위의 두 동작을 동작을 병렬로 수행한다.
	//
	// ComputeAndMarkRelevanceForViewParallel 함수가 끝나면 StaticMesh들의 FMeshDrawCommand는 ViewCommands에 들어 있습니다.
	//
	// 참고 자료 : https://scahp.tistory.com/75?category=848072
	ViewBit = 0x1;
	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ++ViewIndex)
	{
		FViewInfo& View = Views[ViewIndex];

		// ⭐⭐⭐⭐⭐⭐⭐
		// 특정 View에 그려진 Mesh들에 대한 FMeshDrawCommand이 저장되는 곳!
		// 아래 ComputeAndMarkRelevanceForViewParallel 함수에서 이 FViewCommands로 StaticMesh(!)들에 대한 ViewCommands가 들어갑니다.
		// 물론 가시성 결과를 토대로 View에 그려질 StaticMesh들에 대한 FViewCommands만 들어갑니다.
		// StaticMesh들의 FMeshDrawCommand는 미리 캐싱이 되어 있기 때문에 빠르게 옮길 수 있습니다.
		FViewCommands& ViewCommands = ViewCommandsPerView[ViewIndex];
		// ⭐⭐⭐⭐⭐⭐⭐

		if (bIsInstancedStereo)
		{
			SCOPE_CYCLE_COUNTER(STAT_ViewRelevance);
			ComputeAndMarkRelevanceForViewParallel(RHICmdList, Scene, View, ViewCommands, ViewBit, HasDynamicMeshElementsMasks, HasDynamicEditorMeshElementsMasks);
		}
		ViewBit <<= 1;
	}
	// ⭐⭐⭐⭐⭐⭐⭐


	// ⭐⭐⭐⭐⭐⭐⭐
	// 여기까지 왔으면 렌더링 될 Static Mesh들에 대한 FMeshDrawCommand는 모두 ViewCommandsPerView에 저장되어 있습니다.
	// ⭐⭐⭐⭐⭐⭐⭐


	// ⭐⭐⭐⭐⭐⭐⭐
	// Dynamic Mesh들에 대한 FMeshBatch를 생성한다.
	//
	// Static Mesh의 경우 미리 FMeshBatch, FMeshDrawCommands를 미리 만들어두고 캐싱을 해두지만,
	// Dynamic Mesh의 경우 매 프레임 그때 그때 FMeshBatch, FMeshDrawCommand를 생성합니다.
	// 일단 여기서는 DynamicMesh에 대한 FMeshBatch만 생성합니다.
	// FMeshBatch는 Primitive를 렌더링할 때 필요한 모든 데이터를 가지고 있습니다.
	// 모든 Pass에서의 렌더링에 필요한 모든 데이터를 가지고 있기 때문에 특정 Pass에 대해 Stateless하다고 합니다.
	//
	// Dynamic Mesh들에 대한 FMeshBatch는 View.DynamicMeshElements에 저장됩니다.
	//
	// 참고 자료 : https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/
	{
		SCOPED_NAMED_EVENT(FSceneRenderer_GatherDynamicMeshElements, FColor::Yellow);
		// Gather FMeshBatches from scene proxies
		GatherDynamicMeshElements(Views, Scene, ViewFamily, DynamicIndexBuffer, DynamicVertexBuffer, DynamicReadBuffer,
			HasDynamicMeshElementsMasks, HasDynamicEditorMeshElementsMasks, MeshCollector);
	}
	// ⭐⭐⭐⭐⭐⭐⭐


	// ⭐⭐⭐⭐⭐⭐⭐
	// 여기까지 왔다면 
	// 화면 상에 그려질 모든 Static Mesh들에 대한 FMeshDrawCommand는 ViewCommands에 들어있고,
	// 화면 상에 그려질 모든 Dynamic Mesh들에 대한 FMeshBatch는 View.DynamicMeshElements에 모두 들어 있습니다.
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

		// ⭐⭐⭐⭐⭐⭐⭐
		// 각 Pass 별로 
		// Static Mesh에 대한 FMeshDrawCommand를 모으고, 
		// Dynamic Mesh들에 대한 FMeshDrawCommand 생성하여,
		// View.ParallelMeshDrawCommandPasses에 모두 저장합니다.
		//  
		// 참고 자료 : https://scahp.tistory.com/75?category=848072
		SetupMeshPass(View, BasePassDepthStencilAccess, ViewCommands);
		// ⭐⭐⭐⭐⭐⭐⭐

	}

	INC_DWORD_STAT_BY(STAT_ProcessedPrimitives,NumProcessedPrimitives);
	INC_DWORD_STAT_BY(STAT_CulledPrimitives,NumCulledPrimitives);
	INC_DWORD_STAT_BY(STAT_OccludedPrimitives,NumOccludedPrimitives);
}
```


```cpp
static int32 OcclusionCull(FRHICommandListImmediate& RHICmdList, const FScene* Scene, FViewInfo& View, FGlobalDynamicVertexBuffer& DynamicVertexBuffer)
{
	SCOPE_CYCLE_COUNTER(STAT_OcclusionCull);	
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_OcclusionReadback));

	// INITVIEWS_TODO: This could be more efficient if broken up in to separate concerns:
	// - What is occluded?
	// - For which primitives should we render occlusion queries?
	// - Generate occlusion query geometry.

	int32 NumOccludedPrimitives = 0;
	FSceneViewState* ViewState = (FSceneViewState*)View.State;
	
	// ⭐
	// Hierarchical Z Buffer로 Occlusion Culling을 수행할지 결정한다.
	// 프로젝트 셋팅에서 설정을 할 수 있고, OpenGL이냐 여부에 따라 또 달라진다.
	//
	// Disable HZB on OpenGL platforms to avoid rendering artifacts
	// It can be forced on by setting HZBOcclusion to 2
	bool bHZBOcclusion = !IsOpenGLPlatform(GShaderPlatformForFeatureLevel[Scene->GetFeatureLevel()]);
	bHZBOcclusion = bHZBOcclusion && GHZBOcclusion;
	bHZBOcclusion = bHZBOcclusion && FDataDrivenShaderPlatformInfo::GetSupportsHZBOcclusion(GShaderPlatformForFeatureLevel[Scene->GetFeatureLevel()]);
	bHZBOcclusion = bHZBOcclusion || (GHZBOcclusion == 2);
	// ⭐

	// ⭐
	// Precomputed visibility 데이터를 가지고 가시성 체크를 수행한다. 
	//
	// Use precomputed visibility data if it is available.
	if (View.PrecomputedVisibilityData)
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_LookupPrecomputedVisibility);

		FViewElementPDI OcclusionPDI(&View, nullptr, nullptr);
		uint8 PrecomputedVisibilityFlags = EOcclusionFlags::CanBeOccluded | EOcclusionFlags::HasPrecomputedVisibility;
		for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
		{
			if ((Scene->PrimitiveOcclusionFlags[BitIt.GetIndex()] & PrecomputedVisibilityFlags) == PrecomputedVisibilityFlags)
			{
				FPrimitiveVisibilityId VisibilityId = Scene->PrimitiveVisibilityIds[BitIt.GetIndex()];
				if ((View.PrecomputedVisibilityData[VisibilityId.ByteIndex] & VisibilityId.BitMask) == 0)
				{
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
					INC_DWORD_STAT_BY(STAT_StaticallyOccludedPrimitives,1);
					STAT(NumOccludedPrimitives++);

					// ...
					// ... 디버깅용 처리
					// ...
				}
			}
		}
	}
	// ⭐

	float CurrentRealTime = View.Family->CurrentRealTime;
	if (ViewState)
	{
		if (ViewState->SceneSoftwareOcclusion)
		{	
			// ⭐
			// SW Occlusion 컬링을 수행하는 경우.
			// 잘못하면 겁나 느려진다.....
			// ⭐

			SCOPE_CYCLE_COUNTER(STAT_SoftwareOcclusionCull)
			NumOccludedPrimitives += ViewState->SceneSoftwareOcclusion->Process(RHICmdList, Scene, View);
		}
		else if (Scene->GetFeatureLevel() >= ERHIFeatureLevel::ES3_1)
		{
			// ⭐
			// 일반적인 HW Occlusion Query
			// ⭐

			// ⭐
			// 이번 프레임에도 Occlusion Query 테스트를 수행할 경우 bSubmitQueries는 true
			bool bSubmitQueries = !View.bDisableQuerySubmissions; 
			// ⭐
#if !(UE_BUILD_SHIPPING || UE_BUILD_TEST)
			bSubmitQueries = bSubmitQueries && !ViewState->HasViewParent() && !ViewState->bIsFrozen;
#endif

			if( bHZBOcclusion )
			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_MapHZBResults);
				check(!ViewState->HZBOcclusionTests.IsValidFrame(ViewState->OcclusionFrameCounter));
				ViewState->HZBOcclusionTests.MapResults(RHICmdList);
			}
 
			// ...
			// ... VR 관련 처리.
			// ...

			View.ViewState->PrimitiveOcclusionQueryPool.AdvanceFrame(
				ViewState->OcclusionFrameCounter,
				FOcclusionQueryHelpers::GetNumBufferedFrames(Scene->GetFeatureLevel()),
				View.ViewState->IsRoundRobinEnabled() && !View.bIsSceneCapture && IStereoRendering::IsStereoEyeView(View));

			// ⭐⭐⭐⭐⭐⭐⭐
			// Occlusion Query의 결과를 가져옵니다.
			// 옵션에 따라 병렬로도 처리하기도 합니다.
			NumOccludedPrimitives += FetchVisibilityForPrimitives(Scene, View, bSubmitQueries, bHZBOcclusion, DynamicVertexBuffer);
			// ⭐⭐⭐⭐⭐⭐⭐

			if( bHZBOcclusion )
			{
				QUICK_SCOPE_CYCLE_COUNTER(STAT_HZBUnmapResults);

				ViewState->HZBOcclusionTests.UnmapResults(RHICmdList);

				if( bSubmitQueries )
				{
					ViewState->HZBOcclusionTests.SetValidFrameNumber(ViewState->OcclusionFrameCounter);
				}
			}
		}
		else
		{
			// ⭐
			// OccOcclusion을 수행하지 않는 경우
			// ⭐

			// No occlusion queries, so mark primitives as not occluded
			for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
			{
				View.PrimitiveDefinitelyUnoccludedMap.AccessCorrespondingBit(BitIt) = true;
			}
		}
	}
	RHICmdList.SetCurrentStat(GET_STATID(STAT_CLMM_AfterOcclusionReadback));
	return NumOccludedPrimitives;
}
```


------------------------------         

references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
