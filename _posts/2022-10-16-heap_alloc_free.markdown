---
layout: post
title:  "힙 할당, 해제 구현(glibc)"
date:   2022-10-16
categories: ComputerScience
---         

[https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/](https://azeria-labs.com/heap-exploitation-part-1-understanding-the-glibc-heap-implementation/),              
[https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/](https://azeria-labs.com/heap-exploitation-part-2-glibc-heap-free-bins/),                           
[동적 할당 메타 데이터 구조](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob;f=malloc/malloc.c;h=6e766d11bc85b6480fa5c9f2a76559f8acf9deb5;hb=HEAD#l1059)            
           
<img width="611" alt="mallocfree1" src="https://user-images.githubusercontent.com/33873804/196029468-e99ddeef-001a-471f-89b5-54d5fcba4f59.png">                      
<img width="611" alt="mallofree2" src="https://user-images.githubusercontent.com/33873804/196029470-cddb2fa6-c7cd-4a48-95b5-beb86f728fe8.png">
