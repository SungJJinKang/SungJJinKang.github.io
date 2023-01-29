---
layout: post
title:  "Unreal Engine 블루프린트가 상대적으로 느린 이유?"
date:   2022-02-26
tags: [UE]
---

아래의 사진은 왼쪽의 블루프린트 바이트코드에서, 오른쪽 위의 블루프린트와 C++ 네이티브 코드( MakeVector 함수 ) 사이의 중간 Helper 함수를 거쳐서, C++ 네이티브 코드 ( MakeVector 함수 )를 호출하는 것을 보여줌.                

<img width="688" alt="20220226163602" src="https://user-images.githubusercontent.com/33873804/155834652-ccf44fe9-10db-45c3-a45d-d9c4d7dab530.png">           

블루프린트에서 나온 바이트 코드를 실행하는 것이 무슨 특별한 것이 있을 것 같지만 별거 없다.             
바이트 코드가 있으면 그 바이트 코드와 상응하는 C++ 네이티브 함수들이 있다. 그럼 해당 바이트 코드에 맞는 C++ 네이티브 코드를 호출을 해주면 된다.           

```cpp
enum EExprToken
{
  ...
  EX_Return = 0x04, // Return from function.
  EX_Jump = 0x06,   // Goto a local address in code.
  EX_JumpIfNot  = 0x07, // Goto if not expression.
  EX_Let  = 0x0F,   // Assign an arbitrary size value to a variable.

  EX_LocalVirtualFunction = 0x45, // Special instructions to quickly call a virtual function that we know is going to run only locally
  EX_LocalFinalFunction = 0x46, // Special instructions to quickly call a final function that we know is going to run only locally
  ...
};

위와 같은 바이트 코드들이 있으면 그 바이트 코드에 맞게 아래와 같은 C++ 네이티브 함수를 호출한다.

DEFINE_FUNCTION(UObject::execJump)
{
    CHECK_RUNAWAY;

    // Jump immediate.
    CodeSkipSizeType Offset = Stack.ReadCodeSkipCount();
    Stack.Code = &Stack.Node->Script[Offset];
}
IMPLEMENT_VM_FUNCTION( EX_Jump, execJump );
```

블루프린트는 위에서 보앗듯이 성능적으로 손해보는 부분(네이티브하게 도는 경우에 비해 여러 Indirection이 추가됨)들이 있지만 결국 **개발 효율성 측면에서 블루프린트의 개발 속도를 무시할 수 없다. 코드 한 줄 고치고 느려터진 빌드를 기다려본 경험이 있으면 블루프린트가 매우 효율적인 도구**라는 것을 알 것이다.                
또한 기획자나 아티스트가 간단한 코드를 직접 짜서 불필요한 시간 낭비를 줄임.                 
                    
한 프레임 몇 번 호출안되는 코드는 그냥 조금 느려도 상관 없기 때문에 블루프린트를 사용해도 괜찮다.                           
           
references : [Blueprints vs. C++: How They Fit Together and Why You Should Use Both](https://youtu.be/VMZftEVDuCE), [https://zhuanlan.zhihu.com/p/92268112](https://zhuanlan.zhihu.com/p/92268112)              