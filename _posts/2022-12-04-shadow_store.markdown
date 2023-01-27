---
layout: post
title:  "x64 Shadow Space ( Spill space, Home space 라고도 부름)"
date:   2022-12-04
tags: [ComputerScience]
---            

x64의 기본 calling convention인 fast call에서는 함수의 파라미터를 기본적으로 레지스터로 전달함 ( 파라미터 4개까지는 레지스터로, 그 외는 스택에. )        
**Callee는 레지스터로 전달 받은 이 4개의 함수 파라미터는 스택 영역의 Shadow Space에 임시로 저장해두는데 이를 "Shadow Store"이라 한다.**          
**스택에 임시로 저장하는 이유는? 함수 내의 다른 연산을 위해서 해당 레지스터들을 사용해야하니 전달 받은 파라미터를 스택에 저장해두는 것**이다.    
또한 **해당 함수 내에서 또다른 함수를 호출하면 결국 그 함수를 호출에 파라미터 전달로 레지스터를 사용해야하니 매개변수를 미리 스택에 저장해두는 것**이다.         
이러한 Shadow Store를 위한 스택 공간은 Caller가 확보해준다.         
그리고 이 Shadow Store를 위한 스택 공간을 Free하는건 Calling Convention에 따라 Caller가 하기도, Callee가 하기도 한다.     
( ShadowStore는 Caller가 스택으로 전달하는 공간과 스택에 저장한 Return Address 사이에 위치한다. )               

[Shadow Store](https://stackoverflow.com/a/30194393)                 
[https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170](https://learn.microsoft.com/en-us/cpp/build/x64-calling-convention?view=msvc-170)         
                      
                      
그렇다고 항상 Shadow Store를 수행하는건 아니다. Optimized 빌드에서는 Shadow Store가 필요하지 않다고 판단되면 수행하지 않는다. ( Caller는 해당 함수가 Shadow Store를 수행하는지 안하는지 알 수 없으니 Shadow Store를 위한 공간은 항상 확보한다. )                          
[Shadow Store를 수행하지 않는 경우?](http://masm32.com/board/index.php?topic=9227.0)                      
        
Shadow Space는 디버깅에도 도움을 준다. 함수 내에서 코드가 얼마 정도 진행이 되면 파라미터 전달에 사용한 레지스터는 다른 연산에 사용되어버리니 복구가 안되는데, Shadow Store를 수행한 경우 스택 영역의 Shadow Space를 참조하면 전달 받았던 파라미터를 알 수 있다.        
         
       
[x64 abi, 좋은 내용 많다. 까먹을 때쯤이면 복습!](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions?view=msvc-170 )             

