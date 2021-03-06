---
layout: post
title:  "constexpr의 definition이 헤더에 있어야하는 이유"
date:   2021-03-07
categories: C++
---

매우 매우 간단하다.          

constexpr은 컴파일 타임에 결정된다는 의미다.      
그럼 만약 헤더에 구현이 안되어 있으면 그 constexpr을 보는 다른 translation unit(cpp)들은 어떻게 컴파일을 할 수 있겠나???       
constexpr은 컴파일 타임에 값이 알려져 있어야 한다는 것은 링커 단계 전에 그 값을 알아야한다는 것이므로 definition을 다른 translation unit이 볼 수 있어야 한다.   

