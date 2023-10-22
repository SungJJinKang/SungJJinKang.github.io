---
layout: post
title:  "Android GPU Counter"
date:   2022-09-14
tags: [ComputerGraphics]
---         
        
Mali 계열 GPU는 [HWCPIPE](https://github.com/ARM-software/HWCPipe)를 사용해왔는데, Adreno 계열 GPU에 대해서는 GPU Counter를 가져올 방법이 없었는데 이를 구현한 듯 ( Mali, Adreno 둘 다 지원하는 ) 보이는 [라이브러리](https://github.com/google/hardware-perfcounter)가 있어 가져옴.                   

아직 사용해보지는 않아서 잘 되는지는 모름.        
                            
references : [https://android.googlesource.com/kernel/msm/+/android-msm-marlin-3.18-nougat-dr1/drivers/gpu/msm/adreno_perfcounter.c](https://android.googlesource.com/kernel/msm/+/android-msm-marlin-3.18-nougat-dr1/drivers/gpu/msm/adreno_perfcounter.c)