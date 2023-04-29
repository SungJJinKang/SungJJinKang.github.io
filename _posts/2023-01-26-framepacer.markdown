---
layout: post
title:  "Frame Pacing"
date:   2023-01-26
tags: [ComputerGraphics, Recommend]
---            

[Frame Pacing for Mobile Devices](https://docs.unrealengine.com/5.1/en-US/frame-pacing-for-mobile-devices-in-unreal-engine/),        
[Frame Pacing Library](https://developer.android.com/games/sdk/frame-pacing)              
         
두 번째 링크는 안드로이드 공식 문서인데 Frame Pacing이 필요한 이유, 원리가 아주 자세히 설명되어 있다.            
          
대표적으로 한 가지 케이스만 보면...       
         
![framepacing1](https://user-images.githubusercontent.com/33873804/214901591-02b8ae8f-0894-4159-82e3-4c0c8f3647a4.png)           
위의 사진을 보자..           
맨 위 숫자는 기기의 화면이 갱신됨을 의미한다.           
60hz 기기에서 30hz로 렌더링을 처리하면 위의 사진과 같이 게임이 한 프레임을 처리하는 동안 기기의 화면은 2번 전환된다.(모니터 주사율의 개념)               
"Render C"가 빨리 처리되어 다른 프레임보다 일찍 프레임 버퍼를 제출했다.           
이로 인해 2번째 프레임(Render B)에서 제출한 프레임 버퍼는 화면에 한번만 표시(주사)되게 된다.(60hz 기기에서 30hz로 렌더링을 하면 한 프레임의 결과물을 화면이 2번 주사되는 동안 사용되는 것이 정상이다.)                   
이 영향으로 세 번째 프레임(Render C)에서 제출한 프레임 버퍼는 화면이 3번 주사되는 동안 사용되었다.(새로운 버퍼가 제출되지 않으면 이전 프레임의 프레임 버퍼를 화면에 보여준다.)              
그리고 이후 네 번째 프레임(Render D)의 프레임 버퍼가 주사된다.             
             
결과적으로는... 한 프레임 당 화면에 2번 주사되지 못하고. 세 번째 프레임의 결과물이 화면에 세 번 주사되게 되었다. 한 화면이 길게 화면에 표시되게 된 것이다..            
이는 유저에게 버벅임으로 느껴질 것이다. 다른 프레임보다 특정 프레임이 화면에 길게 표시되니 말이다.....      
세 번째 프레임이 비정상적으로 길게 표시되니 Frame Time 수치도 증가한다.              

Frame Pacing은 이를 개선해준다.           
![framepacing2](https://user-images.githubusercontent.com/33873804/214901582-3ddbe14e-a97b-4705-8b36-36faa9ebb484.png)            
위의 사진을 보자.....            
세 번째(Render C)의 결과물(프레임 버퍼)를 늦게 제출함으로서 두, 세 번째 프레임의 결과물이 적절하게 화면에 2번씩 주사되는 것을 볼 수 있다.        
