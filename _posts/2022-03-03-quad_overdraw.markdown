---
layout: post
title:  "Quad Overdraw ( Over Shading ) ( 한 Fragment 보다 작은 삼각형이 성능면에서 문제가 되는 이유 )"
date:   2022-03-03
categories: ComputerGraphics ComputerScience
---

![viewmode_quad_overdraw](https://user-images.githubusercontent.com/33873804/156515043-c367689f-b287-4735-9e64-90195cd3192f.jpg)        

이런 사진을 어디서 본적이 있는가?       
이 사진은 언리얼 엔진 4의 "Quad Overdraw" 뷰 모드이다.              

**"Quad Overdraw"는 OverDraw의 일종으로 한 "Qaud"보다 더 작은 삼각형이 OverDraw 되는 것**을 말한다.                     
**GPU는 픽셀 쉐이딩 연산을 수행할 때 한 Fragment씩 연산을 하지 않는다. 대걔 인접한 2x2 혹은 8x8의 Fragment씩 "Quad" 단위로 연산을 한꺼번에 수행**한다.               
그 후 Quad 내의 Fragment 중 **불필요한 Fragment는 버린다.**                     
**불필요한 Fragment가 많아진다는 것**은 그 만큼 불필요한 연산량이 늘어나는 것이고 **연산시 효율이 안좋아진다.**               
            
**"삼각형의 개수가 문제가 아니라"**, Screen Space에서의 **"삼각형의 크기가 문제인 것"**이다!!              
삼각형의 크기가 작을 수록 더 많은 불필요한 연산을 한다는 것이다.            
여기서 오버드로우까지 발생한다고 생각을 해보아라. 상상하기도 싫다....       

( 요즘 나오는 게임들 ( 특히 언리얼 엔진5 )만 보아도 삼각형의 개수가 정말 셀 수 없이 많아진다. 그렇기 때문에 거리가 조금만 멀어져도 삼각형이 한 Fragment보다도 작을 수 있다. 이러한 삼각형들이 오버드로우 된다면 버려지는 픽셀 연산량은 ( 오버드로우로 인해, 삼각형이 작아 Quad 단위 연산시 버려지는 Fragment 연산 )은 상상할 수 없을만큼 커질 것이다. )                            

흔히들 사용하는 **LOD**도 "삼각형의 개수를 줄이는 것이 목적이 아닌", **Screen Space에서의 삼각형의 크기(!!)를 키우려는 것이 목적**이다!!              
              
-----------------------------          
              
근데 사실 요즘 하드웨어에서 오버드로우는 큰 문제가 되지 않는다..........               
모바일 환경에서는 문제가 될 수도....             
                      
                  
reference : [https://unrealartoptimization.github.io/book/pipelines/pixel/](https://unrealartoptimization.github.io/book/pipelines/pixel/)                       