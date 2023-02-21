---
layout: post
title:  "자체 엔진 최적화 - 1 (작성 중)"
date:   2023-02-19
tags: [InHouseEngine]
---          
                           
"Superluminal Performance" 프로파일링 결과.          

![1](https://user-images.githubusercontent.com/33873804/219942395-d7354d9b-d818-4d5e-b80a-68121697dc75.png)
![2](https://user-images.githubusercontent.com/33873804/219942390-0da5ac49-9846-4049-9cb8-358863611862.png)
![3](https://user-images.githubusercontent.com/33873804/219942392-65b64d29-9696-4a43-a4a7-312f21bbedb4.png)
![4](https://user-images.githubusercontent.com/33873804/219942393-6f68a0ab-f443-4eef-b0ea-be4699b33382.png)
            
모델 매트릭스 연산에서 가장 많은 시간이 소요되는 중.             
테스트 씬 자체가 오브젝트(엔티티) 수가 매우 많아(4000개 가량) 모델 매트릭스 연산에 시간이 많이 소요되는 중.             

--------------------         

컬링(프러스텀 컬링, 오클루전 컬링, Distance 컬링)을 병렬로 처리 중인데, 여전히 비용이 큼.             
컬리을 Async로 처리(Job 스레드에서 처리)해보자. 현재 프레임의 컬링 결과를 다음 프레임에 읽어 사용하자.          
오브젝트 Sorting(가까운 오브젝트 먼저 그림)도 Async로 처리해서 다음 프레임에 읽어 사용하도록 처리해보자.             
