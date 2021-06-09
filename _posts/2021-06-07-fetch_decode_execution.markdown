---
layout: post
title:  "Fetch-Decode-Execute Instruction Cycle"
date:   2021-06-09
categories: ComputerScience
---

흔히 Fetch-Decode-Execute 사이클로 알려진 명렁어 사이클은 CPU가 컴퓨터가 켜지고 꺼질 때 까지 따르는 사이클이다.      
세개의 단계로 구성되어 있는데 fetch, decode, execute로 구성되어 있다.       

이러한 3개의 단계는 현대 CPU에 와서는 "CPU 파이프라이닝" 덕분에 3개의 단계가 동시에 실행이 된다.      

우선 CPU 3 사이클에 대해 공부하기 전에 기본적으로 CPU가 어떻게 구성되어 있는지를 알아야한다.     

**CPU의 기본 구성 요소**            

**프로그램 카운트(PC)**는 다음에 실행할 명령어의 메모리 주소를 가지고 있는 특별한 레지스터이다. **Fetch 단계** 동안 프로그램 카운터에 저장된 주소는 **메모리 주소 레지스터(MAR)**에 복사가 되고 프로그램 카운터는 하나 증가하여 그 다음에 실행할 명령어 주소를 가지게 된다.     
그 후 CPU는 메모리 주소 레지스터가 가지고 있는 메모리 주소의 위치로 가서 실행할 명령어를 얻는다. 그리고 그 명령어를 **메모리 데이터 레지스터(MDR)**로 복사한다. 이 메모리 데이터 레지스터는 메모리에서 전송 받은 데이터(여기서 말하는 데이터는 말 그대로 숫자와 같은 데이터일 수도 있고 명령어일 수도 있다)를 가지고 있기도 하고, 메모리에 저장이 되기를 기다리는 데이터를 가지고 있기도하다(메모리로 부터 데이터를 받기도 메모리에 저장을 하기도 하는 양방향 레지스터이다).      
최종적으로는 메모리 데이터 레지스터에 저장되어 있는 명령어는 **현재 명령어 레지스터(CIR)**로 복사되는데 이 현재 명렁어 레지스터는 메모리로부터 이제 막 전송된 명령어를 임시로 보관하는 장소와 같은 역할을 한다.        

Decode 단계에서는 **컨트롤 유닛(CU)**이 현재 명령어 레지스터에 보관되어 있는 명령어를 Decode(해석)한다. 이 컨트롤 유닛은 CPU 내의 **산술 논리 장치(ALU)**나 **부동 소수점 장치(FPU)** 장치로 시그널을 전송한다. 산술 논리 장치는 더하기 빼기, 반복해서 더하여 곱하거나 반복해서 빼서 나누는 것과 같은 산술 연산을 수행하는 장치이다. 산술 논리 장치는 또한 AND, OR, NOT, Bit Shift와 같은 논리 연산도 수행한다. 부동 소수점 장치는 부동 소수점 연산을 하는 장치이다.      

**3가지 단계에 대한 설명**         

컴퓨터들은 다른 명령어 세트를 기반으로 하여 서로 다른 명령어 사이클을 가질 수는 있지만 대부분 아래 설명할 사이클을 기반으로 작동한다.       

**Fetch 단계** :        
프로그램 카운터로 부터 다음 실행할 명령어가 저장되어 있는 메모리 주소를 통해 다음에 실행할 명령어를 가져와서 현재 명령어 레지스터(CIR)에 저장한다.( 당연히 이 동작 후 프로그램 카운터는 다음에 실행될 명령어의 주소를 가지게 된다.)         

**Decode 단계** :       
Decode 단계에서는 현재 명령어 레지스터(CIR)에 저장되어 있는 인코딩된 명령어를 디코더로 해석을 한다. 

**유효한 주소 읽기 단계** :    
이 단계를 이해하기 위해서는 주소 모드에 대한 이해가 필요하다.            
[이 글](https://sungjjinkang.github.io/ComputerScience/2021/06/09/addressing_mode.html)을 읽어보기 바란다.         

디코딩 된 명령어가 메모리 명령어(메모리와 레지스터 사이의 데이터를 읽거나 쓰는 명령어)라면 Execute 단계는 다음 CPU 클락에 수행된다.    
명령어가 간접 주소를 가지고 있는 경우 유효한 주소가 메인 메모리로부터 읽어지고 요구되는 데이터는 메모리로부터 읽어서 메모리 데이터 레지스터(MDR)에 저장된다.      
만약 명령어가 직접 주소인 경우 이 단계(유요한 주소 읽기)에서는 아무것도 하지 않는다.     
만약 이것이 IO 명령어이거나 레지스터 명령어인 경우 현재 CPU 클락에서 수행이 된다.     

**Execute 단계** :           
CPU의 컨트롤 유닛(CU)는 디코드 된 명령어를 컨트롤 시그널로 관련된 CPU 유닛으로 전송하는데 이는 명령어에 의해 요구되는 행동을 수행하기 위함이다. 예를 들면 레지스터에서 값 읽기, 다시 레지스터에 값 쓰기, 수학 연산을 위해 ALU에 전송을 하고 다시 레지스터 쓰기와 같은 것들이 있다. 만약 ALU가 연관이 된 명령어인 경우 ALU는 다시 컨트롤 유닛에 상태 시그널을 전송한다. 그 연산으로부터의 결과는 메인 메모리에 저장되거나 외부 장치로 보내진다. ALU로부터의 피트백을 바탁으로 프로그램 카운터(PC)는 다음으로 Fetch 되어야할 명령어의 주소를 업데이트한다.       

**위와 같은 4가지 단계가 계속 반복된다.**      



**Decode 단계에 대한 자세한 설명** :      

현대 CPU는 내부적으로는 마이크로코드라고 불리는 많은 명령어를 실행한다.        
로우 레벨 명령어로 쓰여진 이 마이크로코드는 하이 레벨 명령어를 수행하기 위해 사용된다.          
우리가 흔히 어셈블러를 통해 보는 명령어들은 실은 여러 마이크로코드들을 묶어둔(추상화한) 하이 레벨 명령어이다.            


Modern Intel CPUs actually implement many instructions using so-called microcode. Microcode consists of code written in a simpler low-level instruction set used to implement high-level instructions (for example, a rep-prefixed instruction might be implemented as a microcoded loop). Because this effectively requires the CPU itself to "compile" your input instruction stream into microcode, one can imagine that it is this microcode that is being cached (to avoid the overhead of repeatedly compiling it).

Of course, the precise details of caching "decoded" instructions varies greatly by processor, so no general statement is possible.


인코딩 된 명령어 ? 디코딩된 명령어?
decode 단계에서는 구체적으로 어떤 것을 해석해서 어떤 것을 도출하는 과정인가?     

references : [https://en.wikipedia.org/wiki/Instruction_cycle](https://en.wikipedia.org/wiki/Instruction_cycle), [https://stackoverflow.com/questions/30061932/what-is-the-decoded-form-of-an-instruction](https://stackoverflow.com/questions/30061932/what-is-the-decoded-form-of-an-instruction) , [https://en.wikipedia.org/wiki/Microcode](https://en.wikipedia.org/wiki/Microcode)