---
layout: post
title:  "언리얼 엔진 매터리얼 에디터에서의 플랫폼 통계"
date:   2022-11-08
categories: ComputerScience ComputerGraphics UE UnrealEngine
---            
               
언리얼 엔진 매터리얼 에디터에서 플랫폼 통계에 나오는 명령어 카운트는 실제 머신에서 실행되는 명령어 카운트와 다르다.            
                  
**OpenGL**쪽 Mali 오프라인 컴파일러의 경우 GPU 종류에 따라 최종 명령어 개수가 다르다. 기기에서 쉐이더 컴파일 과정에서 드라이버가 해당 기기 GPU의 명령어 구성, 하드웨어적 기능과 관련해서 최적화를 한번 더 하기 때문이다.                          
그래서 Mali 오프라인 컴파일러는 최종적으로 기기에서 실행되는 명령어 카운트 수를 GPU 종류 별로 보여준다. 또한 GPU의 Warp 구조 상 분기에 따라 Best Path ( 한쪽 분기가 탔을 때 ), Worst Path ( 양쪽 분기 다 타서 연산 비용이 들 때 ) 각각의 명령어 카운트를 보여준다.                  
                  
**Metal**의 경우 언리얼 엔진에서 보여주는 명령어 카운트가 실제 기기에서 실행되는 것과 완전히 다르다.                  
Metal의 경우 쉐이더 코드의 명령어 카운트를 보여주는 API를 애플쪽에서 제공해주지 않기 때문에 언리얼 상에서 보여지는 명령어 카운트는 언리얼쪽에서 임의로 계산한 값이다.            
                    
아래는 UDN의 댓글에서 확인한 내용.                  
"Last but not least, the "instructions" mentioned in above statement is only a rough estimate and does not refer to actual shader code instructions. This is a backend dependent estimate; the Metal backend for instance uses the number of source lines of the resulting Metal shader to report this value since there is no public API to work with the Metal IR."                    


references : [https://udn.unrealengine.com/s/question/0D54z000076GC2rCAG/constants-folding](https://udn.unrealengine.com/s/question/0D54z000076GC2rCAG/constants-folding)
