---
layout: post
title:  "언리얼 엔진 PreShader ( Constant Folding )"
date:   2022-11-08
tags: [UE]
---          
                 

PreShader~는 CPU에서 연산해서 GPU로 넘겨주는 데이터이다.              
WriteNumberOpcodes로 op 코드 작성해주면 이 op코드를 cpu에서 연산 후 gpu로 넘겨준다.                    
FMaterialUniformExpressionFoldedMath도 cpu에서 연산을 끝낸다.             

![preshader](https://user-images.githubusercontent.com/33873804/200552828-5eb24536-cdf9-4e21-9314-ddafb8855f2a.PNG)               
```cpp
"AddUniformExpressionWithHash(Hash, new FMaterialUniformExpressionFoldedMath(GetParameterUniformExpression(A),GetParameterUniformExpression(B),FMO_Add), bIsFullPrecision ? ConvertMaterialValueTypeToFullPrecisionIfFloatType(GetArithmeticResultType(A, B)) : GetArithmeticResultType(A, B), TEXT("(%s + %s)"), *GetParameterCode(A), *GetParameterCode(B));"                   
```              
파라미터 A, B가 유니폼 Expression ( 유니폼 버퍼, 상수 버퍼 )이다.        
드로우콜 전에 GPU로 전송하여 드로우콜 동안 변경되지 않는다.          
이러한 유니폼 Expression은 CPU에서 최대한 연산을 한 후 GPU로 전송한다.
CPU에서 덧샘(FMO_Add) 연산을 끝내고 최종 결과를 Uniform 버퍼로 전달해주는 것이다.              
                 
EvaluatePreshader 함수에서 op코드들에 대한 연산을 수행.                   
<img width="582" alt="EvaluatePreShader1" src="https://user-images.githubusercontent.com/33873804/200570943-6233b0f8-43a2-4603-8c9d-ffaacb54aa15.png">              
                 
FillUniformBuffer 함수도 참고.            
          
![constant folding](https://user-images.githubusercontent.com/33873804/200552831-511c5d1d-c540-45eb-8733-255343273769.PNG)               

매터리얼 노드에서 상수로 넣어준 값뿐만아닐 MaterialParameter ( 유니폼 버퍼 )도 어쨌든 CPU에서 GPU로 넘겨줄 때 상수 값으로 넘겨주니 마찬가지로 PreShader로 처리된다.
       

references : [https://udn.unrealengine.com/s/question/0D54z000076GC2rCAG/constants-folding](https://udn.unrealengine.com/s/question/0D54z000076GC2rCAG/constants-folding)