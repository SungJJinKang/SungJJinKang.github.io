---
layout: post
title:  "OpenGL, D3D11 Portable한 게임 엔진 만들기 ( 작성 중 )"
date:   2022-02-02
categories: ComputerScience
---

개발 중인 DoomsEngine은 원래는 OpenGL을 기반으로 코드를 짯다.            
그런데 계속 구직 활동을 하다보니 회사에서 D3D11, 12에 대한 지식을 요구하는 것 같아 D3D11을 공부하게 되었고 이를 개발 중이던 개인 엔진에 적용을 시켜야겠다고 생각하였다.         
기존 OpenGL코드를 모두 날려버리기 보다는 OPENGL, D3D11을 추상화하여서 두 API 모두를 Portable하게 지원하기로 결정하였다.      

또한 GLSL ( OPENGL 쉐이더 언어 )로 작성을 하면 자동으로 이를 HLSL ( D3D 쉐이더 언어 )로 바꾸는 기능을 완전히 자동화하여 구현할 것이다. GLSL, HLSL 두 버전으로 쉐이더를 작성하는건 아주 번거러운 일이니 어찌보면 필수적으로 있어야할 기능이다. 이를 위해 [glslcc](https://github.com/septag/glslcc)이라는 오픈 소스를 활용할 것이고, 관련 기능들을 자동화하여서 엔진을 사용할 때는 윗단에서는 이러한 변환 과정을 몰라도 되게 자동화를 할 것이다. 또한 쉐이더의 Uniform Buffer ( Constant Buffer )의 offset이나 size 관련 데이터 reflect하여서 확인할 수 있게 기능을 구현할 것이다.               

작업 기간은 3주 정도 걸렸다. 기존에 OPENGL로 짜여진 코드에서 D3D11을 추가하려하니 완전히 개념적으로 다른 부분들이 몇가지 있어서 고생을 조금 했다.        

-----------------------------------            

우선 OPENGL, D3D11 명령어 코드를 완전히 분리하여 런타임에 Explicit dynamic linking으로 불러올 것이다.       
이를 위한 작업을 하였고 본격적으로 OPENGL과 D3D11의 여러 개념들을 추상화하는 작업에 들어갔다.           
몇몇 부분에서는 불가피하게 if, else문으로 두 API마다 각기 다른 동작을 구현해주었기는 했지만 최대한 if, else문을 자제하려고 노력하였다. ( 특히 매 프레임 수 천번 호출될 기능들은 최대한 if, else문을 사용하지 않기 위해 노력하였다. ) ( 그런데 분기 예측이 들어가서 if, else문 사용해도 큰 성능 차이는 없으려나?????.... )              
그리고 여기서 말하는 if, else문은 엔진 내의 여러 그래픽 API 추상화 클래스 ( FrameBuffer, TextureView, Material, ...) 내부적으로 if, else문을 사용하는 것이니, 엔진을 사용하는 프로그래머는 이 두 API 차이에 따른 내부 동작을 몰라도 된다.                      

대표적으로 OPENGL과 D3D11이 개념적으로 다른 부분에 대해 몇가지 서술해보겠다.            

1. OPENGL의 경우 Program에 쉐이더를 붙인 후 Program을 전체 파이프라인에 붙이는 형태인데 반해, D3D11은 ShaderView를 각각의 파이프라인의 단계에 맞게 일일이 붙여주어야한다. : 이를 해결하기 위한 OPENGL쪽 Extension이 있지만 번거러운 것 같아 그냥 if, else 문으로 처리해주었다.                       

2. 위와 마찬가지로 상수 버퍼 ( OPENGL : Uniform Buffer Object, D3D11 : Constant Buffer ) 또한 OPENGL과 달리 D3D11은 각각의 파이프라인의 단계에 맞게 일일이 붙여주어야한다. : 이 또한 불가피하게 if, else문으로 처리하였다.                      

3. OPENGL의 Render Buffer라는 개념이 D3D11에는 존재하지 않고 D3D에서는 FrameBuffer ( Render Target View )에 붙일 텍스쳐의 Usage 옵션을 통해서 텍스쳐과 Pinned Memory로 읽어올지, GPU에 둘지 등의 동작이 결정된다. : if, else 문으로 처리.         

4. OPENGL에는 텍스쳐의 포맷이 압축 포맷이냐, 그렇지 않은 포맷이냐에 따라 호출하는 함수도 달라지지만, D3D11의 경우 텍스쳐 생성시 Format 변수 하나에 압축 포맷, 비압축 포맷 중 하나를 선택하여 저장하면 된다. : if, else 문으로 처리.                

5. OPENGL의 경우 FrameBuffer 사용시 텍스쳐가 몇개 붙어 있는지와 상관 없이 FrameBuffer를 바인딩해서 사용하면 되지만 D3D11의 경우에는 텍스쳐의 개수만큼 생성된 RenderTargetView를 붙여야한다.        

6. OPENGL의 경우 Texture Resource를 바로 PipeLine에 붙일 수 있지만, D3D11의 경우 Texture Resource와 별개로 ID3D11ShaderResourceView를 사용하여 목표로하는 PipeLine 단계에 붙여야한다. ( 이렇게 D3D11의 경우 OPENGL과 달리 Resource와 그 Resource를 활용하기 위한 ~View 오브젝트들이 존재한다. )                    


그리고 위에서 말한 [glslcc](https://github.com/septag/glslcc)를 사용하여서 GLSL로 작성된 쉐이더 파일을 자동으로 HLSL로 변환시켜주는 과정도 구현을 완료하였다. 프로그래머는 그냥 GLSL 쉐이더 파일을 로드하는 것 처럼 사용하면 아랫단에서 자동으로 현재 사용 중인 API를 확인하고 그 API 타입에 맞는 쉐이더 언어로 변환시켜준다.    그리고 쉐이더의 Uniform Buffer ( Constant Buffer )의 offset이나 size 관련 데이터 또한 reflect하여 확인할 수 있게 구현을 하였다. 일일이 offset, size를 계산할 필요 없이 변수 명을 가지고 Uniform Buffer를 업데이트할 수 있게 구현하였다. ( 다만 이 방법은 내부 해쉬테이블을 참조해서 offset, size 데이터를 얻어와야하니 약간은 느리기 때문에 프레임마다 수 천번 업데이트해야하는 데이터의 경우 추천하지 않는다. )            
그리고 이 Reflection 데이터를 활용하여서 D3D에서 요구하는 Input Layout 또한 자동으로 생성하여 적용되게 구현하였다.                              

프로그래머는 그냥 유니티 처럼 쉐이더를 Material 클래스에 붙이고 사용하면 된다. InputLayout이나 UniformBuffer 와 같은 번거러운 일은 윗단에서는 몰라도 된다. 아랫단에서 다 자동화를 시켜서 사용 중인 API에 맞게 필요한 동작을 수행하게 구현하였다.                

아쉽게도 HLSL -> GLSL로의 변환은 불가능해보인다.         

그런데 문제가 하나 있었는데 glslcc가 Column-major을 기준으로 작성한 쉐이더 코드를 맘대로 Row-major matrix의 연산으로 바꾸어 버리는 것이다. mat4x4 * vec4 연산을 vec4 * mat4x4 연산으로 바꾸어버린다. 이렇게 되면 엔진에서 Matrix 데이터를 쉐이더로 넘길 때 Transpose를 일일이 해주어야한다..... 기존 수학 라이브러리는 Column-major로 작성되어 있고 hlsl도 column-major가 기본 값이기 때문에 그냥 사용하면 될 것이라 생각했는데 이 glslcc 요놈이 맘대로 Matrix convention을 바꾸어버린 것이다. glslccc단에서는 D3D가 HLSL에서는 Column-major을 사용하고, D3D API 단에서는 Row-major을 사용하니 이렇게 한 것 같은데 내가 원하는 동작은 아니다... 나는 그냥 Column-major을 그대로 사용하고 싶었다.....      

아래와 같은 OPENGL 쉐이더가 있으면
```
layout(binding = 0, std140) uniform Global
{
    mat4 viewProjection;
    mat4 model;
}

void main()
{
	gl_Position =  viewProjection * model * vec4(aPos, 1.0);
}
```

glslcc는 아래와 같은 HLSL로 변환을 한다.         

```
cbuffer Global : register(b0)
{
    row_major float4x4 _26_viewProjection : packoffset(c0);
    row_major float4x4 _32_model : packoffset(c4);
}
void vert_main()
{
    gl_Position = mul(float4(aPos, 1.0f), mul(_32_model, _26_viewProjection));
}
```                    

matrix 변수들에 row_major 옵션이 붙어 있고, 밑에 vert_main 함수에서 곱셈을 할 때도 mat4x4 * vec4 연산을 vec4 * mat4x4으로 바꾼다.        
아마 D3D11의 API들이 Row-major로 되어 있으니 쉐이더단에서도 이렇게 바꾸는 것 같은데....       
이러면 엔진단에서 matrix 데이터를 넘길 때 한번 transpose를 해야하니 매우 매우 귀찮아진다........       
.....
....작성중


문제는 또 있었다. 두 API의 좌표 시스템이 다르다는 것이다.           
Matrix 곱셈 연산은 오른손 좌표계든, 왼손 좌표계든 어차피 결과는 똑같아서 상관이 없지만, LookAt 매트릭스나, 카메라 관련 행렬을 구할 때 이는 문제가 된다.         
그래서 이러한 수확 관련 명령어 코드들 또한 런타임에 동적으로 불러오기로 결정하였다. 그냥 if, else 문으로 처리할까 고민을 하였는데 그냥 명령어 코드단에서 완전히 분리하기로 결정하였다. ( if, else으로 처리했을 때와 동적링킹을 사용하기 때문에 inlining을 하지 못하는데서 오는 성능 하락을 비교해보지 못하였다.... )            