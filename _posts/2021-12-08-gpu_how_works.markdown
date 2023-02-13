---
layout: post
title:  "DRAM의 데이터가 어떻게 GPU로 전달될까? ( Pinned Memory )"
date:   2021-12-08
tags: [ComputerScience, ComputerGraphics]
---

이 글은 어떻게 데이터가 디스크에서 GPU, VRAM으로 전달되는지에 대한 동작 원리를 설명하는 글이다.        
시간이 된다면 [이 글](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-AsynchronousBufferTransfers.pdf)을 처음부터 끝까지 읽어보기 바란다. OpenGL을 기반으로한 글이지만 어차피 동작 원리는 DirectX와 별반 다를 것이 없으니 GPU가 어떻게 동작하는지에 대한 이해를 도와준다.           

-------------------------              

GPU로 데이터를 전송하고, 전송받는 것은 쉽지만, CPU와 GPU가 통합된 메모리를 사용하는 ( Unified Memory )가 없는 PC 아키텍쳐에서는 데이터 전송을 빠르게 하는 것이 쉽지가 않다. 더욱이 OpenGL API 스펙은 그것을 어떻게 효율적으로 할 수 있는지도 설명해주지 않고, 일반적인 데이터 전송 함수들은 프로그램의 실행의 중단을 발생시키기 때문에 CPU와 GPU 모두의 연산 시간을 낭비시킨다.           
이 장에서는 버퍼 오브젝트와 익숙한 독자들을 위해, 우리는 드라이버에서 무슨 일이 일어나는지 설명할 것이고 최대 속도의 CPU, GPU간 데이터 전송을 위해 비전통적인 방법을 포함해 여러 다양한 방법들을 알려줄 것이다. 만약 프로그램이 메쉬, 텍스쳐 데이터를 자주, 효율적으로 전송할 필요가 있다면, 이러한 함수들은 성능 향상에 큰 도움을 줄 수 있다. 이 장에서는 우리는 OpenGL 3.3을 기준으로 설명할 것이고, 이 버전은 DirectX로 치면 DirextX3D 10과 같다.             
                     

- **용어 정리**       
OpenGL의 스펙의 용어를 사용해보면 우리는 GPU를 디바이스라고 부른다. 그리고 **OpenGL API 함수들을 호출할 때, 드라이버는 그 호출들을 커맨드로 전송한다 그리고 그 호출들을 CPU의 내부 큐**에 추가한다. 이러한 커맨드들은 GPU ( 디바이스 )에 의해 비동기적으로 사용된다. 이러한 큐들을 우리는 이미 "커맨드 큐"라는 용어로 알고 있다. 그렇지만 조금 더 정확한 단어는 **"디바이스 커맨드 큐"**가 맞다.            
CPU 메모리에서 디바이스 ( GPU ) 메모리로의 데이터 전송은 "업로딩"이라 부르고, 디바이스 ( GPU ) 메모리에서 CPU 메모리로의 전송을 "다운로드"라고 부른다.          
마지막으로 **"Pinned Memory"**는 GPU가 PCI 버스를 통해 직접적으로 접근할 수 있는 메인 메모리의 일부분이다.             
( [이 글1](https://sungjjinkang.github.io/gpu_memory), [이 글2](https://sungjjinkang.github.io/IO_System)를 참고해보기 바란다. )               
              
               
- **버퍼 오브젝트들**                
많은 버퍼 오브젝트 타깃들이 존재한다.       
OpenGL 기준으로 GL_ARRAY_BUFFER, GL_ELEMENT_ARRAY_BUFFER, GL_PIXEL_PACK_BUFFER, GL_TRANSFORM_FEEDBACK_BUFFER 등이 있다.           
버퍼 오브젝트들은 리니어한 메모리 영역 ( 연속된 )으로 CPU 메모리 ( DRAM ) 혹은 GPU 메모리 ( VRAM )에 할당된다. 이 버퍼 오브젝트들은 다양한 용도로 사용될 수 있는데, 버텍스 데이터를 저장하는 용도로, 쉐이더가 거대한 연속된 메모리 영역을 접근할 수 있도록 텍스쳐 버퍼 용도로, 유니폼 버퍼, 텍스쳐 업로드 다운로드를 위한 픽셀 버퍼 오브젝트로도 사용될 수 있다.         
              

- **메모리 전송**             
메모리 전송은 OpenGL에서 중요한 역할을 한다. 메모리 전송을 이해하는 것이 3D 어플리케이션에서 높은 성능을 달성하는데 가장 중요한 요소이다. 두가지 주요 GPU 아키텍쳐가 존재한다. Discrete GPU ( 일반적으로 PC에서 사용하는 GPU가 CPU와 별개로 존재하는 GPU 형태이다 ), Integrated GPU ( 흔히 콘솔에서 보는 GPU로 CPU와 GPU가 같은 메모리 공유해서 사용한다. )가 그것이다. Integrated GPU의 경우 CPU와 GPU는 같은 다이를 공유하고, 메모리 공간도 공유한다. 이는 보통 GPU로의 데이터 전송이 PCI를 통해 이루어지는 Discrete GPU에 비해 데이터 전송면에서 많은 성능 향상을 가져다 준다.         
그러나 일반적으로 Integrated 유닛들은 Discrete GPU와 비교하여 그냥 그럭저럭한 성능을 가진다. **Discrete GPU의 유닛은 GPU 보드 위에 훨씬 빠른 메모리를 가지고 있고 이는 Integrated GPU의 Unified 메모리보다 몇배는 빠르다.**                           
**DMA 컨트롤러**는 CPU 사이클을 소비하지 않고 **OpenGL 드라이버가 유저 메모리 ( DRAM )에서 GPU 메모리로 메모리 블록을 비동기적으로 전송**하는 것을 도와준다. ( IO 통신을 위해서는 주소 버스에 CPU가 매번 전송할 위치를 써주어야한다. 이렇게 매번 CPU가 IO 통신을 위해 주소 버스에 주소를 쓰는 동작을 하게되면 CPU가 다른 일을 할 수 없으니 이를 DMA 컨트롤러가 대신하는 것이다. ) 이러한 비동기적 데이터 전송은 픽셀 버퍼 오브젝트를 사용할 때 가장 자주 사용되는 방식 중 하나이다. 픽셀 버퍼 오브젝트 뿐만아니라 다른 버퍼 타입의 데이터 전송에도 사용된다. 기억해야하는 것은 이러한 비동기적 데이터 전송은 **CPU의 관점에서 비동기적이라는 것**이다. CPU가 할 일을 DMA 컨트롤러가 대신 해주니 CPU 관점에서는 비동기적이다. 그러나 Fermi, Nothern Islands **아키텍쳐의 GPU들은 버퍼 전송과 렌더링을 동시에 할 수 없다.** 그래서 커맨트 큐에 있는 모든 OpenGL 커맨드들은 GPU에 의해서 **순차적으로 처리 ( GPU는 데이터 전송을 비동기적으로 할 수 없다. )**된다. 이러한 한계는 부분적으로는 **드라이브로부터 오는 한계**이기 때문에, 자주 변한다. ( 드라이버만 고치면 해결할 수 있다. ) 실제로 CUDA API에서는 비동기적 데이터 전송을 지원한다. 또한 **NVIDIA Quadro 아키텍쳐에서는 텍스쳐 전송을 하면서 동시에 렌더링을 하는 것이 가능**하다.                 
GPU로 데이터를 주고 받는데는 두 가지 방법이 있는데 OpenGL 기준으로 glBufferData, glBUffersSubData가 있다. 이러한 함수들은 직관적으로 사용할 수 있지만, 최고의 성능을 위해서는 그 뒤에서 어떻게 동작하는지를 알 필요가 있다.             

<img width="399" alt="20211208200616" src="https://user-images.githubusercontent.com/33873804/145198167-a6686566-0bc4-4566-81eb-7490a31d8e07.png">          

위의 사진에서 보이듯이 이 함수들은 우선 유저 모드 메모리 영역 ( DRAM )의 데이터를 **GPU가 직접 접근할 수 있는 DRAM 내부의 Pinned Memory로 데이터를 우선 복사한다. ( DRAM -> DRAM 복사 )** 이러한 과정은 일반적인 **memcpy와 비슷**하다. **일단 완료되면 드라이버는 DMA 전송을 시작**한다. 이 DMA 전송은 위에서 말했듯이 **비동기적**이기 때문에 Pinned Memory로 전송이 끝나고 DMA 전송 커맨드만 전송하면 DMA 전송이 시작되기 전 glBufferData는 반환된다. 데이터 전송의 목적이는 Usage hint, 드라이버 구현에 달려있는데 이는 나중에 설명할 것이다. 몇몇 경우에는 데이터는 그냥 DRAM Pinned 메모리에 머무르고 GPU가 이 메모리에 직접 접근하기도 한다. ( VRAM으로 데이터 전송이 안이루어진다는 것이다. ) ( 결과적으로는 한번의 데이터 이동만 발생한 것이다. ) 데이터를 어떻게 생성하느냐에 따라 이 한번의 데이터 이동 ( DRAM -> DRAM )도 안할 수 있다. ( 처음부터 데이터를 Pinned 메모리에 올리는 것이다. )        
GPU에 데이터를 올리는 더 효과적인 방법은 **glMapBuffer, glUnmapBuffer 함수를 사용해 내부 드라이버의 메모리 영역의 주소를 직접 가져오는 것이다. ( Pinned 메모리에다 그냥 바로 쓰는 것 )** 대부분의 경우 이러한 메모리들은 Pinned 되어 있다. 물론 이 또한 드라이버에 따라 다르다. ( 여기서 **Pinned되어 있다는 것은 해당 메모리 영역이 Disk로 Page out되지 않고, 쓰기 동작을 수행할 때도 캐싱을 하지 않고 항상 DRAM에 쓴다는 것을 의미**한다. ) 우리는 이 주소를 가지고 버퍼를 직접 채울 수 있다. **예를 들면 그냥 디스크에서 텍스쳐, 메쉬 데이터를 읽어 올 때 읽어올 버퍼를 따로 만드는 것이 아니라 그냥 여기 Pinned 메모리 영역으로 곧바로 읽어오는 것이다. 결과적으로 유저 모드 메모리 영역에서 Pinned 메모리로의 복사가 없으니 메모리 복사를 한번 덜할 수 있게 된다. Write Combine 옵션을 붙이면 [Write Combine 버퍼](https://sungjjinkang.github.io/nonTemporalMemoryHint)를 활용할 수도 있다.**             

아래의 사진은 Pinned 메모리를 사용했을 떄의 GPU로의 데이터 전송을 보여준다.           

<img width="393" alt="20211208200623" src="https://user-images.githubusercontent.com/33873804/145200441-67db91cc-6257-4b9d-b780-33e007334c9a.png">          

          
- **Usage Hints**           
OpenGL 드라이버가 데이터를 저장할 수 있는 장소로 주로 두가지 장소가 있는데 CPU 메모리 ( DRAM ), GPU 메모리 ( VRAM )이 그것이다. DRAM은 **페이지 Locked ( Pinned Memory )**일 수 있다. 페이지 Locked이라는 의미는 **Disk로 Evict 되지 않는다**는 의미이고, **GPU가 직접 접근** 할 수 있다는 ( DMA를 통해 ) 의미이기도 하다. **반면**에 **Paged 메모리**라는 것도 있는데 이 Page 메모리도 마찬가지로 GPU가 접근할 수는 있지만 훨씬 **비효율적**이다. 우리는 드라이버에게 어떤 메모리를 사용할 것인지 힌트를 줄 수 있다. 물론 드라이버가 항상 그 힌트를 따르는 것은 아니다. 드라이버가 어떻게 구현되어 있느냐에 다 달려있다.               

여기까지가 기본적인 DRAM, GPU간 데이터 전송의 얘기이고 이후의 내용은 [이 글](https://www.seas.upenn.edu/~pcozzi/OpenGLInsights/OpenGLInsights-AsynchronousBufferTransfers.pdf)을 참고하라.             
