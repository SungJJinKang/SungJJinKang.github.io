---
layout: post
title:  "OpenGL과 DirectX 차이"
date:   2021-09-27
tags: [ComputerGraphics]
---

OpenGL과 DirectX 둘은 무엇이 다를까?           
윈도우 플랫폼 전용이다 플랫폼 독립적이다 이런 것들은 제외하고,          
프로그래머 입장에서 보면 이 **둘의 차이는 하드웨어 리소스를 어떻게, 누가 관리하냐**이다.        

**OpenGL**의 경우 텍스쳐나 버텍스 데이터, 커맨드 버퍼와 같은 **리소스들의 관리를 대부분 Driver단에서 대신해준다.** 그러니 어플리케이션 개발자는 편하다.          
반면 **DirectX**는 이러한 **리소스 관리를 어플리케이션단에서 직접해야한다.**                  

언뜻보면 OpenGL이 어플리케이션 개발자가 신경쓸 일이 덜한 것 같지만 그 만큼 성능상 최적화할 수 있는 부분이 덜하다.         
뭔 말이냐면 **DirectX의 경우 리소스 관리를 직접 자유자재로 할 수 있으니 GPU가 어떻게 돌아가는지만 잘 이해하고 있으면 그 만큼 성능상 최적화를 할 수 있다**는 것이다.         

예를 들면 DirectX12에서는 GPU 관련 메모리를 할당할 때 어디에 할당할지를 정해줄 수 있다.     
그래픽 관련 작업에 사용할 메모리를 할당할 때,      
한번 데이터를 GPU에 올려두면 CPU가 이후 다시 수정할 일이 적은 데이터인 경우 ( 텍스쳐나 버텍스 데이터, 렌더 타켓 텍스쳐 같은 것들이 있다. ) D3D12_HEAP_TYPE_DEFAULT 옵션을 통해 GPU의 로컬 메모리 VRAM에 메모리를 할당할 수 있고,         
CPU가 자주 접근하거나 읽거갈 일이 많은 데이터인 경우에는 D3D12_HEAP_TYPE_UPLOAD 옵션을 통해서 시스템 메모리 ( DRAM )에 할당을 할 수 있다.         

더 자세히 알고 싶다면 [이 글1](https://docs.microsoft.com/en-us/windows/win32/direct3d10/d3d10-graphics-programming-guide-resources-mapping)과 [이 글2](https://docs.microsoft.com/ko-kr/windows/win32/direct3d12/recording-command-lists-and-bundles)을 읽어보기 바란다.         
[Vulkan, DX12의 리소스 관리](https://gpuopen.com/wp-content/uploads/2018/05/gdc_2018_tutorial_memory_management_vulkan_dx12.pptx)         

------------------------       


최근에는 OpenGL 진영에서도 이러한 OpenGL의 성능상 문제를 해결하기 위해 리소스 관리를 직접할 수 있는 Vulkan을 밀고있다. 근데 잘 되는 것 같지는 않다. 안드로이드나 애플 진영에서도 각자의 API를 밀고 있다보니 Vulkan이 설 진영이 점차 없어지고 있다. 둠과 같은 게임도 예전에는 플랫폼에 종속되지 않기 위해 OpenGL을 사용했지만 최근에는 윈도우쪽에서는 DirectX로 넘어간 것 같다.        