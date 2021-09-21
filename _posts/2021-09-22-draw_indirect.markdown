---
layout: post
title:  "Draw Indirect with Compute Shader"
date:   2021-09-22
categories: ComputerGraphics
---

Draw Indirect는 쉽게 말하면 그릴 Mesh들의 버텍스, 쉐이더, 텍스쳐, 트랜스폼 등등의 오브젝트를 그리기 위한 데이터들을 왕창 GPU에 다 옮겨두고 GPU에게 Mesh들을 그리는 것을 전적으로 맡기는 것이다.        

생각을 해보자. 동일한 Shader로 만들어진 10개의 Material와 10개의 Mesh를 가지고 각 Material, Mesh 당 100개의 오브젝트를 그린다고 가정해보자.       
**일반적으로 렌더링**을 하게되면 어떤 특정 Material, Mesh의 하나의 Instance를 그리는데 한번의 Uniform Data ( Model Matrix... )를 GPU에 전송해야하고, 한번의 Draw 커맨드 전송이 필요하다.        
```
for(material)
{
    for(mesh)
    {
        for(instance)
        {
            Uniform Data 전송
            Draw 커맨드 전송
        }
    }
}
```
결과적으로 10 * 10 * 100 -> 10000 번의 Draw Call이 발생한다!!!           
아마 이 글을 읽는 사람은 이 Draw Call 커맨드를 매번 GPU에 전송하는 것이 얼마나 느릴지 알 것이다.      

여기서 조금 발전하면 **GPU Instancing 기법**이 있다.             
이 기법은 똑같은 Matrial, 똑같은 Mesh를 사용할 때는 한번의 Draw Call 커맨드로 여러 Instance를 그리는 것이다.    
```
for(material)
{
    for(mesh)
    {
        여러 Instance들의 Uniform Data를 한꺼번에 전송
        DrawInstanced 커맨드 전송
    }
}
```
조금 나아졌다. 10 * 10 -> 100번의 Draw Call이 발생한다.      
GPU Instancing 방법을 사용하여도 몇개의 Instance를 그릴지는 CPU에서 GPU로 전송해주어야한다.             

다음으로 이 글에서 말할 **Draw Indirect**은 어떨까??         
```
for(shader))
{
    if(Uniform Buffer를 업데이트 할 필요가 있을 때만)
    {
        Uniform Buffer를 업데이트
    }
    
    DrawIndirect 커맨드 전송
}
```
1번의 DrawCall만으로 끝난다. ( OpenGL 기준 ARB_multi_draw_indirect )        
이는 Draw Indirect 방식이 어떤 Mesh, 어떤 Material을 가지고 몇번을 그릴 지 등의 모든 **데이터들이 CPU에서 GPU로 매번 전송하는 것이 아니라 VRAM에 미리 그러한 데이터들을 가져다두고 GPU에게 그 VRAM에 전송해둔 데이터를 가지고 그리라고 명령을 내리는 방식**이기 때문에 1번의 Draw Call을 가지고 렌더링하는 것이 가능한 것이다.           

이 Draw Indirect 방법은 심지어는 특정 Mesh를 몇번 그릴지가 ( Instance 수 ) 바뀌지 않으면 그냥 GPU가 가지고 있는 Instance 수를 그대로 사용하면 되기 때문에 매번 CPU가 Instance 개수를 GPU에 전송해줄 필요도 없다. 그냥 Draw Indirect 커맨드만 달랑 전송하면 된다. CPU에서 GPU로 가는 데이터의 양이 획기적으로 줄었다.               
CPU에서 GPU로 가는 데이터, 커맨드의 수가 줄었다는 것은 DRAM - VRAM간의 Memory Latency로 인한 속도 저하 뿐만 아니라 **CPU - GPU 간의 동기화가 줄었다는 것을 의미한다. 이 동기화가 주는데서 오는 성능 향상은 매우 크다.** ( GPU가 CPU에 Interrupt를 날리면 CPU는 이 Interrupt를 처리하기 위해 많은 시간을 소모한다. ) ( CPU - GPU간의 동기화에 대해 잘 모르겠다면 [이 글](https://sungjjinkang.github.io/computerscience/computergraphics/2021/09/04/gpu_architecture.html)을 읽어보기 바란다. )        

어찌되었든 **핵심은 CPU와 GPU간의 상호작용 ( 데이터 전송, 동기화.... )을 최소화하는 것**이다.        

간단히 코드를 살펴보면 이렇다.
```
typedef  struct {
   GLuint  count;
   GLuint  instanceCount;
   GLuint  first;
   GLuint  baseInstance;
} DrawArraysIndirectCommand;

glMultiDrawArraysIndirect(GLenum mode​, const void *indirect​, GLsizei drawcount​, GLsizei stride​);
```
glMultiDrawArraysIndirect의 indirect에 위의 DrawArraysIndirectCommand 데이터를 전송해서 VRAM에 있는 Vertex, Indice, Transform 등의 데이터를 활용하여 몇개의 instance를 그릴지를 정한다.          
그리고 위에서 말했듯이 몇개의 instance를 그릴지를 이렇게 매번 CPU에서 GPU로 전송하지 않고 null값을 전달하면 GPU는 알아서 VRAM에 이미 전송되어 있던 DrawArraysIndirectCommand 데이터를 활용하여 Draw를 수행한다.    
( 자세한건 [이 글](https://www.khronos.org/opengl/wiki/Vertex_Rendering#Indirect_rendering)을 참고하라. )      

그런데 한 가지 문제가 더 있다.         
일반적으로는 렌더링을 할 때는 CPU단에서 View Frustum Culling이나 Occlusion Culling을 한 후 최종적으로 화면에 그려질 오브젝트들에 대해서만 Draw Call을 던지니 컬링이 된 오브젝트들은 절대로 GPU 파이프라인에 들어갈 일이 없다.       
반면 이 Draw Indirect 방법은 컬링 여부와 관계 없이 무조건 인스턴스 개수만큼 메쉬 데이터가 GPU 파이프라인에 들어갈 것 같아보인다.           

여기서 **Compute Shader**가 등장한다.     
Compute Shader를 통해 View Frustum Culling이나 Occlusion Culling을 수행한 후 각 Instance가 컬링되었는지 여부를 VRAM에 저장하고 ( **Instance가 컬링되었는지 여부를 CPU로 전송하지 않는다. 왜? 느리기 때문에** ) 있다가 Instance를 그릴지 여부를 결정할 때 활용한다!!            
알다 싶이 GPU는 병렬적으로 데이터를 처리하는데 특화되어 있기 때문에 이러한 Culling 작업을 빠르게 처리할 수 있을 것 같다.               

아래 두 사진은 일반적인 Rendering과 Draw Indirect 기법을 활용한 렌더링간의 GPU 상태 변화, 커맨드 전송의 차이를 보여준다. 어떤 사진이 Draw Indirect 기법을 사용했을 때인지는 잘 알것이다.        



**다만 이 Draw Indirect 기법도 일장일단이 있어보인다.**               
CPU에서 수행했던 ( 대부분의 게임에서 ) View Frustum Culling, Occlusion Culling 연산을 GPU에서 대신하다보니 그 만큼 GPU의 연상량이 늘어났다.          
**일반적으로 대부분의 게임은 GPU Bound ( 프레임 저하의 원인이 GPU인 )하다보니 되도록이면 GPU의 연산량을 덜어주려고 하는데 이 방법은 오히려 CPU에서 하던 일까지 GPU에게 맡겨버리는 꼴이 되었다.**                                      
정확하게 이 Compute Shader를 활용하여 컬링을 하고 Draw Indirect를 하는 방법이 CPU에서 컬링을 하는 방법보다 실제 게임에서 작동을 했을 때 더 빠른지, 더 느린지에 대해서는 잘 알지 못하겠다.        
( GPU가 병렬적으로 연산을 하는데 특화되어 있기 때문에 이 방법이 더 빠를 것 같기도 하다. )           
              
현재는 Masked SW Occlusion Culling ( CPU 단에서 Occlusion Culling )을 구현하고 있고 현재 제작 중인 [게임 엔진](https://github.com/SungJJinKang/EveryCulling)에서도 CPU단에서 Frustum Culling을 하고 있기 때문에 이 방법을 당장은 구현해보지 못할 것 같다.        
그렇지만 최근 나온 게임들에서도 (수 백개의 나무, 풀 같은 오브젝트를 그리는데 많이들 활용한다)[https://assetstore.unity.com/packages/tools/utilities/gpu-instancer-117566?locale=ko-KR]고 알고 있는데 나중에 한번 구현해 보고 싶다.         

----------------------


이와 비슷한 개념이 OpenGL의 Conditional Rendering이 있다. 이것은 Occlusion Query 기법에서 사용되는데, Occlusion Query 기법을 간단히 설명하면 Occluder를 먼저 그리고 Occludee들에 대해 단순하게( 간단한 Fragment Shader를 가지고 ) 한번 오브젝트를 그려보고 어떤 Fragment도 Depth Test를 통과하지 못하면 그 오브젝트에 대해서는 Draw API를 호출하지 않는 것이다. 이를 통해 간단하게 한번 오브젝트를 그려보고 만약 안그려도 된다고 판단이 되면 해당 오브젝트를 그리는 것을 생략하여 GPU 연산을 아낄 수 있다.        
그런데 해당 오브젝트에 대한 Draw API 호출을 할지 말지 결정하기 위해서는 이 Query 데이터 ( 오브젝트가 그려질지 말지 )를 다시 DRAM으로 가져와야한다. VRAM에서 DRAM으로 데이터를 전송하는 것은 느리다....     
여기서 Conditional Rendering이 등장하는데 Occlusion Query를 통해 어떤 오브젝트가 컬링이 되었는지 안되었는지 여부를 다시 DRAM으로 가져오지 말고 그냥 GPU에서 그 데이터를 가지고 있다가 Draw 커맨드가 들어오면 가지고 있던 Query 데이터를 가지고 Draw를 할지 말지 결정하라는 것이다. 이는 Query 데이터를 다시 DRAM으로 가져오는 것이 느리고 심지어는 Query 작업이 아직 끝나지 않았으면 CPU가 기다려야하는데서 오는 성능 저하를 방지하기 위함이다.    
일단 CPU단에서 GPU에 Draw 커맨드를 전송하되 그 Draw 커맨드를 처리할지 말지(그릴지 말지) 여부를 그냥 GPU에게 맡기는 것이다.                       
생각을 해보면 Query 연산이 다 끝나기를 기다리고 그 데이터를 다시 DRAM으로 가져올 때까지 기다릴바에는 그냥 GPU에 Draw 커맨드를 던져두고 CPU는 그 시간에 다른 연산을 더 하는 것이 이득이다.     

이 Occlusion Query + Conditional Rendering 기법도 위에서 말한 GPU에서 알아서 오브젝트를 그릴지 말지 여부를 판단한다는 점에서 비슷해 보이지만 Occlusion Query 기법은 단순하게라도 ( 단순한 Fragment Shader 사용 ) 오브젝트 ( Occludee )를 그려보아야한다. 또한 Occludee들에 대해 Draw 커맨드를 일일이 전송해야한다.         
이에 반해 Compute Shader를 이용하면 Occludee를 직접 그리는 것이 아니라 단순하게 Occluder를 그린 후 Depth Buffer와 Occludee의 가장 가까운 Depth 값을 비교하는 등과 같은 등의 여러 기법들을 활용하여 빠르게 Cull 여부를 판단할 수 있기 떄문에 Query 방법에 비해 더 빠르다. ( [6페이지 참고](https://www.slideshare.net/dgtman/sw-occlusion-culling) )                   

references : [https://de.slideshare.net/CassEveritt/beyond-porting](https://de.slideshare.net/CassEveritt/beyond-porting), [https://stackoverflow.com/questions/19534284/what-are-the-advantage-of-using-indirect-rendering-in-opengl](https://stackoverflow.com/questions/19534284/what-are-the-advantage-of-using-indirect-rendering-in-opengl), [https://www.slideshare.net/dgtman/sw-occlusion-culling](https://www.slideshare.net/dgtman/sw-occlusion-culling)

