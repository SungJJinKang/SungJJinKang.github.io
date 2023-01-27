---
layout: post
title:  "GPU는 어떻게 Mip-map 레벨을 정하나?"
date:   2022-02-08
tags: [ComputerGraphics]
---

아주 아주 간단하다. 그냥 삼각형을 가지고 화면상에 점 하나를 그려보면 된다.           
그럼 그 점 ( Fragment )가 몇 개의 텍스쳐의 텍셀에 걸쳐있는지를 보면 된다.                  
화면 상에 Fragment 하나가 텍스쳐의 텍셀 한개와 사이즈가 딱 일치한다? -> 지금의 Mip-map 레벨이 최적이다.       
화면 상에 Fragment 하나가 텍스쳐의 텍셀 여러개를 포함한다? -> 필요한 텍스쳐의 해상도보다 더 높은, 고품질의 해상도를 사용하고 있다. 더 적은 해상도의 텍스쳐를 사용해도 괜찮다.          

references : [https://docs.tizen.org/application/native/guides/graphics/texturing/#filtering-for-minification](https://docs.tizen.org/application/native/guides/graphics/texturing/#filtering-for-minification), [https://paroj.github.io/gltut/Texturing/Tut15%20How%20Mipmapping%20Works.html](https://paroj.github.io/gltut/Texturing/Tut15%20How%20Mipmapping%20Works.html)                    
