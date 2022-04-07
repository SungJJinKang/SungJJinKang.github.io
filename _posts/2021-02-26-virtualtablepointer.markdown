---
layout: post
title:  "문뜻 발견한 virtual table pointer의 존재"
date:   2021-02-26
categories: C++ ComputerScience
---

예전에 virtual 함수를 찾을 때는 각 클래스가 가지고 있는 virtual table을 look up하여 오브젝트 타입에 맞는 virtual function을 찾는 다는 것을 배웠었다. 또한 이 virtual table를 look up을 위해서는 virtual table pointer라는 것을 모든 virtual class의 오브젝트들이 가지고 있다.     
나는 이 사실을 까먹은 채 내가 정의하지 않은 써드파티 라이브러리 struct assimpAABB3D로 부터 내가 정의한 클래스 AABB3D로 데이터를 복사하기 위해 memmove 함수를 통해 데이터를 옮겼다.       

그런데 디버그를 해보니 제대로 값이 들어와 있지 않은 것이다... 메모리 까지 까서 봤는 데 왠걸 나도 알지 못하는 4바이트가 AABB3D 오브젝트의 첫 4바이트를 차지하고 있던 것이었다. 아무리 클래스 정의를 뒤져봐도 이 4바이트의 정체를 알 수 없었다. 그래서 디버거에서 각 오브젝트의 멤버변수를 보니 **vfptr이라는 변수가 있는 것이다.**      
처음 봤을 때는 이 변수가 뭔지를 몰랐다. 그렇지만 금방 깨달았다. virtual table 포인터라는 것을... 문득 virtual table pointer의 존재를 다시 한번 깨달았다.      

그래서 이 문제를 해결하기 위해 mAABB3D.mLowerBound의 주소를 destination으로 memmove를 하니 잘되었다.      

```cpp
struct assimpAABB3D
{
    aVector3 mLowerBound; // 3 float number
    aVector3 mUpperBound; // 3 float number
}

class AABB3D : public RenderPhysics
{
    public:
        math::Vector3 mLowerBound; // 3 float number
        math::Vector3 mUpperBound; // 3 float number

        ~~
    }
}

assimpAABB3D mAssimpAABB3D;
AABB3D mAABB3D;

std::memmove(&mAssimpAABB3D, &mAABB3D, typeof(math::Vector3) * 2); // Weird!!
```     

오늘의 교훈 :       
1. 디버깅은 정말 중요하다!!!!            
2. 마소는 프로그래머의 신이다.        