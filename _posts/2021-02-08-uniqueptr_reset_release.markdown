---
layout: post
title:  "std::unique_ptr의 reset과 release 차이"
date:   2021-02-08
categories: C++
---

std::unique_ptr가 소유중인 오브젝트를 파괴하고 싶을 때 reset을 써야할까? release를 사용해야할까?    
답은 reset이다. reset() 이렇게 하면된다. reset을 호출하면 unique_ptr이 소유중인 포인터의 오브젝트를 파괴한다.    
release의 경우에는 그냥 소유중인 오브젝터의 포인터를 놓아주는 것이지 그 오브젝트를 파괴하지는 않는다.