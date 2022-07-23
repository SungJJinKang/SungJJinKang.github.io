---
layout: post
title:  "UE4 렌더링 최적화 관련"
date:   2022-07-23
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---         
                
[Shadow Pass 전용 LOD Bias](https://www.reddit.com/r/unrealengine/comments/7stjf7/show_offshadow_lod_biasing/)                   
-> ForceLODShadow이라는 기능이 있지만 이 기능은 특정 LOD Level을 모든 Mesh에 일괄 적용하는 방식이라 가까운 Mesh의 Shadow Pass에서의 비주얼 퀄리티가 나빠질 수도 있다.                                          
                  
[Unreal’s Rendering Passes](https://unrealartoptimization.github.io/book/profiling/passes/)             
[NEW STATE Mobile reduces GPU usage by 22% with Android GPU Inspector](https://developer.android.com/stories/games/new-state-mobile)                   