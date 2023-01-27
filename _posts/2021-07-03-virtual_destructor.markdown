---
layout: post
title:  "부모 클래스 소멸자에서 virtual 안붙이면 어디까지 호출되냐?"
date:   2021-07-03
tags: [C++]
---

```cpp
class A
{
public:
	~A()
	{

	}
};

class B : public A
{
public:
	~B()
	{

	}
};

int main()
{
	A* a = new B();
	B* b = new B();

	delete a;
	delete b;
}
```

결론만 말하면 a를 삭제했을 때는 클래스 A의 소멸자만 호출된다.      
당연하다 virtual이 안붙었으니 virtual table에는 소멸자에 대한 정보는 없을 것이고 그냥 포인터 타입 그래도 클래스 A의 소멸자만 호출된다.        

b를 삭제하면 클래스 B의 소멸자 -> 클래스 A의 소멸자 순으로 호출된다.       
