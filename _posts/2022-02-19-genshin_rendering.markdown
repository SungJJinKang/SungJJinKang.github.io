---
layout: post
title:  "원신 콘솔 플랫폼 개발 경험 및 렌더링 파이프라인 기술 소개 해설"
date:   2022-02-19
tags: [ComputerGraphics, Recommend]
---

이 글은 [유나이트 서울 2020 - 원신 콘솔 플랫폼 개발 경험 및 렌더링 파이프라인 기술 소개](https://youtu.be/00QugD5u1CU) 영상에 나온 여러 내용들에 대한 해설이다.            


--------------------------          

- 움직이지 않는 포인트, 스팟 라이트에 대해서는 매 프레임 쉐도우 맵을 만들지 않고 쉐도우 맵을 재사용함 ( **쉐도우 캐싱, Pre-Baked 쉐도우 맵**이라 부름 ) ( Directional Light의 경우에는 사용 불가. )     
- 캐싱된 쉐도우 맵의 경우 8개의 Cascade를 사용 ( 카메라와의 거리에 따라 다른 해상도의 쉐도우 맵 사용 -> 멀리 있는 쉐도우의 품질은 낮아도 티가 안남 )        
- 매 프레임 5개의 캐싱된 Cascade 쉐도우 맵이 업데이트 됨. ( 캐릭터가 움직임에 따라 당연히 캐싱된 쉐도우 맵도 바뀌어야한다. )            
- 첫 4개의 Cascade는 매 프레임 업데이트하고, 뒤의 4개의 Cascade는 한 프레임 당 하나씩만 업데이트된다. -> 그래서 총 매 프레임마다 5개의 Cascade가 업데이트된다. ( 뒤의 4개의 Cascade의 경우 매 프레임 업데이트되지 않으니 그만큼 연산량을 아낌 )           

- 스크린 스페이스 쉐도우 맵 사용       
- 각 픽셀마다 Poisson 노이즈 소프트 쉐도우를 11번씩 계산. 노이즈의 패턴이 반복되는 것을 막기 위해 픽셀마다 샘플링 좌표를 회전시켜서 사용.       
- 연산량이 너무 비싸다. -> 모든 픽셀마다 계산할 필요없다. 쉐도우 마스크 텍스쳐를 만들어서 가장 자리만 노이즈 연산을 한다. ( 쉐도우의 가장 자리에만 티가 나기 때문 ) -> 쉐도우의 가장자리 부분만 소프트한 쉐도우 값 ( 0 ~ 1 )을 가지고 안쪽의 쉐도우들은 0 아니면 1의 값을 가짐. ( 이렇게 해서 GPU 리소스를 30% 절감 )                    
![20220220014157](https://user-images.githubusercontent.com/33873804/154810095-0800c253-92b4-453c-95a2-af2b7bcc1a58.png)      
위 그림의 빨간색 부분이 쉐도우의 가장자리 부분으로 이 부분에 대해서만 추가적인 노이즈 연산을 해주면 된다.        
                   
- 그럼 쉐도우 마스크 텍스쳐를 어떻게 생성할까?       
- 쉐도우 맵의 4x4 픽셀을 가지고 그 중 총 몇개의 픽셀에 쉐도우가 그려지는지를 1 픽셀로 표현. ( 4x4 픽셀 중 8개의 픽셀에 쉐도우가 그려짐 -> 쉐도우 마스크 텍스쳐의 픽셀은 0.5의 색깔 값을 가짐. )      
- 4x4 픽셀을 모두 읽는건 느리니 그냥 랜덤하게 몇개의 샘플만을 가지고 연산 -> 정확도가 떨어짐 ( 실제로는 가장자리인데 가장 자리가 아닌걸로 그려짐 ) -> Blur 처리해서 사용.      
- 쉐도우의 가장 자리만 추가적인 노이즈 연산을 한 경우와 그렇지 않은 경우를 비교해도 퀄리티의 차이를 못느낌.         
           
- Ambient Occlusion ( 구석진 부분에는 더 강한 음영이 생김 )        
- 원신은 HBAO ( 화면 전체에 적용 ), AO Volume ( 정적인 오브젝트에 사용 ), Capsule AO ( 캐릭터에 사용 ) 총 3가지의 기법을 사용.      

![20220220015213](https://user-images.githubusercontent.com/33873804/154810480-0a4012c7-8374-4dc4-873d-bc45bd82dd89.png)       
![20220220015220](https://user-images.githubusercontent.com/33873804/154810501-39b1e689-0714-4253-90b1-b9d285cacc14.png)      
- HBAO 사용하고 안하고의 차이가 크다. AO Volume도 마찬가지.      

- AO Volume은 물체의 그림자가 바닥에 드리우는 경우 사용하기 좋음. ( 기술적으로 HBAO로는 이런 표현을 하는 것이 불가능 )       
- AO Volume을 사용하기 위해선 해당 물체의 차폐 ( Occlusion ) 정보를 빌드 타임에 메쉬의 모델 스페이스를 기준으로 미리 저장해두어야한다.          
- 자세한 정보는 GDC 2012를 참고 ( 못찾겠다 )          
- AO Volume은 캐릭터와 같은 에니메이션이 들어가는 오브젝트에는 사용 불가 -> 정적인 오브젝트에만 사용         

- 캐릭터에 사용하는 Capsule AO의 경우에는 캐릭터의 팔, 다리와 같은 Rig에 캡슐을 붙인 후 런타임에 에니메이션에 따라 움직이는 해당 캡슐에 대해 차폐 정보를 업데이트해서 사용.            

- 모든 Ambient Occlusion 텍스쳐는 1/2 x 1/2로 렌더링 후 노이즈를 줄이기 위해 블러 처리를 거친 후 화면 해상도로 업스케일링 해서 사용.     
- 블러와 업스케일링은 양방향 필터링 처리함.       
- 양방향 필터링은 수 많은 반복되는 연산이 있기 때문에 2번의 블러처리, 1번의 업스케일링 과정을 하나의 패스로 Compute Shader에서 처리.        

- 로컬 광원에는 클러스터 디퍼드 라이팅을 구현. 시야 내 최대 1024개의 광원을 지원.      
- 로컬 광원에 대한 쉐도우의 경우 미리 Bake된 정적인 오브젝트에 대한 쉐도우 텍스쳐와 동적 오브젝트에 대한 것을 합쳐서 사용.      
- 미리 Bake된 정적 오브젝트에 대한 쉐도우 텍스쳐는 압축률도 높아야하고 압축 해제도 빨라야했음.      

- Volumetric Fog.     
- Volumetric Fog에도 로컬 광원 적용.     
- 광원의 형태에 따라 Volumetric Fog도 달라짐.               
![20220220021328](https://user-images.githubusercontent.com/33873804/154811368-43dd60c7-6d3b-4a0b-a69e-0d82e15f898c.png)       

- God Ray         
- 일반적으로는 Volumetric Fog를 이용하면 God Ray도 만들 수 있다.     
- 그러나 원신의 경우에는 해상도 문제와, God Ray가 깔끔하지, Sharp하지 않아서 별도의 패스로 God Ray를 만듬.        
아래의 사진을 보면 Volumetric Fog를 이용한 God Ray에 비해 별도의 패스로 처리한 God Ray가 훨씬 깔끔한 것을 알 수 있음. 원신이라는 게임의 그래픽 특성상 깔끔하고 샤프한 God Ray가 필요했음.         
![20220220021954](https://user-images.githubusercontent.com/33873804/154811671-1b35066b-1293-4738-8337-0defd7d73f7c.png)     
![20220220022011](https://user-images.githubusercontent.com/33873804/154811662-a7acd06a-abb3-4dfe-a8c9-4ec72aed5493.png)       
              
- Image Based Lighting               

- Reflection Probe
- 원신에서는 조명이 계속 변화해서 큐브맵을 Bake해서 사용하지는 못하고, Probe마다 저해상도 G버퍼를 저장해서 런타임에 이 저해상도 G버퍼에서 큐브맵을 생성해서 사용.      
- ReLight, Convolve, Compress 3단계로 런타임에 큐브맵을 생성           
- 큐브맵의 6면을 처리하기 위해 비동기식 Compute 쉐이더 사용.           
- ReLight 단계는 미니 G버퍼로 큐브맵을 생성하는 과정.          
- 큐브맵에 대한 mip-map 생성 후 각 mip-map 레벨마다 Convolve 처리. ( Convolve란 주변 픽셀에 대해 Convolutation Filter를 곱하여서 새로운 픽셀 값을 만들어내는 과정이다. 블러 처리도 Convolution의 일종. [이 글](https://bkshin.tistory.com/entry/OpenCV-17-%ED%95%84%ED%84%B0Filter%EC%99%80-%EC%BB%A8%EB%B3%BC%EB%A3%A8%EC%85%98Convolution-%EC%97%B0%EC%82%B0-%ED%8F%89%EA%B7%A0-%EB%B8%94%EB%9F%AC%EB%A7%81-%EA%B0%80%EC%9A%B0%EC%8B%9C%EC%95%88-%EB%B8%94%EB%9F%AC%EB%A7%81-%EB%AF%B8%EB%94%94%EC%96%B8-%EB%B8%94%EB%9F%AC%EB%A7%81-%EB%B0%94%EC%9D%B4%EB%A0%88%ED%84%B0%EB%9F%B4-%ED%95%84%ED%84%B0)을 참고하세요 )       
- 이후 압축을 한다.           

- Ambient Probe          
- 마찬가지로 런타임에 생성.     
- 위의 Reflection Probe의 ReLight가 단계가 끝난 큐브맵에서 Ambient 데이터만 가져와서 SH 값으로 저장 ( SH가 뭐지... )           
- 마찬가지로 컴퓨트 쉐이더로 6면을 처리.           

- 위의 Probe들은 쉐도우를 가지고 있지 않다. -> 2시간마다 미니 G버퍼용 쉐도우맵을 베이크해두어 3밴드 그림자 SH에 저장(?) ( 디스크 공간 절약을 위해 )                   
- 현재 시간을 기준으로 2시간 마다 베이크된 앞, 뒤 두개의 쉐도우 맵을 보간하여서 ReLight 단계에 적용하여 사용.            
             
- 실내와 실외의 Reflection Probe를 구분하여 다르게 처리.                     
- 아티스트가 인테리어 메쉬에 대해서는 인테리어 마스크를 사용하여 어떤 픽셀이 인테리어 메쉬의 일부인지를 구분되게 해주어야한다.                        
![20220220024846](https://user-images.githubusercontent.com/33873804/154812725-bfc1a8ba-60a2-41ab-b02a-026f7bd62166.png)       
- 이를 기준으로 Ambient Probe도 실내와 실외의 Fragment에 대해 다른 픽셀 데이터를 생성.       
![20220220024733](https://user-images.githubusercontent.com/33873804/154812690-37aa1fcc-a303-405b-9b30-692b41953360.png)      
![20220220024740](https://user-images.githubusercontent.com/33873804/154812693-7f2efafe-f18c-40e2-a38d-c3a9c86fe343.png)       
위 사진에서 보시다 싶이 인테리어 마스크를 적용한 경우 실내에는 푸른 빛이 없어진 것을 볼 수 있다.          
         
- Screen Space Reflection.         
- Temporal Filter를 적용함. HI-Z 버퍼를 사용.        
![20220220025113](https://user-images.githubusercontent.com/33873804/154812803-736143be-5a0a-46d9-a58f-0d0eb66cd188.png)       
![20220220025119](https://user-images.githubusercontent.com/33873804/154812805-4a7b7358-111e-42ac-8251-4b10b58b9cbb.png)           











