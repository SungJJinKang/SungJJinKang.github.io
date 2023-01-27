---
layout: post
title:  "부동 소수점, Fast Math, ulp"
date:   2022-12-26
tags: [ComputerScience]
---            

[https://nybounce.wordpress.com/2016/06/24/ieee-754-floating-point%EB%B6%80%EB%8F%99%EC%86%8C%EC%88%98%EC%A0%90-%EC%82%B0%EC%88%A0%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC/](https://nybounce.wordpress.com/2016/06/24/ieee-754-floating-point%EB%B6%80%EB%8F%99%EC%86%8C%EC%88%98%EC%A0%90-%EC%82%B0%EC%88%A0%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC/),            
[https://junstar92.tistory.com/253](https://junstar92.tistory.com/253)           
            
ULP라는 개념도 처음 앎.           
[Metal 스펙 문서](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)의 254 페이지를 보면 각각의 연산(function)의 최소 보장 정확도가 나와있는데..                   
일부 부동 소수점 연산들의 경우 최소 보장 정확도(Accuracy)가 Correctly rounded가 아닌 N ulp 이하만 보장하면 된다고 되어 있다. 최근 UE4 Metal Half Precision 적용 작업을 하면서 일부 기기에서만 정밀도 문제로 아티팩트가 발생하는 이유를 여기를 보면 알 수 있다.           
최소 보장 정확도가 4 ulp 이하 이런 연산들의 경우 4 ulp 이하로만 보장을 해주면 되니 기기, GPU마다 연산의 결과가 다르게 나올 수 있는 것이다. 그래서 일부 기기에서만 아티팩트가 발생하는 것이다...                
          
Metal에는 Fast Math라고 NAN이나 INF가 Input으로 들어올 가능성을 배제하거나, 오버/언더 플로우를 고려하지 않고 쉐이더를 컴파일하여 성능을 더 취하는(성능 향상을 이루는) 방법이 있다. Fast Math 옵션이 켜져 있는 경우 위의 것들을 고려하지 않으니 Floating Point 연산 순서를 보장한다거나 하는 것들을 수행하지 않아도 되어 성능을 더 높인다. 자세한건 위의 Metal 스펙 문서에 다 나와있다.           
언리얼 엔진에서는 기본적으로 이 Fast Math 옵션(EnableMathOptimisations)이 Enable되어 있다.             

