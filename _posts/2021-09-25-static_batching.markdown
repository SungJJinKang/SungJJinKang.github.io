---
layout: post
title:  "Static Bathcing 구현하기 ( View Frustum 컬링 적용하기 ) ( 작성 중 )"
date:   2021-09-25
categories: ComputerScience ComputerGraphics
---

**Static Batch는 동일한 매터리얼 사용하는 오브젝트들의 Transform Data를 모두 World Space로 Proeject한 후 GPU에 왕창 올려두고 하나의 Draw Call로 그리는 기법이다.**                      
핵심은 여러 오브젝트들을 한번의 Draw Call로 그리는 것이다.            
동일한 Material ( Shader, Texture가 같다 )를 사용하니 버텍스 데이터만 함께 모아서 그리면 한번의 Draw Call로 그릴 수 있다.             
또한 오브젝트가 움직이지 않다 보니 매번 Transform Data ( 모델 매트릭스 )를 새롭게 GPU에 전송해줄 필요가 없으니 **맨 처음 한번만 GPU에 World Space 버텍스 데이터를 왕창 올려두고 상태 ( State ) 설정 후 Draw Call을 때려주면 된다.**              
여러 메쉬들의 버텍스 데이터를 연속되게 한 곳에 모아야하니 메모리는 좀 더 잡아 먹을 것 같지만 큰 문제는 되지 않을 것 같다.            

내가 생각하기에 그 보다 더 큰 문제는 **Culling 기법을 적용하기가 힘들다**는 것이다.      
특정 오브젝트가 컬링되었는지 안되었는지를 CPU에서 드로우 콜을 떄릴 때 GPU에 전송을 해주어 컬링 된 오브젝트의 버텍스는 아예 그래픽 파이프라인에도 들어가지 않게 만들고 싶은데 쉽지 않을 것 같다.        
가장 쉬운 방법은 **인덱스 버퍼만 매 프레임 갱신에 GPU에 전송해주어서 컬링이 되지 않은 오브젝트들의 버텍스의 인덱스를 전송해주면 될 것 같다.** 이러면 매 프레임 ( 컬링된 오브젝트의 종류가 바뀐 경우 ) 인덱스 리스트를 새롭게 만들어서 GPU에 전송해주어야한다.      

그래서 이 글에서는 필자가 어떻게 Static Batching을 사용하면서 [CPU 단계에서 수행한 Culling](https://github.com/SungJJinKang/EveryCulling)의 결과를 Static Batching 렌더링에 반영하여 최적화를 하였는지에 대해 글을 작성해 볼 것이다.         
그리고 Static Batching 적용 전 후의 퍼포먼스 차이.        
Static Batching을 적용한 상태에서 CPU Culling ( 현재는 멀티스레드 뷰프러스텀 컬링만 구현되어 있다 )의 결과를 반영하기 전 후의 퍼포먼스 차이에 대해서도 서술할 것이다. ( 오클루전 컬링이 적용이 되었다면 퍼포먼스 차이가 매우 클 것 같은데 뷰 프로스텀 컬링만 적용을 했기 때문에 두 경우의 퍼포먼스 차이가 크지는 않을 것 같기도 하다. )               

---------------------------------------------------         

우선 Static Batching 메쉬를 만드는 것 자체는 매우 간단하다. 내 엔진에서 Material은 모두 객체화가 되어 있으니 같은 Material을 참조하는 Renderer 컴포넌트들을 모아서 그 메쉬들의 버텍스를 World Space로 Project 후 한 곳에 모아주면 된다.       
매우 간단하다.           

생각해보니 Batching된 버텍스들은 버텍스 쉐이딩 단계에서 ModelMatrix가 필요없다. 그러니 단위 행렬을 Matrix에 넣어주거나 ( 이 경우 불필요한 4x4 연산이 버텍스마다 들어간다. ) 혹은 쉐이더를 따로 만들어줘야한다.        
성능을 고려하면 아무래도 후자의 방법을 채택해야하는데 이 경우 쉐이더마다 일일이 Batching 버전을 직접 만드는 것은 매우 귀찮으니 Shader Permutation을 직접 구현해야했다. Shader Pemutation은 쉽게 생각하면 #ifdef 같은 Preprocessor을 쉐이더에도 적용한다고 생각하면 된다. 유니티에서도 [Shader Variant](https://docs.unity3d.com/2019.3/Documentation/Manual/SL-MultipleProgramVariants.html)라는 비슷한 기능을 지원한다. 다만 GLSL에서 직접 지원을 하지는 않으니 직접 만들어야겠다.....      
