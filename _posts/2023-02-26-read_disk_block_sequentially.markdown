---
layout: post
title:  "FileOpenLog, .ubulk 파일의 성능적 관점에서의 의미"
date:   2023-02-26
tags: [ComputerScience, UE, Recommend]
---          

언리얼 엔진의 [FileOpenLog](https://twitter.com/jack_knobel/status/1068294442962972672) 최적화, .ubulk 파일 구조는 왜 존재할까요?                     
이는 파일 IO 성능 때문입니다.              
                  
인게임에서 패키지를 읽는 순서를 저장해두었다가 pak 파일 생성시 그 순서대로 패키지들을 차례대로 저장하는 **FileOpenLog**는 디스크 Seek Time을 줄이는데 도움이 됩니다. HDD의 헤드가 한쪽 방향으로 회전하며 데이터를 읽다가 반대 방향의 데이터를 읽어야할 때 오는 성능적 손해를 줄일 수 있다.         
"HDD의 헤드가 한쪽 방향으로 회전하며 데이터를 읽다가 반대 방향의 데이터를 읽어야할 때 오는 성능적 손해"를 줄이기 위한 다른 방법으로는 에셋을 중복해서 패키징에 포함시키는([Asset duplication](https://www.reddit.com/r/PS5/comments/nkt67n/lets_talk_about_asset_duplication/?utm_source=share&utm_medium=web2x&context=3)) 방법도 있습니다. 한 에셋에서 어떤 다른 에셋의 데이터를 포함시키고자 할 때 Indirection으로 해당 에셋을 가리키는 것이 아니라 그냥 해당 에셋(데이터)을 한번 더 통째로 In-place에 저장함으로서 디스크 Seek 타임을 줄이는 것입니다. 게임 패키징 사이즈가 커지는 대신 디스크 Seek Time을 줄여 런타임 IO 성능을 높이는 방법입니다.       
디스크 Seek Time으로 인한 성능 저하는 HDD에서만 문제가 되지 SSD에서는 문제가 안되기는 하지만, OS에서 "Read-ahead Caching(프로그램이 데이터를 순차적으로 읽을 것을 가정하고 뒤의 데이터들을 미리 읽어와 캐싱해둠)", "Prefetching(파일 읽기 패턴을 인지하여 미리 읽어와 캐싱해둠)"등의 동작이 존재하기 때문에 이러한 최적화 동작을 활용하기 위해서는 연속해서 읽을 데이터들을 보조 기억 장치 내에 연속해서 저장해주는 것이 좋습니다.    
                                 
그리고 기본적으로 보조 기억 장치에 저장된 데이터를 읽을 때는 블록 단위로 읽기 때문에 FileOpenLog를 통해 연속해서 읽을 데이터들을 인접하게 저장해 최대한 같은 블록 내에 속하게 해주는 것이 성능에 도움이 됩니다.                    
이는 HDD뿐만 아니라 SSD에서도 여전히 유효합니다.           
![언리얼 엔진4 텍스처 스트리밍 분석 (2)](https://user-images.githubusercontent.com/33873804/235318130-fdd41576-9955-4914-a57e-e33acde29c40.png)                    
                                     
언리얼 엔진은 연속해서 읽을 데이터들을 인접하게 저장하기 위해 몇몇 타입의 에셋들을 패키징할 때 .uasset 파일과 별도로 **.ubulk 파일**을 패키징에 포함시키는데요.       
대표적인 예는 텍스처입니다.        
텍스처를 패키징할 때 왜 별도의 .ubulk 파일을 함께 저장하는지는 사내 스터디에서 발표한 발표 자료로 대체하겠습니다.           
![언리얼 엔진4 텍스처 스트리밍 분석 (3)](https://user-images.githubusercontent.com/33873804/235318507-9ced7cf6-5114-4644-b188-48fd94672d24.png)           
![언리얼 엔진4 텍스처 스트리밍 분석 (4)](https://user-images.githubusercontent.com/33873804/235318509-97d28245-834b-4b1d-8fd8-3ca7c35de85f.png)         
![언리얼 엔진4 텍스처 스트리밍 분석 (5)](https://user-images.githubusercontent.com/33873804/235318673-182e3f6b-c548-4e99-805b-9e2315f46633.png)           
![언리얼 엔진4 텍스처 스트리밍 분석 (6)](https://user-images.githubusercontent.com/33873804/235318674-49f9441e-7f34-4726-be10-16220b007b28.png)          
![언리얼 엔진4 텍스처 스트리밍 분석 (7)](https://user-images.githubusercontent.com/33873804/235318675-91aec90e-36e2-45ab-af2b-c94a6258d7db.png)              
![20230430025824](https://user-images.githubusercontent.com/33873804/235317302-5687574a-912f-4a76-9bf5-f992f280e9a9.png)            

실제로 패키징된 텍스처 파일을 보시면 .uasset, .uexp, .ubulk 파일 이렇게 세 개의 파일로 저장되는 것을 보실 수 있을겁니다. ( 대부분의 플랫폼이 이러합니다. )             

----------------------------     

추가적으로 ChatGPT의 도움을 받아...               
               
----------------------------               
               
Please Explain me **what is the "disk block"** and tell me it **in relation with windows kernel file system**                                   
               
---------------------------               
               
In a file system, **a disk block (also known as a sector or cluster) is a fixed-sized unit of storage on a physical disk.** The size of a disk block can vary depending on the file system and the specific configuration of the disk. Common sizes include 512 bytes, 1 kilobyte, 2 kilobytes, 4 kilobytes, and 8 kilobytes.               
               
The purpose of disk blocks is **to allow the file system to efficiently manage the physical storage space on the disk.** When a file is stored on disk, it is divided into one or more disk blocks. **The file system keeps track of which disk blocks are allocated to each file, and it can easily read and write the data in those blocks as needed.**               
               
In the Windows kernel, the file system is responsible for managing the allocation and use of disk blocks. When a file is created or extended, the file system determines which disk blocks to allocate to the file. When data is written to the file, the file system writes the data to one or more disk blocks and updates its internal bookkeeping to reflect the changes.               
               
**When a file is read sequentially from disk, the file system can take advantage of the fact that disk blocks are stored in a contiguous manner on the physical disk.** This allows **the file system to read multiple disk blocks at once, reducing the number of seeks required to access the data.** The file system can also use techniques like **caching and prefetching** to further optimize performance when reading sequentially.               
               
In summary, disk blocks are fixed-sized units of storage on a physical disk that are used by the file system to efficiently manage the storage space. The Windows kernel file system is responsible for managing the allocation and use of disk blocks and can use sequential access and caching techniques to optimize performance when accessing them.                 

---------------------------                      
                                       
In the Windows operating system, the file I/O subsystem is managed by the Windows kernel. When an application requests to read data from a file, **the kernel retrieves the data from the file system and caches it in memory. Subsequent read requests for the same data can then be satisfied from the cache, which can be much faster than reading the data from disk.**                                             
                                                                   
For sequential file reads, Windows can optimize performance by using techniques like read-ahead caching and prefetching. **Read-ahead caching involves reading and caching blocks of data ahead of the current read position, based on the assumption that the application will continue reading sequentially.** **Prefetching involves identifying patterns in file access and preloading data into the cache before it is actually requested.** These techniques can help reduce disk seeks and improve overall performance for sequential file reads.                                            
                                                                  
In addition to caching and prefetching, **the Windows kernel also provides support for overlapped I/O, which allows multiple file I/O requests to be submitted at once and executed concurrently. This can further improve performance by reducing the amount of time the application has to wait for I/O operations to complete.**                                       
                      
Overall, the Windows kernel provides a number of optimizations and techniques for managing file I/O and improving performance, including those specifically designed for sequential file reads.                      
                      
