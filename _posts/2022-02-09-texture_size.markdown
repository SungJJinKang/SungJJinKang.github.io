---
layout: post
title:  "텍스쳐 사이즈와 렌더링 성능의 연관성"
date:   2022-02-09
categories: ComputerScience ComputerGraphics
---

텍스쳐 사이즈가 크다는 말은 GPU가 텍스쳐를 읽을 때 그만큼 여러 캐시 라인에 걸쳐서 텍스쳐를 읽어야한다는 것을 의미한다.          

샘플링 과정에서 일반적으로 GPU는 목표로하는 Fragment 주위 4, 8개 ( 샘플링 정책에 따라 다르다 )의 텍셀을 읽어온다. 만약 텍스쳐 사이즈가 크다면 주위 4개의 텍셀이 넓은 메모리 영역에 퍼져있다는 의미고 그만큼 텍셀을 읽어오는데 더 많은 시간이 소모된다. ( GPU는 엄청난 수의 코어를 가지고 있고, 그만큼 레지스터가 차지하는 사이즈가 크다. 그 때문에 캐시의 사이즈가 매우 작기 때문에 Cache miss 가능성이 매우 높고 Cache 정책 또한 Write through 정책을 사용하기 때문에 메모리에서 데이터를 읽는 속도가 느리다. )                               
그래서 mipmap을 통해서 적절한 텍스쳐 사이즈를 선택하면 그만큼 샘플링 과정에서 읽어오는 텍셀 데이터들이 인접해 있을 가능성이 높고 데이터를 읽어오는 시간도 더 적게 소모된다.          

텍스쳐 샘플링과는 상관없다. 텍스쳐가 크든, 작든 샘플링 과정에서 필요한 텍셀의 개수는 항상 동일하다.            




references : [https://stackoverflow.com/questions/25287807/scaling-and-performance](https://stackoverflow.com/questions/25287807/scaling-and-performance)  ,  [https://www.reddit.com/r/opengl/comments/bjk2ib/how_do_higher_resolution_textures_effect/](https://www.reddit.com/r/opengl/comments/bjk2ib/how_do_higher_resolution_textures_effect/)  ,  [https://developer.download.nvidia.com/books/HTML/gpugems/gpugems_ch28.html](https://developer.download.nvidia.com/books/HTML/gpugems/gpugems_ch28.html)                       