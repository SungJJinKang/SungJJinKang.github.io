---
layout: post
title:  "Static Bathcing 구현하기"
date:   2022-02-22
categories: ComputerScience ComputerGraphics
---

**Static Batch는 동일한 매터리얼 사용하는 오브젝트들의 메쉬 버텍스들을 모두 World Space로 Proeject한 후 GPU에 왕창 올려두고 하나의 Draw Call로 그리는 기법이다.**                     

핵심은 여러 오브젝트들을 **하나의 Draw Call로 그리는 것**, 렌더링을 위한 **데이터들을 매번 갱신해줄 필요가 없다는 것**이다.              
동일한 Material ( Shader, Texture가 같다 )을 사용하는 오브젝트들의 메쉬 버텍스 데이터만 함께 모아서 그리면 한번의 Draw Call로 그릴 수 있다.       

다만 오브젝트들의 메쉬 데이터들을 모두 World Space로 옮겨서 일일이 메모리(VRAM)에 올려두어야하니 **메모리 사용량이 증가하는 문제**가 있다. ( 실제로 문제가 되어서 Static Batch할 수 있는 메쉬의 버텍스 개수에 제한을 두었다. )              

내가 생각하기에 그 보다 더 큰 문제는 **Culling 기법을 적용하기가 힘들다**는 것이다.      
특정 오브젝트가 컬링되었는지 안되었는지를 CPU에서 드로우 콜을 떄릴 때 GPU에 전송을 해주어 컬링 된 오브젝트의 버텍스는 아예 그래픽 파이프라인에도 들어가지 않게 만들고 싶은데 쉽지 않을 것 같다.        
가장 쉬운 방법은 **인덱스 버퍼만 매 프레임 갱신해서 GPU에 전송해주어서 컬링이 되지 않은 오브젝트들의 인덱스는 렌더링에서 제외시켜주면 될 것 같다.**             

그래서 이 글에서는 필자가 어떻게 Static Batching을 사용하면서 [CPU 단계에서 수행한 Culling](https://github.com/SungJJinKang/EveryCulling)의 결과를 Static Batching 렌더링에 반영하여 최적화를 하였는지에 대해 글을 작성해 볼 것이다.         
그리고 Static Batching 적용 전 후의 퍼포먼스 차이.        
Static Batching을 적용한 상태에서 CPU Culling ( 현재는 멀티스레드 뷰프러스텀 컬링만 구현되어 있다 )의 결과를 반영하기 전 후의 퍼포먼스 차이에 대해서도 서술할 것이다.                 

---------------------------------------------------         

우선 Static Batching 메쉬를 만드는 것 자체는 매우 간단하다. 내 엔진에서 Material은 모두 객체화가 되어 있으니 같은 Material을 참조하는 Renderer 컴포넌트들을 모아서 그 오브젝트들의 메쉬의 버텍스등를 World Space로 Project 후 한 곳에 모아주면 된다.       
매우 간단하다.          
추가적으로 Normal 값 계산을 위한 TBN 매트릭스도 CPU단에서 연산한 후 GPU에 올려주었다.         
처음 한번만 이렇게 연산을 하면 된다.         

필자의 엔진에서는 오브젝트들의 Mobility를 설정할 수 있다. ( 언리얼 엔진과 비슷하다. )          
오브젝트의 Mobility가 Static인 경우 엔진에서 자동으로 Static한 오브젝트들 중 같은 Material을 사용하는 오브젝트들을 모아서 자동으로 Static Batch된 메쉬 데이터를 만들어주고 GPU에 올려준다.        
그냥 **오브젝트에 Static Mobility만 셋팅해주면 모든 과정이 자동**으로 이루어진다.             
[영상](https://youtu.be/bBDbO7hS12g)             
Static Batching을 사용했을 때 당연히 그렇지 않은 경우보다 빠르다.         
모두 자동화가 되어 있으니 사용도 간편하다.         

-------------------------------            

문제는 Batching시 사용하는 쉐이더를 따로 만들어야하는데 쉐이더마다 일일이 Batching 버전을 직접 만드는 것은 매우 귀찮으니 Shader Permutation이 필요할 것 같다. Shader Pemutation은 쉽게 생각하면 #ifdef 같은 Preprocessor을 쉐이더에도 적용한다고 생각하면 된다. 유니티에서도 [Shader Variant](https://docs.unity3d.com/2019.3/Documentation/Manual/SL-MultipleProgramVariants.html)라는 비슷한 기능을 지원한다. 필자도 현재 사용 중인 [glslcc](https://github.com/septag/glslcc)를 활용할 예정이다.        

--------------------------------       

이제는 Culling을 적용할 차례이다.       
