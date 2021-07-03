---
layout: post
title:  "nullptr이 전달되는건지를 컴파일 타임에 체크하기"
date:   2021-07-03
categories: C++
---

함수의 매개변수로 포인터를 전달받는 경우 대개 null 포인터를 걸러내기 위해 assert 등을 사용한다. 그런데 assert도 결국 디버그 모드에서 프로그램을 빌드해보고 테스트 중에 발견하는 것이기 때문에 한계가 있다. 이를 해결하기 위해서 아예 nullptr을 타입으로 가지는 함수를 아예 삭제시켜 컴파일러에서 nullptr이 전달되는지를 확인할 수 있다.         

C++에서는 std::nullptr_t라고 null 포인터의 타입이 있다. 그래서 nullptr을 전달하는 경우 매개변수 타입 중에 std::nullptr_t가 있으면 그 함수가 호출된다. 이 성질을 이용하면 우리는 컴파일 단계에서 매개변수로 nullptr이 전달되는 것을 방지할 수 있다. ( Intellisense 단계에서 알 수 있다. )

```c++
class A
{
public:
	A() = default;

	A(int* data)
	{

	}
	A(std::nullptr_t) = delete; // !!
	
};

int main()
{
	int* p = nullptr;
	A a(p); // 문제 없음!!
	A b(nullptr); // 컴파일러 에러!!!!
}
```

물론 위에서 보다 싶이 한계는 분명이 존재한다. nullptr을 직접적으로 전달한 경우 컴파일 단계에서 문제를 발견할 수 있지만 포인터에 nullptr을 담아서 넘기는 경우에는 컴파일러가 오류를 만들지 않는다. 어찌 보면 당연한 얘기이기는 하다. 왜냐면 타입을 보고 컴파일러가 호출할 함수를 정해 오류를 만드는 것이기 때문이다.      