---
layout: post
title:  "Moved 된 오브젝트의 primitive타입의 멤버 변수가 초기화 되지 않아 생기는 문제"
date:   2021-02-20
categories: C++
---

프로젝트를 계속 진행하다 보니 뭔가 이상한 점을 발견하였다. GPU 버퍼가 멋대로 파괴되었다가 재생성하고 이러한 과정을 반복하는 것이다. GPU 버퍼가 이렇게 불필요하게 파괴되었다 재생성 되는 것은 엄청난 오버헤드이기 때문에 빨리 원인을 찾아야 했다. 그리고 원인을 찾았다.    

원인은 버퍼를 관리하는 클래스 오브젝트를 std::vector에 넣어두었는 데 이 std::vector 사이즈가 증가하며 내부적으로 reallocation하는 과정에서 realoocation 전 array에 있던 오브젝트들이 파괴되면서 이러한 문제가 발생한 것이다.     
필자는 기본적으로 생산성과 실수를 줄이기 위해 최대한 내가 Move constructor을 따로 정의하지 않고 컴파일러가 알아서 만들어 주는 Move Constructor, Copy Constructor을 사용하고 노력하였다.   
컴파일러를 만드는 천재 프로그래머들을 믿고 맡긴 것이다. 그런데 여기서 문제가 발생하였다. 컴파일러가 자동으로 생성해주는 Move Constructor을 멤버 변수들을 타입에 맞는 Move semantic을 사용하여 Move를 한다.     
**나는 당연히 다른 클래스들의 Move semantic 처럼 int 타입의 멤버 변수도 move 후 0으로 초기화를 해줄 것이라고 생각했다.**     
그러나 컴파일러는 그렇게 하지 행동하지 않았다. 굳이 불필요하게 int 타입을 0으로 초기화 시켜줄 필요가 없는 것이었다. 내가 컴파일러를 만든다고 해도 굳이 0으로 초기화를 하는 명령어를 넣지는 않았을 것 같다.    
결국에는 0으로 초기화 되지 않은 기존의 오브젝트는 원래의 버퍼 ID를 가지고 있었고 reallocation이 끝난 후 파괴되면서 소멸자가 호출되어 버퍼가 Release되었던 것이다.    
버퍼를 관리하는 오브젝트가 파괴되면 자동으로 버퍼를 Release해주는 그래픽스 API를 호출하는 것 자체는 리소스 관리 측면에서 실수를 줄여줘 매우 좋은 디자인이라 생각한다. ( 필자는 dynamic array의 경우에도 unique_ptr을 적극 활용하여서 실수로 memory leak이 발생할 것을 막기 위해 노력해왔다. )  

그래서 결국 난 GPU 버퍼 ID(int)를 보관하고 Move시 Move된 오브젝트의 버퍼ID를 0으로 초기화 시켜주는 클래스를 만드는 것으로 이 문제를 해결하였다.     


```c++
class A
{
private:
	static inline int BufferCount = 0;
	int mBufferID;
    std::vector<Texture> mTextures{};
	
	void GenerateBuffer()
	{
		this->mBufferID = A::BufferCount++;
		std::cout << "Generate Buffer : " << this->mBufferID << std::endl;
	}
	void ReleaseBuffer()
	{
		std::cout << "Release Buffer : " << this->mBufferID << std::endl;
	}

public:
	A()
	{
		this->GenerateBuffer();
	}
	~A()
	{
		this->ReleaseBuffer();
	}

	A(const A&) = delete;
	A(A&&) noexcept = default;
};

int main()
{
	std::vector<A> vector;
	vector.emplace_back();
	vector.emplace_back();
	vector.emplace_back();
}

---------   
output :

Generate Buffer : 0
Generate Buffer : 1
Release Buffer : 0
Generate Buffer : 2
Release Buffer : 0
Release Buffer : 1

```    
output에서 보이듯이 reallocation 과정에서 Buffer과 Release된 것을 알 수 있다.     

아래는 필자가 해결책으로 만들 간단한 클래스이다.    

```c++
template <typename T>
class ZeroResetMoveContainer
{
	private:
		T data;
	public:
	ZeroResetMoveContainer() : data{ 0 }
	{}
			
	ZeroResetMoveContainer(const ZeroResetMoveContainer&)
	{
		NODEFAULT; 
		// Don't try ZeroResetMoveContainer
		// Think if you copy ZeroResetMoveContainer and Copyed Object is destroyed
		// Other ZeroResetMoveContainer Objects can't know their bufferId is invalidated.
		// This will make bugs hard to find, debug
	}
	ZeroResetMoveContainer(ZeroResetMoveContainer&& bufferID) noexcept
	{
		this->data = bufferID.data;
		bufferID.data = 0;
	}

	ZeroResetMoveContainer& operator=(const ZeroResetMoveContainer&)
	{
		NODEFAULT;
		// Don't try ZeroResetMoveContainer
		// Think if you copy ZeroResetMoveContainer and Copyed Object is destroyed
		// Other ZeroResetMoveContainer Objects can't know their bufferId is invalidated.
		// This will make bugs hard to find, debug
	}
	ZeroResetMoveContainer& operator=(ZeroResetMoveContainer&& bufferID) noexcept
	{
		this->data = bufferID.data;
		bufferID.data = 0;
		return *this;
	}

	ZeroResetMoveContainer(T ID) : data{ ID }
	{}
	void operator=(T iD) noexcept
	{
		this->data = iD;
	}

	operator T()
	{
		return this->data;
	}

	operator T*()
	{
		return &(this->data);
	}

	T& GetReference()
	{
		return this->data;
	}

	const T& GetReference() const
	{
		return this->data;
	}
};

using BufferID = typename ZeroResetMoveContainer<unsigned int>;
}
```


오늘의 교훈 : vector에 오브젝트 포인터가 아닌 클래스 오브젝트를 저장하려면 Move constructor을 잘 정의해야 불필요한 오버헤드를 줄일 수 있다.    
