---
layout: post
title:  "가상 메모리 주소, 메모리 페이징에 대한 나의 이해"
date:   2021-03-05
categories: ComputerScience
---

우선 가상 메모리 주소를 왜 사용할까?? Physical한 메모리 주소를 사용하려면 프로그램 단에서 일일이 관리해야하는데 이걸 OS단에서 대신 해주어서 프로그램은 자신의 virtual address space만 신경 쓰면된다.   
또한 virtual address의 사이즈가 실제 physical address보다 클 수 있는 데 이건 실제 physicla 메모리 사이즈보다 메모리를 더 많이 활용하게 해준다. 이건 후에 서술할 virtual memory(disk) 덕분이다.         
프로그램(프로세스)마다 virtual address space가 각각 자신의 것을 가지고 있는 데 이 virtual address가 같다고 physical address도 같은건 아니다.    
또한 virtual address는 연속되어 있는 것 처럼 보이지만 이 virtual address가 가리키는 physical address는 연속되어 있지 않고 파편화 되어있는 경우가 많다.          

1. X86 OS환경 내에서는 가상 메모리 주소 32비트로 구성되어 있다. 상위 20비트는 페이지의 physical address를 찾는 데 사용하고 하위 12비트는 페이지 내의 우리가 찾는 정확한 physical address를 찾는 데 사용된다.    

2. 우선 최상위 10비트는 Page Directory의 index이다.(Page Directory는 운영체제에 따라 없어서 상위 20비트가 바로 Page Table의 index를 가리키는 경우도 있다.).     
Page Direcotry는 페이지 테이블의 base address들을 가지고 있다. 그럼 최상위 10비트를 가지고 이 Page Direcotry 내의 index를 찾아간다. 그럼 그 위치에는 페이지 테이블의 base address를 가지고 있다.     

3. 그럼 이 주소를 따라가면 페이지 테이블이 나오는 데 이 페이지 테이블은 페이지 테이블 엔트리들로 구성되어 있다. 여기서 다음 10비트가 index로 사용되어 페이지 테이블 엔트리를 찾게된다.         
이 index를 따라가면 Page Table Entry를 발견하는 데 이 Page Table Entry는 메모리 내에 페이지의 base 주소를 가진다. 이 페이지가 우리가 비로소 찾아왔던 실제 데이터들의 묶음(4KB)이다(왜 묶음이냐? 페이징 기법이 뭔지를 찾아봐라).    

4. 그리고 마지막 12비트는 이 페이지내에서의 base 주소로 부터의 offset을 가리킨다. 이 offset을 따라가면 비로소 원하는 virtual address에 대한 physical address 주소를 가질 수 있다.      
TLB라는 개념이 여기서 나오는 데 TLB는 virtual address의 상위 20비트를 가지고 찾은 페이지(페이지 프레임)의 physical address를 저장하고 이 값을 가지고 나중에 offset과 합쳐서 실제 페이지 내의 원하는 주소의 physical address를 가진다.        

위에 나온 Page Direcotry, Page Table, Page는 모두 Physical memory 내에 위치해 있다. TLB는 MMU에 있다.    

 
![24467A4E579115AB09 (1)](https://user-images.githubusercontent.com/33873804/110157365-6c43ac80-7e2b-11eb-884b-8d9e3efeb7ca.png)    
[https://introfor.tistory.com/70](https://introfor.tistory.com/70)     

------------------------------------

자 그럼 가상 메모리 주소를 어떻게 Physical한 주소로 변환할까?       

1. CPU에 있는 MMU(memory management unit)이라는 장치가 이 작업을 하는 데 우선 MMU는 MMU 내에 TLB(Translation lookaside buffer)라는 메모리 캐시를 먼저 탐색한다.    
TLB내에는 최근에 변환했던 virtual address와 그 physical address가 저장되어 있다.      
그럼 MMU는 우선 이 TLB를 먼저 탐색한다.      
만약 TLB에 원하는 virtual address가 있으면 바로 거기 저장된 physical address를 반환하면 된다. (후술하지만 정확히는 TLB가 페이지 테이블 엔트리를 저장하고 하위 offset비트로 이 엔트리에서 실제 physical memory address로 접근한다.)        

2. 그런데 만약 TLB에 없다면 MMU는 virtual address의 상위 10비트를 가지고 메모리 내의 페이지 테이블을 확인한다.     
그리고 중간 10비트로 페이지 테이블 내의 페이지 테이블 엔트리를 찾는다. 이 페이지 테이블 엔트리는 4KB 사이즈의 페이지의 base 주소를 가지고 있다.    
이 페이지가 비로소 우리가 찾아왔던 실제 메모리 데이터들이다. 이 4KB의 페이지 내에서 마지막 12비트를 offset으로 사용하여 페이지 내의 목표로 했던 physical address를 TLB에 반환한다.        

3. 여기서 중요한 내용이 나오는 데 원하는 페이지가 메모리 내에 없을 수도 있다.     
이게 무슨 말인가? 메모리에 없다면 어디 있나?? 바로 하드디스크와 같은 디스크에 메모리 내용을 임시로 저장해 둘 수 있다.     
virtual memory라는 개념을 들어보았을 건데 메모리의 용량이 부족하다면 컴퓨터는 메모리에서 오래 동안 사용되지 않은 페이지(4KB)를 디스크 내에 paging file에 임시로 저장해 둔다. 이를 page out이라고도 한다.      
이렇게 찾고자 하는 페이지가 page out되어 있는 경우를 page fault라고한다.       
이 경우에는 disk의 paging file에서 다시 메모리로 page in을 하여 데이터를 disk에 저장되어 있는 페이지를 메모리로 복사해서 가져와야한다. (페이지 단위로 저장하고 로드한다. 대개 4KB이다).     
그럼 이제 디스크에서 불러온 이 페이지의 physical address를 페이지 테이블에 업데이트하고 난 후 이 페이지 주소를 반환한다. ( 여기서 TLB 존재이유가 나온다. TLB가 있으면 이렇게 복잡한 페이지를 찾는 과정을 안거쳐도 되는 것이다. )        

4. 여기서 중요한건 메모리에서 바로가져오든 page in해서 주소를 가져오든 반드시 TLB에 그 변환을 기록해야한다. 이 기록을 한 후 MMU는 처음부터 TLB를 뒤진다. 그럼 당연히 방금 전에 TLB에 기록했으니 그 기록을 보고 MMU는 physical memory에 접근한다.      


-----------------------------


페이지 테이블내의 각각의 페이지 테이블 엔트리는 페이지의 physical address말고도 추가적인 정보를 가지고 있는 데 그 종류는 아래와 같다.    

Reference bit(페이지에 접근이 있었는지, page out할 블록을 정할 떄 사용함),      
Valid bit(해당 page의 실제 데이터가 메모리에 있는지, page out되어 하드디스크 내 paging file에 있는지),      
Dirty bit(해당 페이지의 메모리 데이터가 page in된 이후 데이터가 변하였는지, 변한 경우 다시 page out될때 data들을 disk의 paging file에 copy해줘야한다, 달라진게 없으면 굳이 paging file에 메모리 데이터 내용을 안 써줘도 된다),       
Process ID information(예전에는 모든 프로세스가 하나의 페이지 테이블을 가져 각기 다른 프로그램(프로세스)가 같은 physical address에 접근하려 할 때 이를 구분해주기 위해 프로세스의 ID를 기록하였다. 근데 최근에는 그냥 프로세스마다 자신만의 페이지 테이블을 메모리에 가지고 있다. 이 bit는 잘 안쓰인다.)          

이 페이지 테이블도 여러 종류가 있는데 그건 아래 주소에서 확인하자. 참고로 x86은 Multi-level이라는 방식을 사용한다.       
[페이지 테이블 종류](https://en.wikipedia.org/wiki/Page_table#Page_table_types)


---------------------------              

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

메모리에 관해 읽어보면 좋을 것들 :      
[https://en.wikipedia.org/wiki/Virtual_address_space](https://en.wikipedia.org/wiki/Virtual_address_space)        
[https://en.wikipedia.org/wiki/Page_table#Page_table_types](https://en.wikipedia.org/wiki/Page_table#Page_table_types)        
[https://en.wikipedia.org/wiki/Memory_paging](https://en.wikipedia.org/wiki/Memory_paging)            
[https://en.wikipedia.org/wiki/C_dynamic_memory_allocation#Implementations](https://en.wikipedia.org/wiki/C_dynamic_memory_allocation#Implementations)          
[https://en.wikipedia.org/wiki/Memory_management#Fixed-size_blocks_allocation](https://en.wikipedia.org/wiki/Memory_management#Fixed-size_blocks_allocation)         