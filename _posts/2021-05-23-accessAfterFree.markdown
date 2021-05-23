---
layout: post
title:  "free된 포인터에 접근하기??"
date:   2021-05-23
categories: C++
---

```c++
int main()
{
	int* a = new int;
	*a = 5;

	int* b = a;

	delete a;

	std::cout << *b << std::endl;
}
```

위의 코드는 결과는??        
정확히는 " 알 수 없다 ( Undefined Behaviour ) "이지만 대부분의 경우 5가 출력될 것이다.                 

b는 a와 같은 주소를 가지고 있고 delete a를 해서 a 주소에 위치한 메모리를 해제해주었는데 어떻게 출력이 정상적으로 될까???          

이유는 OS 입장에서는 해당 메모리 블록은 프로그램(프로세스)에 이미 할당을 한 것이기 때문에 메모리 블록 내에서 프로그램이 할당을 했는지 해제를 했는지는 알 수 없다.              

구체적으로 설명하자면 프로그램에서 처음 malloc을 호출하면 프로그램(프로세스)는 OS로부터 일정한 크기의 메모리 블록을 할당받는다. ( 보통 [페이지](https://sungjjinkang.github.io/computerscience/2021/03/05/virtualmemoryaddress.html) 단위이다 ) 그리고 **할당받은 메모리를 다시 해제하면 프로그램은 대걔 그 메모리를 OS에 다시 할당하지 않고 가지고 있는다 ( free 구현에 따라 다르다 ). 대신 메모리를 가지고 있다가 비슷한 사이즈의 할당 요청이 들어오면 들고있던 메모리 주소를 return 해준다. 이는 많은 프로그램들이 할당과 해제를 반복하는데 이 때마다 OS에 메모리 할당을 요청하는 것은 매우 느리기 때문에 프로그램 내부적으로 할당되지 않은 메모리들을 관리하여 OS로의 요청 없이 빠르게 메모리를 할당해주기 위함이다.**        

위의 경우에는 a를 delete한 후 곧 바로 a에 저장되어 있던 주소에 접근하였기 때문에 아직 프로그램이 해당 메모리 블록을 OS에 반환하지 않은 상태이므로 접근이 가능했던 것이다.       
만약 OS가 메모리 블록을 반환하였다면 프로그램은 해당 위치의 메모리에 대한 권한이 없기 때문에 OS는 예외를 던져줄 것이다.       

언제 메모리 블록을 OS에 반환하는지는 구현에 따라, 플랫폼에 따라 다르기 때문에 이렇게 해제한 메모리에 접근하는 것은 "Undefined Behaviour"이다. ( 이 글의 코드들은 MSVC X64 환경에서 테스트된 것이다. )          



```c++
int main()
{
    int* a = new int;
    delete a;
    int* b = new int;
}
```

아래 코드에서 모든 연산이 끝난 후 포인터 a와 포인터 b는 같은 adress 값을 가진다.     
위와 마찬가지로 a가 할당 후 해제되었지만 프로그램은 여전히 메모리 블록을 가지고 있고 다시 메모리 할당 요청이 들어왔을 때 free 상태인 메모리를 반환해주는 것이다. ( 물론 이것은 undefined behaviour이라 플랫폼에 따라 컴파일러에 따라 다른 결과가 나올 수 있다. )             

references : [https://stackoverflow.com/questions/67658574/does-os-know-program-did-malloc-or-free-in-memory-block-gived-from-os](https://stackoverflow.com/questions/67658574/does-os-know-program-did-malloc-or-free-in-memory-block-gived-from-os),  [https://stackoverflow.com/questions/42588482/c-accessing-data-after-memory-has-been-freeed](https://stackoverflow.com/questions/42588482/c-accessing-data-after-memory-has-been-freeed)