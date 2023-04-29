---
layout: post
title:  "가상 메모리 주소, 메모리 페이징에 대한 나의 이해"
date:   2021-03-05
tags: [ComputerScience, Recommend]
---

우선 가상 메모리 주소를 왜 사용할까?? Physical한 메모리 주소를 사용하려면 프로그램(프로세스) 단에서 일일이 관리해야하는데 이걸 OS단에서 대신 해주어서 프로그램은 자신의 virtual address space만 신경 쓰면된다. 프로그램에서는 프로그램에 배정된 virtual address만 접근할 수 있으니 다른 프로그램의 physical address에 접근할 수 없어져서 안정성이 더 높아짐.          
그리고 가상 메모리 덕분에 한 프로그램이 굳이 Physical address 내에 데이터들을 연속되게 가질 필요없다. Physical memory에서 데이터들이 파편화 되어 있어도 어차피 우리는 연속된 virtual address만 신경쓰면 되기 때문이다.      
또한 virtual address의 사이즈가 실제 physical address보다 클 수 있는 데 이건 실제 physicla 메모리 사이즈보다 메모리를 더 많이 활용하게 해준다. 이건 후에 서술할 virtual memory(disk) 덕분이다.         
프로그램(프로세스)마다 virtual address space가 각각 자신의 것을 가지고 있는 데 이 virtual address가 같다고 physical address도 같은건 아니다.    
또한 virtual address는 연속되어 있는 것 처럼 보이지만 이 virtual address가 가리키는 physical address는 연속되어 있지 않고 파편화 되어있는 경우가 많다.(한 페이지 내의 연속된 주소는 physical한 메모리에서도 연속되어있다.)          

1. X86 OS환경 내에서는 가상 메모리 주소 32비트로 구성되어 있다. 상위 20비트는 페이지의 physical address를 찾는 데 사용하고 하위 12비트는 페이지 내의 우리가 찾는 정확한 physical address를 찾는 데 사용된다.    

2. 우선 최상위 10비트는 Page Directory의 index이다.(Page Directory는 운영체제에 따라 없어서 상위 20비트가 바로 Page Table의 index를 가리키는 경우도 있다.).     
Page Direcotry는 페이지 테이블의 base address들을 가지고 있다. 그럼 최상위 10비트를 가지고 이 Page Direcotry 내의 index를 찾아간다. 그럼 그 위치에는 페이지 테이블의 base address를 가지고 있다.     

3. 그럼 이 주소를 따라가면 페이지 테이블이 나오는 데 이 페이지 테이블은 페이지 테이블 엔트리들로 구성되어 있다. 여기서 다음 10비트가 index로 사용되어 페이지 테이블 엔트리를 찾게된다.         
이 index를 따라가면 Page Table Entry를 발견하는 데 이 Page Table Entry는 Physical 메모리 내에 페이지의 base 주소를 가진다. 이 페이지가 우리가 비로소 찾아왔던 실제 데이터들의 묶음(페이지, 4KB)이다(왜 묶음이냐? 페이징 기법이 뭔지를 찾아봐라). 드디어 Physical Memory 주소로 왔다. 현재 까지 아는 것은 Physical Memory 내의 페이지의 base address이다.     

밑에서 배우겠지만 이 Physical address에 내가 찾는 Page가 없을 수도 있다.     

4. 그리고 마지막 12비트는 이 페이지내에서의 base 주소로 부터의 offset을 가리킨다. 이 offset을 따라가면 비로소 원하는 virtual address에 대한 physical address 주소를 가질 수 있다.      
TLB라는 개념이 여기서 나오는 데 TLB는 virtual address의 상위 20비트를 가지고 찾은 페이지(페이지 프레임)의 physical address를 저장하고 이 값을 가지고 나중에 offset과 합쳐서 실제 페이지 내의 원하는 주소의 physical address를 찾는다. ( virtual address와 physical address간의 매핑 정보를 가지고 있는 일종의 캐시라고 생각하면 된다. 페이지 테이블은 메모리 상에 위치해 있으니 상대적으로 CPU 코어에 가까운(더 빠른) TLB에 캐싱을 해두는 것이다. )                 
          
TLB는 MMU의 구성 요소 중 하나이다.               
Page Table은 Physical memory(RAM)에 위치한다. ( Page Table은 page swap out되지 않는다고 한다. )                   
        
실제 page가 어떤 physical address의 주소에 존재하든 프로그래머는 신경쓸 필요없다. 그냥 virtual address만 보면 되니깐.....        

 
![24467A4E579115AB09 (1)](https://user-images.githubusercontent.com/33873804/110157365-6c43ac80-7e2b-11eb-884b-8d9e3efeb7ca.png)    
[https://introfor.tistory.com/70](https://introfor.tistory.com/70)     

------------------------------------

자 그럼 가상 메모리 주소를 어떻게 Physical한 주소로 변환할까?       

1. CPU에 있는 MMU(memory management unit)이라는 장치가 이 작업을 하는 데 우선 MMU는 MMU 내에 TLB(Translation lookaside buffer)라는 메모리 캐시를 먼저 탐색한다.    
TLB내에는 최근에 변환했던 virtual address와 그 physical address( 정확히는 Physical 메모리 상의 페이지의 Base 주소, Base 주소만 알면 virtual address의 하위 12비트 offset만 더해주면 되니.. )            
그럼 MMU는 우선 이 TLB를 먼저 탐색한다.      
만약 TLB에 원하는 virtual address가 있으면 바로 거기 저장된 physical address를 반환하면 된다. (후술하지만 정확히는 TLB가 (virtual address)와 (physical address page의 base address)를 저장하고 이 address에서 하위 offset비트로 이 엔트리에서 실제 physical memory address로 접근한다.)    
즉 MMU는 메모리의 paged memory를 관리하고 virtual address를 physical address로 변환하는 작업을 한다.           

2. 그런데 만약 TLB에 없다면 MMU는 virtual address의 상위 10비트를 가지고 메모리 내의 페이지 테이블을 확인한다. ( 메모리 상에 페이지 테이블의 위치는 PTBR ( Page Table Base Register )에 저장되어 있다. )             
그리고 중간 10비트로 페이지 테이블 내의 페이지 테이블 엔트리를 찾는다. 이 페이지 테이블 엔트리는 4KB 사이즈의 페이지의 base 주소를 가지고 있다.    
이 페이지가 비로소 우리가 찾아왔던 실제 메모리 데이터들이다. 이 4KB의 페이지 내에서 마지막 12비트를 offset으로 사용하여 페이지 내의 목표로 했던 physical address를 TLB에 반환한다.        

3. 여기서 중요한 내용이 나오는 데 원하는 페이지가 메모리 내에 없을 수도 있다.     
이게 무슨 말인가? 메모리에 없다면 어디 있나?? 바로 하드디스크와 같은 디스크에 메모리 내용을 임시로 저장해 둘 수 있다.(한 페이지가 통째로 디스크로 임시로 옮겨짐)             
virtual memory라는 개념을 들어보았을 건데 메모리의 용량이 부족하다면 컴퓨터는 메모리에서 오래 동안 사용되지 않은 페이지(4KB)를 디스크 내에 paging file에 임시로 저장해 둔다. 이를 page out이라고도 한다. ( page out되도 해당 virtual address 그냥 유지된다. virtual address는 그냥 말그대로 가상의 주소다. 현재 Physical Address에 존재하는 데이터를 가리킬수 있고 아닐 수도 있다. page out된 경우에도 당연히 virtual address는 유효함)            
이렇게 찾고자 하는 페이지가 page out되어 있는 경우를 page fault라고한다.       
이 경우에는 disk의 paging file에서 다시 메모리로 page in을 하여 데이터를 disk에 저장되어 있는 페이지를 메모리로 복사해서 가져와야한다. (페이지 단위로 저장하고 로드한다. 대개 4KB이다). ( 이렇게 page in을 하면 해당 physical address에 있던 page는 page out됨 )            
그럼 이제 디스크에서 불러온 이 페이지의 physical address를 페이지 테이블에 업데이트하고 난 후 이 페이지 주소를 반환한다. ( 여기서 TLB 존재이유가 나온다. TLB가 있으면 이렇게 복잡한 페이지를 찾는 과정을 안거쳐도 되는 것이다. )        

4. 여기서 중요한건 메모리에서 바로가져오든 page in해서 주소를 가져오든 반드시 TLB에 그 변환 관계(Virtual address -> Page Table Entry)를 기록해야한다. 이 기록을 한 후 MMU는 처음부터 TLB를 뒤진다. 그럼 당연히 방금 전에 TLB에 기록했으니 그 기록을 보고 MMU는 physical memory에 접근한다. ( TLB는 페이지의 Physical address를 저장하는 게 아니라 페이지 테이블 엔트리의 주소를 저장한다!!!! )      


페이지 테이블 엔트리가 있고 페이지 프레임이 있다.    
페이지 테이블 엔트리는 페이지 테이블 내에 하나의 페이지 virtual address와 physical address의 관계에 대한 정보를 가지고 있다. 여기서 정보로는 우선 페이지의 Physical address base주소, 여러 추가적인 bitflag등이 있다 ( 아래에 나옴 ).           

페이지 프레임은 페이지가 들어갈 수 있는 실제 Physical address 내의 프레임을 말한다. 이 페이지 프레임에는 여러 페이지가 들어올 수 있다. 왜냐면 page out된 경우에는 해당 페이지 프레임에(Physical address의 주소에) 다른 페이지가 들어올 수 있기 때문이다.       

-----------------------------

아래 사진은 페이지가 swap out되어 해당 페이지 접근시 disk io가 필요한 경우, 메인 메모리에서 바로 데이터를 읽는 경우, 캐시에서 읽는 경우의 속도 비교표이다.        
![Slide21](https://user-images.githubusercontent.com/33873804/218319244-f505d0db-e280-4233-8b5e-c912adcc71c9.png)


-----------------------------   

이 글에서는 페이지 사이즈는 4KB로 가정하였지만 Windows 같은 경우 4KB 이상의 [Large Page](https://learn.microsoft.com/en-us/windows/win32/memory/large-page-support)를 지원하기도 한다.     

-------------------------

페이지 테이블내의 각각의 페이지 테이블 엔트리는 페이지의 physical address말고도 추가적인 정보를 가지고 있는 데 그 종류는 아래와 같다.     

Reference bit(페이지에 접근이 있었는지, page out할 블록을 정할 떄 사용함. 가장 접근),      
Valid bit(해당 page의 실제 데이터가 메모리에 있는지, page out되어 하드디스크 내 paging file에 있는지),      
Dirty bit(해당 페이지의 메모리 데이터가 page in된 이후 데이터가 변하였는지, 변한 경우 다시 page out될때 data들을 disk의 paging file에 copy해줘야한다, 달라진게 없으면 굳이 paging file에 메모리 데이터 내용을 안 써줘도 된다),       
Process ID information(예전에는 모든 프로세스가 하나의 페이지 테이블을 가져 각기 다른 프로그램(프로세스)가 같은 physical address에 접근하려 할 때 이를 구분해주기 위해 프로세스의 ID를 기록하였다. 근데 최근에는 그냥 프로세스마다 자신만의 페이지 테이블을 메모리에 가지고 있다. 이 bit는 잘 안쓰인다.)          

이 페이지 테이블도 여러 종류가 있는데 그건 아래 주소에서 확인하자. 참고로 x86은 Multi-level이라는 방식을 사용한다.       
[페이지 테이블 종류](https://en.wikipedia.org/wiki/Page_table#Page_table_types)


---------------------------      

페이징 기법란???       

페이징 기업은 메모리를 일정한 크기의 페이지들로 나누어서 메모리를 관리하는 기법이다. 페이징은 데이터로의 빠른 접근을 위해 사용된다.     
프로그램이 페이지를 필요로 할 때 OS는 특정한 개수의 페이지들을 저장장치(HDD, SDD)에서 메인메모리로 복사하기 때문에 메모리에서 이용가능하다.


What is paging? Why paging is used?       
 
Paging is a memory management technique in which the memory is divided into fixed size pages.     
Paging is used for faster access to data.     
When a program needs a page, it is available in the main memory as the OS copies a certain number of pages from your storage device to main memory. Paging allows the physical address space of a process to be noncontiguous.        

What is paging? Why paging is used?       
OS performs an operation for storing and retrieving data from secondary storage devices for use in main memory.     
Paging is one of such memory management scheme. Data is retrieved from storage media by OS, in the same sized blocks called as pages.     
Paging allows the physical address space of the process to be non contiguous. The whole program had to fit into storage contiguously.     

Paging is to deal with external fragmentation problem.     
This is to allow the logical address space of a process to be noncontiguous, which makes the process to be allocated physical memory.     

------------------------

메모리 풀 기법은 힙 영역에 한번에 대량의 사이즈의 메모리를 할당받아서 최대한 Cache hit의 가능성을 높이는 기법이다.        

용어 정리 :         
Page Directory, Page Table, Page Table Entry, Page(Page Frame)은 각각 다른 것을 나타내는 용어다.     

Page Table :

메모리에 관해 읽어보면 좋을 것들 :      
[https://en.wikipedia.org/wiki/Virtual_address_space](https://en.wikipedia.org/wiki/Virtual_address_space)        
[https://en.wikipedia.org/wiki/Page_table#Page_table_types](https://en.wikipedia.org/wiki/Page_table#Page_table_types)        
[https://en.wikipedia.org/wiki/Memory_paging](https://en.wikipedia.org/wiki/Memory_paging)            
[https://en.wikipedia.org/wiki/C_dynamic_memory_allocation#Implementations](https://en.wikipedia.org/wiki/C_dynamic_memory_allocation#Implementations)          
[https://en.wikipedia.org/wiki/Memory_management#Fixed-size_blocks_allocation](https://en.wikipedia.org/wiki/Memory_management#Fixed-size_blocks_allocation)         
