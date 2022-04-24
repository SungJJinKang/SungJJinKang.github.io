---
layout: post
title:  "Avoid using depth prepasses ( ARM Mali GPU )"
date:   2022-04-24
categories: ComputerScience ComputerGraphics
---

Mali GPU에서는 Depth Pre Pass를 수행하지 마라?          
Mali GPU는 기본적으로 하드웨어단에서 Hidden Surface Removal를 수행하기 때문에 Depth Pre Pass를 수행하면 오히려 성능적으로 손해일 수 있다.             

refererence : [https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses](https://developer.arm.com/documentation/101897/0200/optimizing-application-logic/avoid-using-depth-prepasses)           