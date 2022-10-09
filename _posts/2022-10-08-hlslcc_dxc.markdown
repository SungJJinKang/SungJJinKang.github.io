---
layout: post
title:  "언리얼 엔진 쉐이더 변환 도구 ( hlslcc, ShaderConductor )"
date:   2022-10-09
categories: ComputerScience ComputerGraphics UE4
---         
               
최근에 UE5.1에 올라간 모바일용 AMD FSR을 우리팀 Inhouse UE4.27로 백포팅하는 작업을 하고 있는데 언리얼쪽에서 UE4에 대한 지원을 안하다 보니 쉐이더 컴파일부터 문제가 생겼다.        
OpenGL ES 사용시 DXC 컴파일러를 요구하여 ForceDXC 옵션을 켜서 쉐이더 컴파일을 하면 "Push constant block cannot be expressed as neither std430 nor std140. ES-targets do not support GL_ARB_enhanced_layouts."라는 에러를 뿜어내고, 기존의 hlslcc 컴파일러 사용시 "min16float2"와 같은 명시적 Half precision floating-point 타입의 사용에 대해 에러를 뿜어냈다. 현재 리서치 중이라서 자세한 내용은 모른다.                               
어쨌든 이 쪽 코드는 처음 보는거라서 최근에서야 hlslcc, ShaderConductor 것의 존재도 알았다.                 
그래서 이에 관해 간단한 분석 글을 써보려한다.              
           
-----------------------------------------------------------------
                    
언리얼 엔진은 기본적으로 hlsl ( D3D의 쉐이더 언어 )로 작성된 우버 쉐이더 코드를 필요에 따라 여러 전처리 매크로들을 On, Off하여 사용한다. 여기서 우버 쉐이더란 필요한 기능이 전부 들어 있는 하나의 쉐이더 코드로 필요한 기능을 하나의 쉐이더 코드에 다 때려 넣고 전처리 매크로로 필요한 기능한 포함시켜 컴파일하는 것을 의미한다. ( 자세한건 [여기](https://www.slideshare.net/blindrendererkr/3ds-max-26700918)를 참고하시길 바랍니다... )                
                 
쉐이더 클래스를 보면 ModifyCompilationEnvironment 함수를 통해서 앞서 말한 전처리 매크로를 ON/OFF하는 것을 볼 수 있다.             
<img width="700" alt="FSR" src="https://user-images.githubusercontent.com/33873804/194740058-1f3d51db-9bf6-4dad-97ff-7d0b4d50e9ab.png">             
<img width="621" alt="FSR2" src="https://user-images.githubusercontent.com/33873804/194740061-57cf053f-2006-403e-b01b-05990818da61.png">                 
                  
그리고 우버 쉐이더 코드에서 나오는 수 많은 Permutation ( 전처리 매크로에 따라 분기되는 수 많은 쉐이더 코드 )들 중 어떤 분기를 실제로 컴파일해서 사용할지를 결정하는 코드도 볼 수 있다.         
<img width="697" alt="Shader" src="https://user-images.githubusercontent.com/33873804/194740197-e2e416d6-7b1e-4ae2-b1d3-87159b50b553.png">                 
<img width="386" alt="Shader2" src="https://user-images.githubusercontent.com/33873804/194740200-1238aa24-4235-4f93-8fdc-2b9dbb65763f.png">                 
<img width="648" alt="Shader3" src="https://user-images.githubusercontent.com/33873804/194740202-901832f3-42b9-4ef2-8fc1-133694b7b5f1.png">                    
            
-----------------------------             
           
이렇게 만들어진 최종적인 hlsl 쉐이더 코드를 이제는 목표로 하는 타깃 그래픽 API의 쉐이더 코드로 변환하여야 한다.          
언리얼 엔진에서는 크게 두 가지 방법을 사용한다. [hlslcc](https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/hlslcc/hlslcc)를 사용하는 방법과 [ShaderConductor](https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/ShaderConductor/ShaderConductor)를 사용하는 방법이다.               
hlslcc를 사용하는 방법은 현재 UE5에서는 대부분의 그래픽 API에서 deprecated되었고 UE5에서는 대부분 ShaderConductor를 활용하는 방법을 사용한다.            
                
아래의 사진이 hlslcc를 통한 방법, ShaderConductor를 통한 방법에서 어떻게 hlsl로 작성된 쉐이더 코드가 타깃 쉐이더 언어로 변환되는지 잘 보여준다.            
![hlslcc,dxc](https://user-images.githubusercontent.com/33873804/194741515-ee88360f-2eb7-4341-8ca6-7387fa9ceee6.jpg)                
                    
hlslcc를 이용한 경우에는 hlsl로 작성된 쉐이더 코드를 hlslcc를 통해 Mesa IR(중간 언어)로 변환하고 이 변환된 IR를 최적화하는 과정을 거쳐서 최종적으로 타깃하는 쉐이더 언어로 변환된다.                       
                 
ShaderConductor를 활용하는 방법의 경우 언리얼 엔진에서는 이 방법은 [Shader Conductor](https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/ShaderConductor/ShaderConductor)라는 모듈로 관리하는데 Shader Conductor에서 [DXC 컴파일러](https://github.com/microsoft/DirectXShaderCompiler)를 사용하여 hlsl 쉐이더 코드를 SPIRV라는 중간 언어로 변환한 후 SPIRV에 대해 [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools)를 활용해 SPIRV를 최적화한 후 [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross)를 사용하여 목표로 하는 쉐이더 언어로 변환한다.            
           
쉐이더 코드를 모든 쉐이더 언어별로 전부 작성할 필요 없이 hlsl 하나로만 작성하면 타깃하는 쉐이더 언어로 자동으로 변환되기 때문에 매우 편리하다.                 
또한 중간 언어에서 타깃하는 쉐이더 언어로 변환 전 쉐이더 코드에 대한 최적화를 하기 때문에 성능적인 부분에서도 유리할 것이라고 생각된다.            
               
정확히 해야하는 것이 hlslcc, ShaderConductor는 쉐이더를 컴파일하는 도구가 아닌 hlsl로 작성된 쉐이더 코드를 타깃하는 쉐이더 언어로 변환하는 도구이다.                
이렇게 타깃 쉐이더 언어로 변환된 쉐이더 코드는 기기에서 컴파일된다. ( Metal의 경우 오프라인 컴파일러로 저수준 코드로 변환되기 전 중간 코드로 변환되어 빌드에 포함되고, OpenGL의 경우 고수준 glsl 코드가 필드에 포함되어 유저의 기기에서 컴파일된다. )                
             
-----------------------------             
               
hlslcc는 거의 deprecated 되었으니 ShaderConductor쪽 코드를 살펴보겠다.               
OpenGL을 타깃으로 빌드한다고 가정하고 쭉 살펴보자...                  
               
쿠킹시 우버 쉐이더에서 특정 Permutation을 기준으로 쉐이더 컴파일 Job을 추가한다.                    
<img width="876" alt="1" src="https://user-images.githubusercontent.com/33873804/194750392-111e32af-b4e2-474a-8139-2cb298456a83.png">                 
                        
타깃으로 하는 쉐이더 언어, 그래픽 API, 환경, 사용될 컴파일러 등등에 대한 전처리 매크로를 정의하는 동작들을 수행한다.                 
![2](https://user-images.githubusercontent.com/33873804/194750395-9ca3438d-fdf0-4ffb-8dbd-39eab5e638be.jpg)             
            
이제 ShaderCompilerWorker에서 실제 쉐이더 컴파일 동작 ( 정확히는 HLSL 코드를 타깃으로 하는 쉐이더 언어로 변환하는 작업 )을 수행한다.                   
<img width="835" alt="13" src="https://user-images.githubusercontent.com/33873804/194750399-5ab14ea0-31e2-46ea-919e-ba382274116e.png">              
           
앞서 말했듯이 여기서는 HLSL 코드를 GLSL ( OpenGL의 쉐이더 언어 )로 변환하는 과정을 살펴볼 것이다.             
![14](https://user-images.githubusercontent.com/33873804/194750401-2fad18df-5fb8-4725-a01a-1c80c0151461.jpg)           
              
<img width="835" alt="16" src="https://user-images.githubusercontent.com/33873804/194750403-a7ce8259-a1ee-4b1c-896e-1e7da47ddc12.png">            
          
HLSL 쉐이더 코드에 대한 전처리 동작을 수행하고 ( 우버 쉐이더 코드에서 실제 사용될 코드들만 남은 HLSL 코드로 변환된다. ), ShaderConductor를 통해 실제 쉐이더 코드 변환 작업을 수행한다.         
<img width="835" alt="16_2" src="https://user-images.githubusercontent.com/33873804/194750729-eb35e093-2182-4446-8209-7fd0275f79cf.png">             
                 
ShaderConductor에서는 HLSL 코드에 대해 SPIRV(중간 코드)로 변환하는 과정을 수행하고 변환된 SPIRV에 대한 리플랙션 데이터(버퍼 바인딩 등 다양한 용도로 사용된다)를 만들어낸다.        
그리고 SPIRV 코드를 최종적으로 GLSL 코드로 변환한다.            
<img width="857" alt="17" src="https://user-images.githubusercontent.com/33873804/194750404-f6c27c2b-06fc-438a-a5a1-6bcfc4a1ea0d.png">           
                 
                    
references : [https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/hlslcc/hlslcc](https://github.com/EpicGames/UnrealEngine/tree/ue5-main/Engine/Source/ThirdParty/hlslcc/hlslcc), [https://github.com/microsoft/DirectXShaderCompiler](https://github.com/microsoft/DirectXShaderCompiler), [https://www.slideshare.net/cagetu/gdc-14-bringing-unreal-engine-4-to-opengl](https://www.slideshare.net/cagetu/gdc-14-bringing-unreal-engine-4-to-opengl), [https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ShaderDevelopment/HLSLCrossCompiler/](https://docs.unrealengine.com/4.27/en-US/ProgrammingAndScripting/Rendering/ShaderDevelopment/HLSLCrossCompiler/), [https://github.com/KhronosGroup/SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) , [https://therealmjp.github.io/posts/shader-fp16/](https://therealmjp.github.io/posts/shader-fp16/), [https://chowdera.com/2021/08/20210802230911045t.html](https://chowdera.com/2021/08/20210802230911045t.html)          