---
layout: post
title:  "__declspec(dllexport) vs extern ( 번외로 어떻게 런타임에 export된 함수를 찾을까? )"
date:   2022-09-15
categories: ComputerScience
---         
                                 
갑자기 헷갈려서 기록해두는 글...                 
                         
**""__declspec(dllimport)"는 함수나, 변수, 타입등을 외부에 노출 시킬지 여부를 결정한다.**. 이와 관련해서는 [이 영상](https://youtu.be/dOfucXtyEsU)에서 자세히 확인할 수 있다. 로더가 어떻게 export된 함수, 변수를 런타임에 링크해주는지, 동적으로 링크되는 함수를 호출했을 때 컴파일 타임에는 어떤 주소를 호출하고, 그 주소는 무엇이고, 런타임에 어떻게 진짜 함수의 명령어 코드의 위치를 찾아가는지, GOT, PLT 등등.....                  
                     
반면 "extern", "static"은 내부 링크, 외부 링크에 대한 것이다. 다른 Translation Unit에 있는 정의를 링크할 수 있는지에 대한 것이다. 이는 정적 링크 ( 컴파일 타임 링크 ), 동적 링크 ( 런타임 링크 )와는 무관한 것이다.          
            
--------------------------------        
          
번외로 어떻게 런타임에 export된 함수의 주소를 찾을까?                  
[리눅스에서는 어떻게 하나?](https://stackoverflow.com/a/38194924),                    
[윈도우에서는 어떻게 하나?](https://topic.alibabacloud.com/a/learning-windows-pe-file-learning-1-font-colorredexportfont-tables-pe-font-colorredexportfont_1_31_32673941.html)
