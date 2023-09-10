---
layout: post
title:  "OPTIMIZING FOR THE RADEONTM RDNA ARCHITECTURE 요약(작성 중)"
date:   2023-09-15
tags: [ComputerGraphics]
---

최근 AMD RDNA 기반 플랫폼의 최적화 작업을 하면서 RNDA 아키텍처와 관련된 여러 최적화 자료를 찾아보았다.          
                 
이 글은 [OPTIMIZING FOR THE RADEONTM RDNA ARCHITECTURE](https://gpuopen.com/wp-content/uploads/slides/GPUOpen_Let%E2%80%99sBuild2020_Optimizing%20for%20the%20Radeon%20RDNA%20Architecture.pdf)을 요약한 글입니다.       
이해를 돕기 위한 각주를 추가하였습니다. 그리고 많은 의역이 있으니 참고바랍니다.        

----------------------------------

**AMD RDNA 아키텍처에 대한 이해**                  
                  
[RDNA 아키텍처 WhitePaper](https://www.amd.com/system/files/documents/rdna-whitepaper.pdf)는 RDNA 아키텍처에 대한 풍부한 이해를 도와줍니다.       

WGP 구조 - 1        
<img class="AMD AGC Terminology1" src="/assets/images/63279f982ba057854dec6dcec6b5f0e2c09e2bc6/assets/images/WGP1.PNG"> 
               
WGP 구조 - 2        
<img class="AMD AGC Terminology1" src="/assets/images/39086252e852938531e55a5c6555cf5e.png">      
             
WGP 구조 - 3                         
<img class="AMD AGC Terminology2" src="/assets/images/9e0ca5744ebd9dafa335e140c0910562.png
">          
<img class="AMD AGC Terminology3" src="/assets/images/9ac13ed93c68577dabedf1a013e2bd1.png
">          
                  
                 
또한 아래 이미지는 NVIDIA와 AMD GCN 아키텍처의 용어들의 Equivalent를 소개합니다. ([출처](https://www.olcf.ornl.gov/wp-content/uploads/2019/10/ORNL_Application_Readiness_Workshop-AMD_GPU_Basics.pdf) 31P)          
<img class="AMD AGC Terminology" src="/assets/images/NVIDIA AMD GCN 용어.PNG">         

위에서 본 것들은 AMD RDNA 아키텍처의 일반적인 구조이고, 디테일한 부분은 GPU에 따라 다릅니다.           

----------------------------------

**AMD RDNA 아키텍처에 대한 이해**                  
                  



하나의 스레드 그룹 내 Wavefronts들은 모두 동일한 WGP에서 동작함.