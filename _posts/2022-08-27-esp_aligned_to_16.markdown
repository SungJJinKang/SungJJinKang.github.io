---
layout: post
title:  "함수 호출 전 스택 포인터를 16바이트에 Align 되게 만드는 이유"
date:   2022-08-27
tags: [ComputerScience, Recommend]
---         

함수에서 SSE 레지스터를 활용한 SIMD 명령어를 수행할 가능성이 있으니 함수 호출 전 SSE 레지스터 사이즈에 Align 시켜주는 것.                        
스택 포인터 레지스터와 -16(1111 0000)을 and 연산하여 하위 4비트를 버려서 16 바이트에 Align 하게 만들기도 한다. ( 스택은 높은 주소에서 낮은 주소롤 확장되니 결과적으로 잉여의 스택 공간을 더 확보하는 것이 된다. )                                  
                    
                                    
[https://stackoverflow.com/a/4175400](https://stackoverflow.com/a/4175400),                 
[https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=msvc-170](https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=msvc-170),                     
[https://stackoverflow.com/a/33868627](https://stackoverflow.com/a/33868627),             
[https://youtu.be/cEnpeDMAw_Y](https://youtu.be/cEnpeDMAw_Y),           
[https://youtu.be/D83qM9D2I3E](https://youtu.be/D83qM9D2I3E)                         
 