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

X86 OS환경 내에서는 가상 메모리 주소 32비트로 구성되어 있는 데 우선 최상위 10비트는 Page Directory의 index이다. 이 Page Directory 내의 index를 따라가면 Page table의 위치를 제공한다.     
그럼 다음 10비트는 이 Page Table 내의 index이다. 이 index를 따라가면 Page Table Entry를 발견하는 데 이 Page Table Entry는 메모리 내에 4Kb 사이즈 페이지 프레임의 base 주소를 가진다.    
그리고 마지막 12비트는 이 페이지 프레임 내에서의 base 주소로 부터의 offset을 가리킨다. 이 offset을 따라가면 원하는 virtual address에 대한 physical address 주소를 가질 수 있다.      
 
![3-s2 0-B9780128112779000031-f03-07-9780128112779](https://user-images.githubusercontent.com/33873804/110155324-cee77900-7e28-11eb-9a51-c2300f733589.jpg)                  


자 그럼 가상 메모리 주소를 어떻게 Physical한 주소로 변환할까?       

1. CPU에 있는 MMU(memory management unit)이라는 장치가 이 작업을 하는 데 우선 MMU는 MMU 내에 TLB(Translation lookaside buffer)라는 메모리 캐시를 먼저 탐색한다.    
TLB내에는 최근에 변환했던 virtual address와 그 physical address가 저장되어 있다.      
그럼 MMU는 우선 이 TLB를 먼저 탐색한다.      
만약 TLB에 원하는 virtual address가 있으면 바로 거기 저장된 physical address를 반환하면 된다.       

2. 그런데 만약 TLB에 없다면 MMU는 이제 실제 메모리로 가서 메모리 내의 페이지 테이블을 확인한다. virtual address의 페이지 넘버를 가지고 메모리 내 페이지 테이블 블록(엔트리)을 확인한다. 이 페이지 테이블 블록(엔트리)가 메모리의 데이터(!)를 가지고 있는 것은 아니다(!!). 각 블록은 virtual address에 상응하는 physical memory 주소(!!!)를 가지고 있다.            
이 페이지 테이블 블록은 대개 4KB 정도의 사이지를 가진다. 그 이유는 우선 
이 페이지 테이블을 검색하여 원하는 위치의 테이블 블록(페이지)를 확인하고 이 페이지 블록이 가리키는 physical address를 반환한다.     

3. 여기서 중요한 내용이 나오는 데 원하는 데이터가 메모리 내에 없을 수도 있다. 이게 무슨 말인가? 메모리에 없다면 어디 있나?? 바로 하드디스크와 같은 디스크에 메모리 내용을 임시로 저장해 둘 수 있다. virtual memory라는 개념을 들어보았을 건데 메모리의 용량이 부족하다면 컴퓨터는 메모리에서 오래 동안 사용되지 않은 데이트 블록(페이지)를 디스크 내에 paging file에 임시로 저장해 둔다. 이를 page out이라고도 한다. 이렇게 메모리 블록이 page out되어 있는 경우에는 disk의 paging file에서 다시 메모리로 page in을 하여 우선 데이터를 메모리로 복사한 후 다시 page table로 부터 physical address를 가져온다.     

4. 여기서 중요한건 메모리에서 바로가져오든 page in해서 주소를 가져오든 반드시 TLB에 그 변환을 기록해야한다. 이 기록을 한 후 MMU는 처음부터 TLB를 뒤진다. 그럼 당연히 방금 전에 TLB에 기록했으니 그 기록을 보고 MMU는 physical memory에 접근한다.      



페이지 테이블내의 각각의 페이지 블록(페이지 테이블 엔트리)는 virtual address와 physical address의 매핑 정보 말고도 추가적인 정보를 가지고 있는 데 그 종류는 아래와 같다.    

Reference bit(페이지에 접근이 있었는지, page out할 블록을 정할 떄 사용함),      
Valid bit(해당 page의 실제 데이터가 메모리에 있는지, page out되어 하드디스크 내 paging file에 있는지),      
Dirty bit(해당 페이지의 메모리 데이터가 page in된 이후 데이터가 변하였는지, 변한 경우 다시 page out될때 반드시 해당 내용을 disk의 paging file에 copy해줘야한다, 달라진게 없으면 굳이 paging file에 메모리 데이터 내용을 안 써줘도 된다),       
Process ID information(예전에는 모든 프로세스가 하나의 페이지 테이블을 가져 각기 다른 프로그램(프로세스)가 같은 physical address에 접근하려 할 때 이를 구분해주기 위해 프로세스의 ID를 기록하였다. 근데 최근에는 그냥 프로세스마다 자신만의 페이지 테이블을 메모리에 가지고 있다. 이 bit는 잘 안쓰인다.)          

이 페이지 테이블도 여러 종류가 있는데 그건 아래 주소에서 확인하자. 참고로 x86은 Multi-level이라는 방식을 사용한다.       
[페이지 테이블 종류](https://en.wikipedia.org/wiki/Page_table#Page_table_types)


Reference :      
[https://en.wikipedia.org/wiki/Page_table#Page_table_types](https://en.wikipedia.org/wiki/Page_table#Page_table_types)