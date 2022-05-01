---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석 - 2 ( FMobileSceneRenderer::InitViews )"
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

함께 보면 좋은 참고 자료 : [UE5 MeshDrawCommand (1/2)](https://scahp.tistory.com/74?category=848072), [UE5 MeshDrawCommand (2/2)](https://scahp.tistory.com/75?category=848), [Why Talking About Render Graphs](https://logins.github.io/graphics/2021/05/31/RenderGraphs.html), [EveryCulling](https://github.com/SungJJinKang/EveryCulling)         
                       
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
	// 가시성 결정(컬링) 연산을 수행한다.
	// 가시성 판단은 Primitive가 화면 상에 보여질지를 판단하는 과정으로,
	// 화면 상에 보이지 않는 Primitive에 대해서는 Draw Graphics API를 호출하지 않아, 드로우 콜을 줄이고자 하는 목적으로 수행한다.
	// 대부분은 CPU에서 이러한 연산을 수행하여, GPU의 연산 부담을 덜어주기 위한 목적으로 컬링을 수행한다. ( HW Occlusion Query의 경우 GPU를 활용한다. )
	//
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
		// Frustum Culling이란,
		// 카메라의 절두체를 기준으로 어떤 Primivie가 절두체 밖에 있는 경우 해당 Primitive는 화면에 보이지 않는 것으로 판단을해 Draw Graphics API를 호출하지 않는 것이다.
		// 일반적으로는 Primitive의 AABB ( 바운딩 박스 )를 기준으로 카메라 절두체 Plane과 비교한다.
		// 바운딩 박스를 이용하는 이유는 그 만큼 연산량이 적기 때문이다. 
		// Primitive의 Vertex마다 일일이 절두체 안에 속해 있는지 비교 연산을 한다고 생각을 해보아라.
		// 연산량이 엄청날 것이다. 그렇기 때문에 정확도는 조금 낮을 수 있지만 상대적으로 가벼운 연산량으로 Frustum Culling 테스트를 수행하는 것이다.
		//
		// 엄연히 CPU에서 수행하는 것으로 드로우콜을 최대한 줄여서 GPU의 연산 부담을 낮추어주기 위한 목적이다.
		// 
		// 필자도 이를 직접 구현해보았다.
		// https://github.com/SungJJinKang/EveryCulling#view-frustum-culling-from-frostbite-engine-of-ea-dice--100-
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
		//
		// Occlusion Culling이란 기본적으로 Primitive가 다른 Primitive에 의해 가려져 있는지를 판단하는 것이다.
		// Primitive가 카메라 절두체 밖에 있는지 판단하는 Frustum Culling과는 엄연히 다른 것이다.
		//
		// Primitive들을 단순하게 그려보았을 때 ( Fragment Shading 단계에서 그냥 점만 찍어서 Depth Test를 통과했는지만 확인하여 ),
		// 모든 Fragment가 Depth Test를 통과하지 못하였다면 해당 Primitive는 Cull된 것으로 판단한다.
		// 일반적으로는 Primitive의 AABB ( 바운딩 박스 )를 그려보는 방식을 사용한다.
		// Primitive의 원본을 그리려면 너무 연산량이 많으니 바운딩 박스를 그려보며 바운딩 박스가 Occlude 되었는지 ( 모든 Fragment들이 Depth Test를 통과하지 못하였는지 ) 확인하고,
		// 그렇다면 해당 Primitive도 Cull되었다고 판단하는 것이다.
		// Primitive 원본을 가지고 Occlusion 테스트를 하는 것보다 연산량을 엄청나게 줄일 수 있다.
		// 다만 Cull해도 될 오브젝트를 그리는 경우 ( Draw Call )가 생길 수도 있다.
		//
		// 대표적으로 CPU에서 수행하는 SW Occlusion Culling과 GPU를 활용하는 하드웨어 Occlusion Query 두 방법이 있다.
		//
		// 기본 값은 HW Occlusion Query를 사용한다. 
		// HW Occlusion Query의 경우 Primitive들을 GPU로 위에서 말한 것과 같이 한번 그려본 후 Query의 결과 값을 다시 시스템 메모리 ( Host 메모리 )로 읽어와야한다.
		// VRAM에서 DRAM으로 데이터를 읽어오는 것은 엄청나게 느려터진 동작이다.......
		// VRAM을 읽는 것이 느린 이유는 이 링크를 따라가보면 이유를 알 수 있다 ( https://megayuchi.com/2021/06/06/ddraw-surface-d3d-dynamic-buffer-%EC%97%90%EC%84%9C%EC%9D%98-write-combine-memory/ )
		// ( Conditinal Rendering이라고 시스템 메모리로 읽어오지 않는 방법도 있는데 잘 사용하지 않는 것 같다..... )
		//
		// 또한 Occlusion Query도 결국에는 일반적으로 렌더링을 하는 것과 같이 똑같이 Primitive를 한번 그려보는 것이기 때문에 
		// ( 다만 Pixel Shader는 점만 찍어 Depth Test를 확인하는 방법으로 )
		// Occlusion Query 하나 하나가 Draw Call이다. 이 또한 비용이다.
		//
		//
		// SW Occlusion Culling은 CPU로 Rasterizing을 수행하는 것인데 겁나게 느리다.....
		// CPU는 이러한 대량의, 단순한 연산에 GPU에 비해 많이 느리다...
		// 그렇지만 흔히 GPU Bound한 게임에서는 HW Occlusion Query로 인한 연산 부담을 GPU로부터 덜어주기 위해 사용하기도 한다.
		// 필자의 경우에도 구현을 해보았고, 확실히 GPU Bound한 경우 큰 프레임 향상을 보여주었다.
		//
		// 아래의 링크에서 필자가 구현한 SW Occlusion Culling의 소스코드를 확인할 수 있다.
		// https://github.com/SungJJinKang/EveryCulling#masked-sw--cpu--occlusion-culling-from-intel--100-
		//
		//
		// 일반적으로 Frustum Culling을 처리하고, Culling이 되지 않는 오브젝트들을 대상으로 Occlusion Culling을 수행한다.
		// Frustum Culling의 연산 비용이 상대적으로 저렴하기 때문에 Frustum Culling을 먼저 수행하는 것이다.
		//
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

이제 **오클루전 컬링** 부분이다.         
여기서는 3 ~ 4 프레임 전 발행된 **Occlusion Query 결과를 가지고 Primitive들에 대한 가시성 여부를 PrimitiveVisibilityMap에 저장**하고,         
현재 프레임에서 **발행할 Query들을 결정**하는 동작만 수행한다.    
Query를 결정한다는 것은 결국 Query로 수행 될 Primitive들의 AABB Vertex 데이터를 모은다는 것이다.               
**실제로 Query를 GPU에 보내는 작업은 FMobileSceneRenderer::RenderForward에서 수행**된다.             
`

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


	// ⭐⭐⭐⭐⭐⭐⭐
	// Precomputed visibility 데이터를 가지고 가시성 체크를 수행한다. 
	// Precomputed Visibility는 빌드시 카메라의 특정 위치에서 Primitive가 컬링이 되었는지를 판단하여 미리 저장해두는 개념이다.
	// 월드를 바운딩 박스로 쪼갠 후 카메라가 각 바운딩 박스에 위치해 있다고 가정하고,
	// 월드 내에 Static으로 설정된 Primitive들이 보이는지를 빌드 타임에 연산하고 저장해두는 것이다.
	// 런타임 비용이 0에 가깝다.
	// 
	// Use precomputed visibility data if it is available.
	if (View.PrecomputedVisibilityData)
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_LookupPrecomputedVisibility);

		FViewElementPDI OcclusionPDI(&View, nullptr, nullptr);
		uint8 PrecomputedVisibilityFlags = EOcclusionFlags::CanBeOccluded | EOcclusionFlags::HasPrecomputedVisibility;
		for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap); BitIt; ++BitIt)
		{
			// ⭐
			// Primtive가 Precomputed Visibility Culling 처리 될 수 있다는 Flag를 가지고 있는지..
			if ((Scene->PrimitiveOcclusionFlags[BitIt.GetIndex()] & PrecomputedVisibilityFlags) == PrecomputedVisibilityFlags)
			// ⭐
			{
				// ⭐
				// Primitive의 PrimitiveVisibilityId를 Index로 하여 PrecomputedVisibilityData에서 Culling 여부를 판단.
				FPrimitiveVisibilityId VisibilityId = Scene->PrimitiveVisibilityIds[BitIt.GetIndex()];
				if ((View.PrecomputedVisibilityData[VisibilityId.ByteIndex] & VisibilityId.BitMask) == 0)
				// ⭐
				{
					// ⭐
					// Precomputed Visbility Data에서 Primitive가 컬링이 된 경우.
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
					INC_DWORD_STAT_BY(STAT_StaticallyOccludedPrimitives,1);
					STAT(NumOccludedPrimitives++);
					// ⭐

					// ...
					// ... 디버깅용 처리
					// ...
				}
			}
		}
	}
	// ⭐⭐⭐⭐⭐⭐⭐


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
			// 여기서 가져오는 Occlusion Query 결과는 몇 프레임 전 ( 3 ~ 4 프레임 ) 전에 발행한 ( Occludee에 대한 DrawCall을 보내는 ) Occlusion Query에 대한 결과 값이다.
			// Occlusion Query를 발행하는 부분은 이 글에서 다루지 않는다.
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

```cpp
static int32 FetchVisibilityForPrimitives(const FScene* Scene, FViewInfo& View, const bool bSubmitQueries, const bool bHZBOcclusion, FGlobalDynamicVertexBuffer& DynamicVertexBuffer)
{
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(FetchVisibilityForPrimitives);
	QUICK_SCOPE_CYCLE_COUNTER(STAT_FetchVisibilityForPrimitives);
	FSceneViewState* ViewState = (FSceneViewState*)View.State;
	
	static int32 SubIsOccludedArrayIndex = 0;
	SubIsOccludedArrayIndex = 1 - SubIsOccludedArrayIndex;

	const int32 NumBufferedFrames = FOcclusionQueryHelpers::GetNumBufferedFrames(Scene->GetFeatureLevel());
	uint32 OcclusionFrameCounter = ViewState->OcclusionFrameCounter;

	// ⭐
	// Primitive들의 이전 프레임에서의 가시성/오클루전 결과 값
	TSet<FPrimitiveOcclusionHistory, FPrimitiveOcclusionHistoryKeyFuncs>& ViewPrimitiveOcclusionHistory = ViewState->PrimitiveOcclusionHistorySet;
	// ⭐

	// ⭐
	// Occlusion Culling을 병렬로 처리하는 경우,
	// GOcclusionCullParallelPrimFetch, GSupportsParallelOcclusionQueries 두 변수 모두 기본 값은 false이다.
	if (GOcclusionCullParallelPrimFetch && GSupportsParallelOcclusionQueries)
	// ⭐
	{		
		// ...
		// ... 
		// ...
	}
	else
	{
		
		// ⭐
		//SubIsOccluded stuff needs a frame's lifetime
		//
		// 아래에서 설명하겠다.
		TArray<bool>& SubIsOccluded = View.FrameSubIsOccluded[SubIsOccludedArrayIndex];
		SubIsOccluded.Reset();
		// ⭐

		// ⭐
		// FetchVisibilityForPrimitives_Range에서 Query로 발행될 AABB들이 저장된다.
		static TArray<FOcclusionBounds> PendingIndividualQueriesWhenOptimizing;
		PendingIndividualQueriesWhenOptimizing.Reset();
		// ⭐

		static TArray<FOcclusionBounds*> PendingIndividualQueriesWhenOptimizingSorter;
		PendingIndividualQueriesWhenOptimizingSorter.Reset();

		FViewElementPDI OcclusionPDI(&View, nullptr, nullptr);
		int32 StartIndex = 0;
		int32 NumToProcess = View.PrimitiveVisibilityMap.Num();			

		
		// ⭐	
		FVisForPrimParams Params(
			Scene,
			&View,
			&OcclusionPDI, // FVisForPrimParams::OcclusionPDI
			StartIndex, // FVisForPrimParams::StartIndex
			NumToProcess, // FVisForPrimParams::NumToProcess
			bSubmitQueries, // FVisForPrimParams::bSubmitQueries
			bHZBOcclusion, // FVisForPrimParams::bHZBOcclusion			
			nullptr,
			nullptr,
			nullptr,
			// ⭐
			&PendingIndividualQueriesWhenOptimizing, // FVisForPrimParams::QueriesToAdd
			// ⭐
			&SubIsOccluded // FVisForPrimParams::SubIsOccluded
			);
		// ⭐

		// ⭐⭐⭐⭐⭐⭐⭐
		// 아래에서 정확한 분석을 확인해주세요
		FetchVisibilityForPrimitives_Range<true>(Params, &DynamicVertexBuffer);
		// ⭐⭐⭐⭐⭐⭐⭐

		// ⭐
		// Individual Query의 개수
		int32 IndQueries = PendingIndividualQueriesWhenOptimizing.Num();
		// ⭐
		if (IndQueries)
		{
			// ⭐
			// 한 프레임에 발행 가능한 최대 Query 개수
			// 앞서 말했듯이 3 ~ 4 프레임 전 Query를 사용하기 때문에,
			//
			// 한 프레임에 발행 가능한 최대 Query 개수
			// = ( GPU에서 지원하는 총 Query 수 ) / ( Query 버퍼 프레임 수 ( 3 ~ 4 프레임 ) )
			int32 SoftMaxQueries = GRHIMaximumReccommendedOustandingOcclusionQueries /* Occlusion Query 최대 가능 발행 횟수 */ / FMath::Min(NumBufferedFrames, 2); // extra RHIT frame does not count
			// ⭐

			// ⭐
			// Group된 Query의 개수
			// Group된 Query들이 Individual Query보다 처리 우선 순위가 높다.
			// ( 
			// 	처리하려는 Query 수가 한 프레임에 처리 가능한 Query 수보다 많은 경우,
			// 	Individual Query를 몇 개 버린다는 의미이다. 
			// )
			int32 UsedQueries = View.GroupedOcclusionQueries.GetNumBatchOcclusionQueries();
			// ⭐

			int32 FirstQueryToDo = 0;
			int32 QueriesToDo = IndQueries;


			// ⭐
			// Group된 Query와 Individual Query의 개수 합이 한 프레임에 발행 가능한 Query 개수보다 큰 경우
			// Individual Query 수를 줄인다.
			if (SoftMaxQueries < UsedQueries + IndQueries)
			{
				QueriesToDo = (IndQueries + 9) / 10;  // we need to make progress, even if it means stalling and waiting for the GPU. At a minimum, we will do 10%

				if (SoftMaxQueries > UsedQueries + QueriesToDo)
				{
					// we can do more than the minimum
					QueriesToDo = SoftMaxQueries - UsedQueries;
				}
			}
			// ⭐

			// ⭐
			// 처리 가능한 Query 수가 충분하여 위에서 Individual Query 수를 줄이지 않은 경우
			// 모든 Individual Query를 온전히 수행한다.
			// PrimitiveOcclusionHistory에 상응하는 Query를 저장해주어 나중에 Query의 결과를 얻을 수 있게 한다.
			if (QueriesToDo == IndQueries)
			{
				for (int32 Index = 0; Index < IndQueries; Index++)
				{
					FOcclusionBounds* RunQueriesIter = &PendingIndividualQueriesWhenOptimizing[Index];
					FPrimitiveOcclusionHistory* PrimitiveOcclusionHistory = ViewPrimitiveOcclusionHistory.Find(RunQueriesIter->PrimitiveOcclusionHistoryKey);

					PrimitiveOcclusionHistory->SetCurrentQuery(OcclusionFrameCounter,
						View.IndividualOcclusionQueries.BatchPrimitive(RunQueriesIter->BoundsOrigin, RunQueriesIter->BoundsExtent, DynamicVertexBuffer),
						NumBufferedFrames,
						false,
						Params.bNeedsScanOnRead
					);
				}
			}
			// ⭐


			// ⭐
			// 처리 가능한 Query 수가 충분하지 않아 Individual Query 수를 줄여야하는 경우,
			// Query를 수행한지 오래된 Primitive에 대한 Query를 우선으로 하여 처리한다.
			else
			{
				check(QueriesToDo < IndQueries);
				PendingIndividualQueriesWhenOptimizingSorter.Reserve(PendingIndividualQueriesWhenOptimizing.Num());
				for (int32 Index = 0; Index < IndQueries; Index++)
				{
					FOcclusionBounds* RunQueriesIter = &PendingIndividualQueriesWhenOptimizing[Index];
					PendingIndividualQueriesWhenOptimizingSorter.Add(RunQueriesIter);
				}

				PendingIndividualQueriesWhenOptimizingSorter.Sort(
					[](const FOcclusionBounds& A, const FOcclusionBounds& B) 
					{
						return A.LastQuerySubmitFrame < B.LastQuerySubmitFrame;
					}
				);
				for (int32 Index = 0; Index < QueriesToDo; Index++)
				{
					FOcclusionBounds* RunQueriesIter = PendingIndividualQueriesWhenOptimizingSorter[Index];
					FPrimitiveOcclusionHistory* PrimitiveOcclusionHistory = ViewPrimitiveOcclusionHistory.Find(RunQueriesIter->PrimitiveOcclusionHistoryKey);
					PrimitiveOcclusionHistory->SetCurrentQuery(OcclusionFrameCounter,
						View.IndividualOcclusionQueries.BatchPrimitive(RunQueriesIter->BoundsOrigin, RunQueriesIter->BoundsExtent, DynamicVertexBuffer),
						NumBufferedFrames,
						false,
						Params.bNeedsScanOnRead
					);
				}
				// ⭐
			}


			// lets prevent this from staying too large for too long
			if (PendingIndividualQueriesWhenOptimizing.GetSlack() > IndQueries * 4)
			{
				PendingIndividualQueriesWhenOptimizing.Empty();
				PendingIndividualQueriesWhenOptimizingSorter.Empty();
			}
			else
			{
				PendingIndividualQueriesWhenOptimizing.Reset();
				PendingIndividualQueriesWhenOptimizingSorter.Reset();
			}
		}
		return Params.NumOccludedPrims;
	}
}
```

```cpp
//This function is shared between the single and multi-threaded versions.  Modifications to any primitives indexed by BitIt should be ok
//since only one of the task threads will ever reference it.  However, any modifications to shared state like the ViewState must be buffered
//to be recombined later.
// ⭐
// 엔진의 기본 값은 Occlusion Culling을 싱글 스레드에서 수행하는 것이다.
// 그러므로 true이다.
template<bool bSingleThreaded>
// ⭐
static void FetchVisibilityForPrimitives_Range(FVisForPrimParams& Params, FGlobalDynamicVertexBuffer* DynamicVertexBufferIfSingleThreaded)
{	
	int32 NumOccludedPrimitives = 0;
	
	const FScene* Scene				= Params.Scene;
	FViewInfo& View					= *Params.View;
	FViewElementPDI* OcclusionPDI	= Params.OcclusionPDI;

	// ⭐
	// Occlusion 커링의 대상이 될 Primitive 리스트에서 시작 Index
	const int32 StartIndex			= Params.StartIndex;
	// ⭐
	
	// ⭐
	// Occlusion 컬링의 대상이 될 Primitive의 수.
	// View.PrimitiveVisibilityMap의 Element 개수와 일치한다.
	const int32 NumToProcess		= Params.NumToProcess;
	// ⭐

	const bool bSubmitQueries		= Params.bSubmitQueries;

	// ⭐
	// 모바일의 경우, 옵션을 킨 경우 HI-Z 버퍼를 가지고 Occlusion Test를 수행한다.
	// 테스트할 Fragment가 상대적으로 적어지니 빨라진다.
	const bool bHZBOcclusion		= Params.bHZBOcclusion;
	// ⭐

	const float PrimitiveProbablyVisibleTime = GEngine->PrimitiveProbablyVisibleTime;

	FSceneViewState* ViewState = (FSceneViewState*)View.State;
	
	// ⭐
	// 기본적으로 Occlusion Query는 Query를 발행한 후 3 프레임 후 그 결과값을 가시성 테스트에 사용한다.
	// 3 프레임 전의 Occlusion 가시성을 기준으로 현재 프레임에 렌더링 할 오브젝트를 결정한다는 것이다.
	// Occlusion Query가 오래걸리는 동작이다보니, 
	// 현재 프레임에 발행한 Query의 결과 값을 기다리느라 렌더 스레드를 Stall 시키기 보다는 몇 프레임 후 그 결과값을 가져오도록 구현을 한 것이다. 
	// 이렇게 구현이 되나보니 3 프레임 사이에 카메라의 Transform이 급격하게 변화하면 
	// Occlude 되지 말아야 할 Primitive가 3 프레임 전 결과를 기준으로 Occlude 되었다고 판단이 되어 렌더링을 하지 않는 불상사가 발생할 수 있다.
	// UE4는 이러한 경우에 대한 대비책도 가지고 있는 듯 보인다(?)
	//
	// 다른 플랫폼과 달리 모바일 플랫폼에서는 3 프레임이 아닌 4 프레임 전의 HW Occlusion Query 결과를 가져온다.
	// PC 플랫폼 같은 경우 GPU가 충분히 빨라 3 프레임 정도 지나면 Occlusion Query 연산이 끝나고 CPU로 Query 결과를 가져올 수 있지만,
	// GPU 연산량이 상대적으로 부족한 모바일의 경우에는 4 프레임 정도 충분한 텀을 두고 Occlusion Query 결과를 가져온다.
	const int32 NumBufferedFrames = FOcclusionQueryHelpers::GetNumBufferedFrames(Scene->GetFeatureLevel());
	// ⭐

	bool bClearQueries = !View.Family->EngineShowFlags.HitProxies;
	const float CurrentRealTime = View.Family->CurrentRealTime;
	uint32 OcclusionFrameCounter = ViewState->OcclusionFrameCounter;
	FRHIRenderQueryPool* OcclusionQueryPool = ViewState->OcclusionQueryPool;
	FHZBOcclusionTester& HZBOcclusionTests = ViewState->HZBOcclusionTests;

	int32 ReadBackLagTolerance = NumBufferedFrames;

	const bool bIsStereoView = IStereoRendering::IsStereoEyeView(View);
	const bool bUseRoundRobinOcclusion = bIsStereoView && !View.bIsSceneCapture && View.ViewState->IsRoundRobinEnabled();
	if (bUseRoundRobinOcclusion)
	{
		// ...
		// ... VR 관련 처리들
		// ...
	}
	// Round robin occlusion culling can make holes in the occlusion history which would require scanning the history when reading
	Params.bNeedsScanOnRead = bUseRoundRobinOcclusion;

	// ⭐
	// 예전 프레임에서의 Occlusion Culling 결과를 가져온다.
	// 예전 프레임에서의 결과를 활용한다.
	TSet<FPrimitiveOcclusionHistory, FPrimitiveOcclusionHistoryKeyFuncs>& ViewPrimitiveOcclusionHistory = ViewState->PrimitiveOcclusionHistorySet;
	// ⭐

	TArray<FPrimitiveOcclusionHistory>* InsertPrimitiveOcclusionHistory = Params.InsertPrimitiveOcclusionHistory;
	TArray<FPrimitiveOcclusionHistory*>* QueriesToRelease = Params.QueriesToRelease;
	TArray<FHZBBound>* HZBBoundsToAdd = Params.HZBBoundsToAdd;

	// ⭐
	TArray<FOcclusionBounds>* QueriesToAdd = Params.QueriesToAdd;	
	// ⭐

	const bool bNewlyConsideredBBoxExpandActive = GExpandNewlyOcclusionTestedBBoxesAmount > 0.0f && GFramesToExpandNewlyOcclusionTestedBBoxes > 0 && GFramesNotOcclusionTestedToExpandBBoxes > 0;

	// ⭐
	// Primitive의 Boundig Sphere의 Center와 ViewPort 사이의 거리가 이 값보다 작은 경우 절대 Occlusion Cull 하지 않는다.
	// 제곱을 사용하는 이유는 피타고라스 정리로 계산된 거리 값과 비교를 할 때 a^2 + b^2 = c^2에서 c 즉 거리의 값을 구하기 위해
	// 루트를 해야하는데 루트는 매우 비싼 연산이기 때문에, 차라리 GNeverOcclusionTestDistance을 제곱해서 c^2과 비교하는 것이다.
	// 비교시 두 변수가 양수인 것이 보장되면 두 변수의 제곱을 하여도 비교 연산의 결과 값은 동일하다는 속성을 이용한 것이다. 
	const float NeverOcclusionTestDistanceSquared = GNeverOcclusionTestDistance * GNeverOcclusionTestDistance;
	// ⭐

	// ⭐
	// ViewPort의 World Space 원점 값
	const FVector ViewOrigin = View.ViewMatrices.GetViewOrigin();
	// ⭐

	const int32 ReserveAmount = NumToProcess;
	if (!bSingleThreaded)
	{		
		check(InsertPrimitiveOcclusionHistory);
		check(QueriesToRelease);
		check(HZBBoundsToAdd);
		check(QueriesToAdd);

		//avoid doing reallocs as much as possible.  Unlikely to make an entry per processed element.		
		InsertPrimitiveOcclusionHistory->Reserve(ReserveAmount);
		QueriesToRelease->Reserve(ReserveAmount);
		HZBBoundsToAdd->Reserve(ReserveAmount);
		QueriesToAdd->Reserve(ReserveAmount);
	}
	
	int32 NumProcessed = 0;
	// ⭐
	int32 NumTotalPrims = View.PrimitiveVisibilityMap.Num();
	// ⭐
	int32 NumTotalDefUnoccluded = View.PrimitiveDefinitelyUnoccludedMap.Num();

	//if we are load balanced then we iterate only the set bits, and the ranges have been pre-selected to evenly distribute set bits among the tasks with no overlaps.
	//if not, then the entire array is evenly divided by range.
#if BALANCE_LOAD
	// ⭐
	// 여기서는 Occlusion 컬링을 싱글스레드에서 수행하는 것을 기준으로 하기 때문에 ( 엔진 기본 값임 ), 모든 Primtiive가 이 함수에서의 연산 대상이다.
	// 모든 Primitive를 순회한다.
	for (FSceneSetBitIterator BitIt(View.PrimitiveVisibilityMap, StartIndex); BitIt && (NumProcessed < NumToProcess); ++BitIt, ++NumProcessed)
	// ⭐
#else
	for (TBitArray<SceneRenderingBitArrayAllocator>::FIterator BitIt(View.PrimitiveVisibilityMap, StartIndex); BitIt && (NumProcessed < NumToProcess); ++BitIt, ++NumProcessed)
#endif
	{		
		uint8 OcclusionFlags = Scene->PrimitiveOcclusionFlags[BitIt.GetIndex()];
	
		// ⭐
		// Primitive가 Occlusion 컬링의 대상이 될 수 있는지 확인한다.
		// Occluder에 의해서 가려진 경우 컬링 처리할지를 말하는 것이다.
		// Primitive Component의 옵션을 보면 "CanBeOccluded"라는 옵션이 있다. 그것이 이것이다.
		bool bCanBeOccluded = (OcclusionFlags & EOcclusionFlags::CanBeOccluded) != 0;
		// ⭐

#if !BALANCE_LOAD		
		if (!View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt))
		{
			continue;
		}
#endif

		//we can't allow the prim history insertion array to realloc or it will invalidate pointers in the other output arrays.
		const bool bCanAllocPrimHistory = bSingleThreaded /* 기본 값은 true */ || InsertPrimitiveOcclusionHistory->Num() < InsertPrimitiveOcclusionHistory->Max();		

		if (GIsEditor)
		{
			// ...
			// ... 에디터 관련 처리
			// ... 
		}
		int32 NumSubQueries = 1;
		bool bSubQueries = false;
		const TArray<FBoxSphereBounds>* SubBounds = nullptr;

		// ⭐
		// Sub Primitive들을 각각 Query 할지 여부를 결정한다.
		// 현재는 Hierarchical Instanced Static Mesh에서만 사용된다.
		// Hierarchical Instanced Static Mesh는 인스턴스 Mesh Draw에서 각 Mesh에 대한 LOD 연산이 추가된 개념이다.
		// 일반 Instanced Static Mesh의 경우 각 Mesh들의 LOD가 하나로 통일된다.
		// 자세한건 오른쪽 링크를 참고하세요. ( https://forums.unrealengine.com/t/difference-between-instanced-static-mesh-component-and-hierarchical-instanced-static-mesh-component/308142/3 )
		//
		// Instanced된 Primitive들은 하나의 FPrimitiveSceneInfo에 합쳐저 들어 있기 때문에 SubQuery라는 개념을 두는 것 같다 ( ?, 확인 필요 )
		//
		// 기본 값은 1이다.
		// 만약 Disabled 되어 있다면 Hierarchical Instanced static Mesh 들을 그냥 하나의 Query로 통으로 처리한다.
		// 결국 Instanced된 Mesh들의 Query 연산할 Primitive가 매우 ( 화면상에서 차지할 넓이가 ) 커지게 될 가능서잉 높아지고, Occlusion Cull될 가능성이 낮아진다. 
		// ( 왜냐면 크기가 매우 크니, 그만큼 다른 오브젝트에 의해 가려질 가능성도 낮아지기 때문이다. )
		//
		// Primitive의 Sub Query
		check(Params.SubIsOccluded);
		TArray<bool>& SubIsOccluded = *Params.SubIsOccluded;
		int32 SubIsOccludedStart = SubIsOccluded.Num();
		// ⭐
		// Hierarchical Instanced Static Mesh가 아닌 일반적인 Primitive는 EOcclusionFlags::HasSubprimitiveQueries Flag를 가지고 있지 않다.
		if ((OcclusionFlags & EOcclusionFlags::HasSubprimitiveQueries) && GAllowSubPrimitiveQueries && !View.bDisableQuerySubmissions)
		// ⭐
		{
			FPrimitiveSceneProxy* Proxy = Scene->Primitives[BitIt.GetIndex()]->Proxy;

			// ⭐
			// Instanced Static Mesh 처리된 Prmitive의 Sub Query를 가져온다. SubQuery를 AABB 바운딩 박스이다.
			SubBounds = Proxy->GetOcclusionQueries(&View);
			// ⭐

			NumSubQueries = SubBounds->Num();
			bSubQueries = true;
			if (!NumSubQueries)
			{
				View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
				continue;
			}
			SubIsOccluded.Reserve(NumSubQueries);
		}
		// ⭐

		bool bAllSubOcclusionStateIsDefinite = true;
		bool bAllSubOccluded = true;
		FPrimitiveComponentId PrimitiveId = Scene->PrimitiveComponentIds[BitIt.GetIndex()];

		
		// ⭐
		// Primitive가 Hierarchical Instanced static Mesh가 아닌 경우 NumSubQueries는 1이다.
		for (int32 SubQuery = 0; SubQuery < NumSubQueries; SubQuery++)
		{
		// ⭐
			
			// ⭐
			// Primitive에 대한 과거 Occlusion Culling 결과를 가져온다.
			// 위에서 말한 것처럼 Occlusion Query는 매우 느려터진 동작이기 때문에,
			// 3 ~ 4 프레임 전에 발행한 Query의 결과 값을 사용한다.
			//
			// PrimitiveOcclusionHistory가 있다는 것은 Primitive에 대해 과거에 Occlusion Query를 수행한 적이 있다는 것이다.
			FPrimitiveOcclusionHistory* PrimitiveOcclusionHistory = ViewPrimitiveOcclusionHistory.Find(FPrimitiveOcclusionHistoryKey(PrimitiveId, SubQuery));
			// ⭐

			bool bIsOccluded = false;
			bool bOcclusionStateIsDefinite = false;	
					
			// ⭐
			// Primitive에 대한 과거의 Occlusion Culling 결과 값 데이터가 없는 경우,
			// Primitive에 대한 Occlusion Culling을 처음 수행하는 경우....
			if (!PrimitiveOcclusionHistory)
			// ⭐
			{
				

				// If the primitive doesn't have an occlusion history yet, create it.
				if (bSingleThreaded /* 기본 값은 true */ )
				{					
					// ⭐
					// ViewPrimitiveOcclusionHistory를 만든다.
					//
					// In singlethreaded mode we can safely modify the view's history directly.
					PrimitiveOcclusionHistory = &ViewPrimitiveOcclusionHistory[
						ViewPrimitiveOcclusionHistory.Add(FPrimitiveOcclusionHistory(PrimitiveId, SubQuery))
					];
					// ⭐
				}
				else if (bCanAllocPrimHistory)
				{
					// In multithreaded mode we have to buffer the new histories and add them to the view during a post-combine
					PrimitiveOcclusionHistory = &(*InsertPrimitiveOcclusionHistory)[
						InsertPrimitiveOcclusionHistory->Add(FPrimitiveOcclusionHistory(PrimitiveId, SubQuery))
					];
				}				
				
				// If the primitive hasn't been visible recently enough to have a history, treat it as unoccluded this frame so it will be rendered as an occluder and its true occlusion state can be determined.
				// already set bIsOccluded = false;

				// Flag the primitive's occlusion state as indefinite, which will force it to be queried this frame.
				// The exception is if the primitive isn't occludable, in which case we know that it's definitely unoccluded.
				bOcclusionStateIsDefinite = bCanBeOccluded ? false : true;
			}

			// ⭐
			// Primitive에 대한 과거의 Occlusion Culling 결과 값 데이터가 있는 경우
			//
			else
			// ⭐
			{
				if (View.bIgnoreExistingQueries)
				{
					// If the view is ignoring occlusion queries, the primitive is definitely unoccluded.
					// already set bIsOccluded = false;
					bOcclusionStateIsDefinite = View.bDisableQuerySubmissions;
				}
				// ⭐
				// Primitive Component에서 "CanBeOccluded" 옵션을 Enable한 경우
				else if (bCanBeOccluded)
				// ⭐
				{
					if (bHZBOcclusion)
					{
						if (HZBOcclusionTests.IsValidFrame(PrimitiveOcclusionHistory->LastTestFrameNumber))
						{
							bIsOccluded = !HZBOcclusionTests.IsVisible(PrimitiveOcclusionHistory->HZBTestIndex);
							bOcclusionStateIsDefinite = true;
						}
					}
					else
					{
						// Read the occlusion query results.
						uint64 NumSamples = 0;
						bool bGrouped = false;
						
						// ⭐
						// 과거 프레임 ( 3~4 프레임 전 )의 Occlusion Query 오브젝트 ( FRHIRenderQuery )를 가져온다.
						FRHIRenderQuery* PastQuery = PrimitiveOcclusionHistory->GetQueryForReading(OcclusionFrameCounter, NumBufferedFrames, ReadBackLagTolerance, bGrouped);
						// ⭐
						
						// ⭐
						// PrimitiveOcclusionHistory에 FRHIRenderQuery 데이터가 있는 경우
						if (PastQuery)
						// ⭐
						{
							// ⭐⭐⭐⭐⭐⭐⭐
							// int32 RefCount = PastQuery.GetReference()->GetRefCount();
							// NOTE: RHIGetOcclusionQueryResult should never fail when using a blocking call, rendering artifacts may show up.
							// if (RHICmdList.GetRenderQueryResult(PastQuery, NumSamples, true))
							//
							// FRHIRenderQuery로부터 Occlusion Query 결과를 가져온다.
							if (GDynamicRHI->RHIGetRenderQueryResult(PastQuery, NumSamples, true))
							// ⭐⭐⭐⭐⭐⭐⭐
							{
								// ⭐
								// we render occlusion without MSAA
								//
								// Occludee의 렌더링 중 몇개의 Fragment가 Depth Test를 통과했는지
								uint32 NumPixels = (uint32)NumSamples;
								// ⭐

								// ⭐⭐⭐⭐⭐⭐⭐
								// The primitive is occluded if none of its bounding box's pixels were visible in the previous frame's occlusion query.
								//
								// Occludee의 Query 중 하나의 Fragment도 Depth Test를 통과하지 못한 경우, 최종적으로 Occlude ( Culled ) 되었다고 판단한다.
								bIsOccluded = (NumPixels == 0);
								// ⭐⭐⭐⭐⭐⭐⭐
								
								// ⭐
								// 화면의 전체 Fragment 중 몇개의 Fragment가 Depth Test를 통과했는지 기록한다.
								if (!bIsOccluded)
								{
									checkSlow(View.OneOverNumPossiblePixels > 0.0f);
									PrimitiveOcclusionHistory->LastPixelsPercentage = NumPixels * View.OneOverNumPossiblePixels;
								}
								else
								{
									PrimitiveOcclusionHistory->LastPixelsPercentage = 0.0f;
								}
								// ⭐								

								// Flag the primitive's occlusion state as definite if it wasn't grouped.
								bOcclusionStateIsDefinite = !bGrouped;
							}
							else
							{
								// If the occlusion query failed, treat the primitive as visible.  
								// already set bIsOccluded = false;
							}
						}

						// ⭐
						// PrimitiveOcclusionHistory에 FRHIRenderQuery 데이터가 없는 경우,
						// Primitive가 Occlude 되었었는지 여부만 가져온다.
						else
						// ⭐
						{
							if 
							(
								NumBufferedFrames > 1 || 

								// Occlusion Query 최대 가능 발행 횟수, 기본 값은 MAX_int32. 
								// 안드로이드 플랫폼에서 OpenGL을 사용하는 경우 MAX_int32보다 작을 수 있다.
								GRHIMaximumReccommendedOustandingOcclusionQueries < MAX_int32 
							)
							{
								// If there's no occlusion query for the primitive, assume it is whatever it was last frame
								bIsOccluded = PrimitiveOcclusionHistory->WasOccludedLastFrame;
								bOcclusionStateIsDefinite = PrimitiveOcclusionHistory->OcclusionStateWasDefiniteLastFrame;
							}
							else
							{
								// If there's no occlusion query for the primitive, set it's visibility state to whether it has been unoccluded recently.
								bIsOccluded = (PrimitiveOcclusionHistory->LastProvenVisibleTime + GEngine->PrimitiveProbablyVisibleTime < CurrentRealTime);
								// the state was definite last frame, otherwise we would have ran a query
								bOcclusionStateIsDefinite = true;
							}
							if (bIsOccluded)
							{
								PrimitiveOcclusionHistory->LastPixelsPercentage = 0.0f;
							}
							else
							{
								PrimitiveOcclusionHistory->LastPixelsPercentage = GEngine->MaxOcclusionPixelsFraction;
							}
						}
					}

					// ...
					// ... 디버깅 관련 코드
					// ...
				}
				else
				{					
					// Primitives that aren't occludable are considered definitely unoccluded.
					// already set bIsOccluded = false;
					bOcclusionStateIsDefinite = true;
				}

				if (bClearQueries)
				{					
					if (bSingleThreaded)
					{						
						PrimitiveOcclusionHistory->ReleaseQuery(OcclusionFrameCounter, NumBufferedFrames);
					}
					else
					{
						if (PrimitiveOcclusionHistory->GetQueryForEviction(OcclusionFrameCounter, NumBufferedFrames))
						{
							QueriesToRelease->Add(PrimitiveOcclusionHistory);							
						}
					}
				}
			}

			if (PrimitiveOcclusionHistory)
			{
				if 
				(
					bSubmitQueries /* 이번 프레임에 Primitive에 대한 Occlusion Query를 발행하는 경우 true */ && 
					bCanBeOccluded 
				)
				{
					bool bSkipNewlyConsidered = false;

					if (bNewlyConsideredBBoxExpandActive /* 기본 값은 false */ )
					{
						if (!PrimitiveOcclusionHistory->BecameEligibleForQueryCooldown && OcclusionFrameCounter - PrimitiveOcclusionHistory->LastConsideredFrameNumber > uint32(GFramesNotOcclusionTestedToExpandBBoxes))
						{
							PrimitiveOcclusionHistory->BecameEligibleForQueryCooldown = GFramesToExpandNewlyOcclusionTestedBBoxes;
						}

						bSkipNewlyConsidered = !!PrimitiveOcclusionHistory->BecameEligibleForQueryCooldown;

						if (bSkipNewlyConsidered)
						{
							PrimitiveOcclusionHistory->BecameEligibleForQueryCooldown--;
						}
					}


					bool bAllowBoundsTest;

					// ⭐
					// Occlusion Query에 사용될 Occludee Primitive의 AABB 바운딩 박스 데이터를 가져온다.
					const FBoxSphereBounds OcclusionBounds = (bSubQueries ? (*SubBounds)[SubQuery] : Scene->PrimitiveOcclusionBounds[BitIt.GetIndex()]).ExpandBy(GExpandAllTestedBBoxesAmount + (bSkipNewlyConsidered ? GExpandNewlyOcclusionTestedBBoxesAmount : 0.0));
					// ⭐

					// ⭐
					// Primitive에 대해 Occlusion Query Test를 수행할지 결정한다.
					// Occlusion Query 수를 최대한 줄여보기 위한 노력이다.
					// Occludion Query도 결국 하나 하나가 Draw Call이니,
					// 어차피 Occlude 되지 않을 것 같은 Primitive는 아예 Query Test를 하지 않는 것도 좋은 방법이다.
					//
					// bAllowBoundsTest이 true다 -> Occlusion Query Test를 수행한다.
					
					// ⭐
					// Primitive가 카메라에 너무 가까운 경우 ->
					// Occlude 되지 않을 가능성이 매우 높다.
					// 그럼 그냥 Occlusion Test를 안한다!
					if (FVector::DistSquared(ViewOrigin, OcclusionBounds.Origin) < NeverOcclusionTestDistanceSquared)
					// ⭐
					{
						bAllowBoundsTest = false;
					}
					else if (View.bHasNearClippingPlane)
					{
						bAllowBoundsTest = View.NearClippingPlane.PlaneDot(OcclusionBounds.Origin) <
							-(FVector::BoxPushOut(View.NearClippingPlane, OcclusionBounds.BoxExtent));

					}
					else if (!View.IsPerspectiveProjection())
					{
						// Transform parallel near plane
						static_assert((int32)ERHIZBuffer::IsInverted != 0, "Check equation for culling!");
						bAllowBoundsTest = View.WorldToScreen(OcclusionBounds.Origin).Z - View.ViewMatrices.GetProjectionMatrix().M[2][2] * OcclusionBounds.SphereRadius < 1;
					}
					else
					{
						bAllowBoundsTest = OcclusionBounds.SphereRadius < HALF_WORLD_MAX;
					}
					// ⭐

					// ⭐
					// bAllowBoundsTest이 true다 -> Occlusion Query Test를 수행한다.
					if (bAllowBoundsTest)
					// ⭐
					{
						PrimitiveOcclusionHistory->LastTestFrameNumber = OcclusionFrameCounter;
						if (bHZBOcclusion)
						{
							// Always run
							if (bSingleThreaded)
							{								
								PrimitiveOcclusionHistory->HZBTestIndex = HZBOcclusionTests.AddBounds(OcclusionBounds.Origin, OcclusionBounds.BoxExtent);
							}
							else
							{
								HZBBoundsToAdd->Emplace(PrimitiveOcclusionHistory, OcclusionBounds.Origin, OcclusionBounds.BoxExtent);
							}
						}
						else
						{
							// decide if a query should be run this frame
							bool bRunQuery, bGroupedQuery;
							
							// ⭐
							// 여러 Occlusion Query를 하나로 합쳐서 Query 테스트에 필요한 Draw Call을 줄여볼지 결정한다. 
							if (!bSubQueries && // sub queries are never grouped, we assume the custom code knows what it is doing and will group internally if it wants
								(OcclusionFlags & EOcclusionFlags::AllowApproximateOcclusion))
							{
							// ⭐

								if (bIsOccluded)
								{
									// ⭐
									// 이전 프레임 ( 3 ~ 4 프레임 전 )에 Occlude 되었었다면, 
									// 이번 프레임에도 Query를 실행하고, 여러 Query를 하나로 합친다.
									//
									// Primitives that were occluded the previous frame use grouped queries.
									bGroupedQuery = true;
									bRunQuery = true;
									// ⭐
								}
								else if (bOcclusionStateIsDefinite)
								{
									bGroupedQuery = false;
									float Rnd = GOcclusionRandomStream.GetFraction();
									if (GRHISupportsExactOcclusionQueries)
									{
										float FractionMultiplier = FMath::Max(PrimitiveOcclusionHistory->LastPixelsPercentage / GEngine->MaxOcclusionPixelsFraction, 1.0f);
										bRunQuery = (FractionMultiplier * Rnd) < GEngine->MaxOcclusionPixelsFraction;
									}
									else
									{
										bRunQuery = CurrentRealTime - PrimitiveOcclusionHistory->LastProvenVisibleTime > PrimitiveProbablyVisibleTime * (0.5f * 0.25f * Rnd);
									}									
								}
								else
								{
									bGroupedQuery = false;
									bRunQuery = true;
								}
							}
							else
							{
								// ⭐
								// Primitives that need precise occlusion results use individual queries.
								bGroupedQuery = false;
								bRunQuery = true;
								// ⭐
							}

							// ⭐
							if (bRunQuery /* 이번 프레임에 Occlusion Query Test를 수행하는 경우 */ )
							// ⭐
							{
								const FVector BoundOrigin = OcclusionBounds.Origin + View.ViewMatrices.GetPreViewTranslation();
								const FVector BoundExtent = OcclusionBounds.BoxExtent;

								if (bSingleThreaded /* 기본 값은 true */ )
								{
									checkSlow(DynamicVertexBufferIfSingleThreaded);

									if 
									(
										// Occlusion Query 최대 가능 발행 횟수, 기본 값은 MAX_int32. 
										// 안드로이드 플랫폼에서 OpenGL을 사용하는 경우 MAX_int32보다 작을 수 있다.
										GRHIMaximumReccommendedOustandingOcclusionQueries < MAX_int32 && 
										!bGroupedQuery
									)
									{
										// ⭐
										// 추가될 QueriesToAdd를 넣는다.
										QueriesToAdd->Emplace(FPrimitiveOcclusionHistoryKey(PrimitiveId, SubQuery), BoundOrigin, BoundExtent, PrimitiveOcclusionHistory->LastQuerySubmitFrame());
										// ⭐
									}
									else
									{
										// ⭐⭐⭐⭐⭐⭐⭐
										// PrimitiveOcclusionHistory에 Query를 셋팅한다.
										// SetCurrentQuery 함수에 두번째 매개변수로 들어갈 FRHIRenderQuery가 PrimitiveOcclusionHistory에 저장된다.
										// 이후 위에서 보아듯이 PrimitiveOcclusionHistory를 통해 FRHIRenderQuery의 결과를 얻을 수 있다.
										//
										// 여기서 설정한 Query는 이후에 FMobileSceneRenderer::RenderOcclusion 함수에서 Query가 발행될 때 사용된다.
										PrimitiveOcclusionHistory->SetCurrentQuery(OcclusionFrameCounter,

											// ⭐
											// Query ( FRHIRenderQuery )를 만들어서,
											// GroupedOcclusionQueries 혹은 IndividualOcclusionQueries에 저장한다.
											// 또한 그 Query의 결과를 나중에 확인할 수 있게 PrimitiveOcclusionHistory에 저장한다.
											//
											bGroupedQuery ?
											// ⭐
											// Query를 하나로 합치는 경우 ( Occlusion Query Test의 Draw Call을 최대한 줄이기 위해 )
											View.GroupedOcclusionQueries.BatchPrimitive(BoundOrigin, BoundExtent, *DynamicVertexBufferIfSingleThreaded) :
											// ⭐
											View.IndividualOcclusionQueries.BatchPrimitive(BoundOrigin, BoundExtent, *DynamicVertexBufferIfSingleThreaded),
											//
											// ⭐

											NumBufferedFrames,
											bGroupedQuery,
											Params.bNeedsScanOnRead
										);
										// ⭐⭐⭐⭐⭐⭐⭐
									}
								}
								else
								{
									// ...
									// ...
									// ...
								}
							}
						}
					}
					else
					{
						// If the primitive's bounding box intersects the near clipping plane, treat it as definitely unoccluded.
						bIsOccluded = false;
						bOcclusionStateIsDefinite = true;
					}
				}

				// ⭐
				// PrimitiveOcclusionHistory에 Occlusion Query 결과를 저장한다. ( 물론 3 ~ 4 프레임 전의 Query의 결과이다 )
				// 위에서 보아듯이 이후 프레임에서 사용될 것이다.
				//
				// Set the primitive's considered time to keep its occlusion history from being trimmed.
				PrimitiveOcclusionHistory->LastConsideredTime = CurrentRealTime;
				if (!bIsOccluded && bOcclusionStateIsDefinite)
				{
					PrimitiveOcclusionHistory->LastProvenVisibleTime = CurrentRealTime;
				}
				PrimitiveOcclusionHistory->LastConsideredFrameNumber = OcclusionFrameCounter;
				PrimitiveOcclusionHistory->WasOccludedLastFrame = bIsOccluded;
				PrimitiveOcclusionHistory->OcclusionStateWasDefiniteLastFrame = bOcclusionStateIsDefinite;
				// ⭐

			}

			if (bSubQueries)
			{
				SubIsOccluded.Add(bIsOccluded);
				if (!bIsOccluded)
				{
					bAllSubOccluded = false;
				}
				if (bIsOccluded || !bOcclusionStateIsDefinite)
				{
					bAllSubOcclusionStateIsDefinite = false;
				}
			}
			else
			{
					
				// ⭐⭐⭐⭐⭐⭐⭐	
				// Occlusion Query 결과를 바탕으로 PrimitiveVisibilityMap에 가시성 여부를 저장한다.
				if (bIsOccluded)
				{
					View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
					STAT(NumOccludedPrimitives++);
				}
				else if (bOcclusionStateIsDefinite)
				{
					View.PrimitiveDefinitelyUnoccludedMap.AccessCorrespondingBit(BitIt) = true;
				}					
				// ⭐⭐⭐⭐⭐⭐⭐
			}			
		}

		if (bSubQueries)
		{
			if (SubIsOccluded.Num() > 0)
			{
				FPrimitiveSceneProxy* Proxy = Scene->Primitives[BitIt.GetIndex()]->Proxy;
				Proxy->AcceptOcclusionResults(&View, &SubIsOccluded, SubIsOccludedStart, SubIsOccluded.Num() - SubIsOccludedStart);
			}

			// ⭐⭐⭐⭐⭐⭐⭐	
			// Occlusion Query 결과를 바탕으로 PrimitiveVisibilityMap에 가시성 여부를 저장한다.
			if (bAllSubOccluded)
			{
				View.PrimitiveVisibilityMap.AccessCorrespondingBit(BitIt) = false;
				STAT(NumOccludedPrimitives++);
			}
			else if (bAllSubOcclusionStateIsDefinite)
			{
				View.PrimitiveDefinitelyUnoccludedMap.AccessCorrespondingBit(BitIt) = true;
			}
			// ⭐⭐⭐⭐⭐⭐⭐	

		}
	}

	check(NumTotalDefUnoccluded == View.PrimitiveDefinitelyUnoccludedMap.Num());
	check(NumTotalPrims == View.PrimitiveVisibilityMap.Num());
	check(!InsertPrimitiveOcclusionHistory || InsertPrimitiveOcclusionHistory->Num() <= ReserveAmount);

	// ⭐
	Params.NumOccludedPrims = NumOccludedPrimitives;	
	// ⭐
}
```

```cpp
// ⭐⭐⭐⭐⭐⭐⭐
// 새로운 FRHIRenderQuery를 추가한다.
// FViewInfo::GroupedOcclusionQueries ( FOcclusionQueryBatcher 타입), 
// FViewInfo::IndividualOcclusionQueries ( FOcclusionQueryBatcher 타입 )에 추가된다.
// 그럼 이후 FMobileSceneRenderer::RenderForward 함수에서 GPU로 보내진다.
//
// Query시에는 Primitive의 AABB를 가지고 수행한다.
// 원본 Primitive 데이터를 가지고 Query를 수행하는 것은 매우 비싸고 느리다.
// 그러니 AABB를 가지고 Query를 수행하는 것이다.
// 정확도는 떨어지지만 ( 더 보수적으로 컬링 -> 안 그려져도 되는 것이, 그려질 수 있다. 다만 그려져야 할 것이 안 그려지는 경우는 발생하지 않는다. ),          
// Query 처리 속도는 높이는, 
// 일종의 Trade Off이다.
FRHIRenderQuery* FOcclusionQueryBatcher::BatchPrimitive(const FVector& BoundsOrigin,const FVector& BoundsBoxExtent, FGlobalDynamicVertexBuffer& DynamicVertexBuffer)
// ⭐⭐⭐⭐⭐⭐⭐
{
	// Check if the current batch is full.
	if(CurrentBatchOcclusionQuery == NULL || NumBatchedPrimitives >= MaxBatchedPrimitives)
	{
		check(OcclusionQueryPool);
		CurrentBatchOcclusionQuery = new(BatchOcclusionQueries) FOcclusionBatch;
		CurrentBatchOcclusionQuery->Query = OcclusionQueryPool->AllocateQuery();
		CurrentBatchOcclusionQuery->VertexAllocation = DynamicVertexBuffer.Allocate(MaxBatchedPrimitives * 8 * sizeof(FVector));
		check(CurrentBatchOcclusionQuery->VertexAllocation.IsValid());
		NumBatchedPrimitives = 0;
	}

	// Add the primitive's bounding box to the current batch's vertex buffer.
	const FVector PrimitiveBoxMin = BoundsOrigin - BoundsBoxExtent;
	const FVector PrimitiveBoxMax = BoundsOrigin + BoundsBoxExtent;
	float* RESTRICT Vertices = (float*)CurrentBatchOcclusionQuery->VertexAllocation.Buffer;
	Vertices[ 0] = PrimitiveBoxMin.X; Vertices[ 1] = PrimitiveBoxMin.Y; Vertices[ 2] = PrimitiveBoxMin.Z;
	Vertices[ 3] = PrimitiveBoxMin.X; Vertices[ 4] = PrimitiveBoxMin.Y; Vertices[ 5] = PrimitiveBoxMax.Z;
	Vertices[ 6] = PrimitiveBoxMin.X; Vertices[ 7] = PrimitiveBoxMax.Y; Vertices[ 8] = PrimitiveBoxMin.Z;
	Vertices[ 9] = PrimitiveBoxMin.X; Vertices[10] = PrimitiveBoxMax.Y; Vertices[11] = PrimitiveBoxMax.Z;
	Vertices[12] = PrimitiveBoxMax.X; Vertices[13] = PrimitiveBoxMin.Y; Vertices[14] = PrimitiveBoxMin.Z;
	Vertices[15] = PrimitiveBoxMax.X; Vertices[16] = PrimitiveBoxMin.Y; Vertices[17] = PrimitiveBoxMax.Z;
	Vertices[18] = PrimitiveBoxMax.X; Vertices[19] = PrimitiveBoxMax.Y; Vertices[20] = PrimitiveBoxMin.Z;
	Vertices[21] = PrimitiveBoxMax.X; Vertices[22] = PrimitiveBoxMax.Y; Vertices[23] = PrimitiveBoxMax.Z;

	// Bump the batches buffer pointer.
	Vertices += 24;
	CurrentBatchOcclusionQuery->VertexAllocation.Buffer = (uint8*)Vertices;
	NumBatchedPrimitives++;

	return CurrentBatchOcclusionQuery->Query;
}
```