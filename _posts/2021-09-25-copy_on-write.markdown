---
layout: post
title:  "Copy-on-write"
date:   2021-09-25
tags: [ComputerScience, Recommend]
---

Copy-on-write는 간단하게 말하면 어떤 데이터를 복사해야될 일이 생기는 경우 실제로 복사를 하지는 않고 기존의 데이터를 레퍼런스하고 있다고 데이터의 데이터를 변경해야할 경우가 되서야 비로소 실제로 데이터를 복사하는 것이다. 이를 통해 데이터 복사가 필요한 경우 실제로 복사를 하지 않고 자원 낭비를 막을 수 있다.                       
프로그래밍의 개념에서 보면 배열을 복사하지 않고 포인터로 기존 배열을 가리키고 있다가 배열의 데이터를 변경해야할 때가 되서야 비로소 실제로 데이터를 복사해 새로운 배열을 만드는 것이다.        

이러한 Copy-on-write는 페이징 기법에서 자주 쓰이는데 기존의 프로세스로부터 새로운 프로세스를 만들 떄 ( fork ) 기존 프로세스의 Physical Memory를 복사하기 위해 **새로운 프로세스에 Physical Memory를 새롭게 할당하지 않고 새로운 프로세스의 페이지 테이블에서 페이지들을 기존 프로세스의 Physical Memory 페이지를 참조**하게 만든다.              
이를 구현하기 위해서 페이지 테이블의 페이지들에 읽기 전용 마킹을 하고, 해당 Physcial 페이지 ( 기존 프로세스의 페이지 )를 참조하고 있는 프로세스 ( 프로그램 )의 수를 센다.      
만약 어떤 데이터가 이 페이지에 새롭게 쓰이게 되면 ( 기존 주인 프로세스든 다른 프로세스든 ), 커널은 이 쓰기 시도를 가로채고 기존 페이지의 데이터와 함께 새로운 Physcial 페이지를 할당한다. ( 기존 주인 프로세스가 이 새로운 Physical 페이지를 가질 것이다. ) ( 물론 기존 페이지를 참조하는 프로세스가 한 개라면 ( 주인 프로세스 ) Physical 페이지 할당은 하지 않는다. ) 이후 커널은 새로운 Physical 페이지로 기존 주인 프로세스의 페이지 테이블을 업데이트하고 원래 Physcial 페이지의 참조 카운트를 하나 줄인다. 그 후 원래하려던 쓰기 연산을 수행한다. 이렇게 새롭게 Physical 페이지를 할당함으로서 기존 페이지를 참조하던 다른 프로세스들은 쓰기 동작으로 인한 데이터 변화에서 무관하게된다.        

이러한 Copy-on-write 기법은 0으로 채워진 Physical 페이지를 가져 효율적인 메모리 할당을 지원하는데도 사용된다.         
프로세스가 새롭게 메모리를 할당할 때 (새로운 Logical 페이지)들은 실제로는 (모두 0으로 채워져 있고 copy-on-write로 마킹된 모든 프로세스가 공유하는 Physical 페이지)를 참조한다. ( Logical 페이지가 해당 프로세스 전용 Physcial 페이지를 참조하지 않는다는 것이다. ) 프로세스가 실제로 데이터를 쓸 때까지는 할당받은 Locgical 페이지들은 사실은 모든 프로세스가 공유하는 ( 0으로 채워지고 Copy-on-write로 마킹된 ) Physical 메모리를 참조하고 있는 것이다. 그 후 데이터를 쓸 때 제대로된 프로세스 전용 Physical 페이지를 가진다. ( 참조한다. )                        
이를 통해 Physical 메모리 사이즈보다 더 많은 가상 메모리를 예약할 수 잇고 메모리를 광범위하게 사용할 수 있게된다.         

이러한 Copy-on-write 기법은 Dynamic 링킹에도 사용되는데 일반적으로 dll 파일의 데이터, 코드는 OS에서 관리되며 여러 프로세스가 동일한 dll을 사용할 경우 메모리에 프로세스 수 만큼 dll 데이터를 올리는 것이 아니라 메모리에는 한번만 올려두고 프로세스들은 이것을 참조한다.          
그런데 만약 어떤 프로세스가 dll 데이터를 수정하려하면 어떻게될까? 이때 copy-on-write가 사용된다. 수정하려는 프로세스는 이제서야 비로소 새롭게 메모리 영역을 할당받아 dll 데이터를 복사한다.        

--------------

이 글의 내용이 약간 부정확할 수도 있다. 정확한 내용은 [이 글](https://sungjjinkang.github.io/Window_Memory)을 읽으라.        