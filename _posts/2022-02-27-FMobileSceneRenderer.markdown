---
layout: post
title:  "Unreal Engine4 FMobileSceneRenderer 분석"
date:   2022-02-26
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---

**FMobileSceneRenderer**는 언리얼 엔진4의 모바일용 그래픽스 파이프라인이다.                
이 글은 FMobileSceneRenderer::Render 함수를 위주로 FMobileSceneRenderer의 소스 코드를 분석할 것이다.        
언리얼 엔진 정책상 아마 모든 소스 코드를 담지는 못할 것 같다.              

------------------------------         

references : [Graphics Programming Overview](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/Overview/), [Rendering Dependency Graph](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/RenderDependencyGraph/), [Threaded Rendering](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ThreadedRendering/), [Mesh Drawing Pipeline](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/MeshDrawingPipeline/)             
