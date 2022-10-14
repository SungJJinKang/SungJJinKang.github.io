---
layout: post
title:  "Scalar Replacement of Aggregates"
date:   2022-10-14
categories: ComputerScience
---         
               
대충 설명하면 구조체나 Array 같은 Aggregate들의 멤버, Element 각각을 별개의 Scalar 데이터로 본다.         
이를 통해 각각의 Scalar 데이터(로 보기로 약속한)들에 대해 mem2reg(스택 할당 안하고 레지스터에 바로 쓴다)와 같은 최적화를 도모한다.            
         
"The well-known scalar replacement of aggregates transformation. This transform breaks up alloca instructions of aggregate type (structure or array) into individual alloca instructions for each member if possible. Then, if possible, it transforms the individual alloca instructions into nice clean scalar SSA form."
       
![2 +Scalar+replacement+of+aggregates](https://user-images.githubusercontent.com/33873804/195855156-39dcf6d7-a59f-41b1-8cdd-b876445a7377.jpg)

-------------------

<img width="638" alt="20221014221837" src="https://user-images.githubusercontent.com/33873804/195856837-b70d75eb-7694-4def-b157-b2209e53b04b.png">      
