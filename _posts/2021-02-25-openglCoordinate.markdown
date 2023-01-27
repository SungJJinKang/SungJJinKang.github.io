---
layout: post
title:  "Open GL Coordinate 정리"
date:   2021-02-25
tags: [OpenGL, ComputerGraphics]
---

Open GL은 메쉬를 그리기 위해 3D모델의 local vertex position들을 여러번 변환하여 카메라에 그려준다. 그 과정에서 다양한 Coordinate들이 발생하는 데 그 종류와 변환 방법을 알아보자.     

----------------------------         

Model Position : Model Matrix * Local Vertex Position     
Model Position은 흔히들 말하는 World Position이다. 전체 월드 내에 이 좌표의 위치를 나타낸다.      
Opengl은 Right Hand Rule을 채용하여 z가 유니티와는 반대다.   


View Position : View Matrix * Model Position      
드디어 우리 눈이 보는 시각으로 들어왔다.      
여기서 부터는 -1과 1 사이의 값들만이 우리가 화면상으로 볼 수가 있다.        

Clip Position : Projection Matrix * View Position         
여기서는 projection matrix의 종류에 따라 2D이미지를 만들 수도 아님 현실감 있는 원근감을 표현할 수도 있다.       
그리고 near, far plane 밖의 vertex는 -1과 1의 범위를 벗어나게 되어 화면상에 보이지 않게 된다.       


NDC : Clip Position / Clip Position's w value     
Normalized Device Coordinate로 -1 부터 1 까지의 값을 가진다.      
위에서 Opengl은 Right Hand Rule을 사용하여 z가 유니티랑 반대랑 하였는 데 NDC에서는 다시 좌표값이 Left Hand Rule로 돌아온다.     
이 NDC의 z값이 0 ~ 1로 맵핑 되어 Depth Testing에 사용된다. ( 물론 중간에 Lerp하게 바꿔주는 과정이 있다. )       


Screen Position :     
Screen Positon = ( NDC + 1 ) / 2       
Screen Positon.x *= View Port Width      
Screen Positon.y *= View Port Height    
Vertex의 좌표들이 Screen 상에 좌표로 치환되었다.         
주의할 점은 Screen Position은 Top-Left를 중점으로 한다.         
그래서 y값의 경우 Screen의 Top 부분에서 0의 값을 가진다.       