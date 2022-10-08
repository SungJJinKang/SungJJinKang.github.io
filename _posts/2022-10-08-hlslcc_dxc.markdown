---
layout: post
title:  "언리얼 엔진 쉐이더 컴파일 도구 ( hlslcc, DXC ) (작성중)"
date:   2022-10-09
categories: ComputerScience ComputerGraphics UE4
---         
              
최근에 UE5.1에 올라간 모바일용 AMD FSR을 우리팀 Inhouse UE4.27로 백포팅하는 작업을 하고 있는데 언리쪽에서 UE4에 대한 지원을 안하다 보니 쉐이더 컴파일부터 문제가 생겼다.       
OpenGL ES 사용시 DXC 컴파일러를 요구하여 ForceDXC 옵션을 켜서 쉐이더 컴파일을 하면 "Push constant block cannot be expressed as neither std430 nor std140. ES-targets do not support GL_ARB_enhanced_layouts."라는 에러를 뿜어내고, 기존의 hlslcc 컴파일러 사용시 "min16float2"와 같은 명시적 Half precision floating-point 타입의 사용에 대해 에러를 뿜어냈다. 현재 리서치 중이라서 자세한 내용은 모른다.           
어쨌든 이 쪽 코드는 처음 보는거라서 최근에서야 hlslcc, DXC라는 쉐이더 컴파일러의 존재도 알았다.     
그래서 이에 관해 간단한 분석 글을 써보려한다.             
           
-----------------------------------------------------------------
                   



                    


references : [https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/hlslcc/hlslcc], [https://github.com/microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler), [https://www.slideshare.net/cagetu/gdc-14-bringing-unreal-engine-4-to-opengl](https://www.slideshare.net/cagetu/gdc-14-bringing-unreal-engine-4-to-opengl), [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ShaderDevelopment/HLSLCrossCompiler/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ShaderDevelopment/HLSLCrossCompiler/), [https://github.com/KhronosGroup/SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) , [https://therealmjp.github.io/posts/shader-fp16/](https://therealmjp.github.io/posts/shader-fp16/)           