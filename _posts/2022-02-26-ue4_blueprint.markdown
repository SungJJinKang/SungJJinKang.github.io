---
layout: post
title:  "Unreal Engine 블루프린트가 상대적으로 느린 이유?"
date:   2022-02-26
categories: UnrealEngine4 UE4 ComputerScience ComputerGraphics
---

~~C++ 네이티브 코드의 경우 컴파일러단에서 최적화 ( 인라이닝, 레지스터 재할당, SIMD 레지스터 활용 등등..... )가 많이 들어감. CPU는 그 기계어를 곧바로 실행~~          

~~반면 블루프린트의 경우 바이트 코드로 빌드에 담겨서 런타임에 언리얼의 Virtual Machine에 의해 동작 되기 때문에 컴파일러만큼의 최적화가 불가능. 결국 런타임에 매번 바이트 코드를 명령어 코드로 변환해서 실행해야함 ( 자바 생각하면 된다. ) ( 블루프린트를 빌드 타임에 네이티브 코드로 변환해서 빌드에 넣어주는 기능이 있는데 이건 조금 더 자료 조사가 필요 )~~                   

~~블루프린트의 경우에도 가상 머신단 ( 가상 머신이 별개의 모듈로 존재하는게 아니라 그냥 UObject 시스템에서 동작한다. )에서 C++ 네이티브 코드로 점프를 해서 네이티브 코드로 돌지만 결국 중간에 Helper 코드를 두어서 가상 머신에서 그 Helper 코드를 거쳐서 C++ 네이티브로 코드로 가야하기 때문에 오버헤드는 불가피함. 또한 빌드 타임에 컴파일러를 통한 최적화를 하지 못하기 때문에 이로 인한 성능 손해도 있을 것이다.~~                        

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

블루프린트는 위에서 보앗듯이 성능적으로 손해보는 부분들이 있지만 결국 **개발 효율성 측면에서 블루프린트의 개발 속도를 무시할 수 없다. 코드 한 줄 고치고 느려터진 빌드를 기다려본 경험이 있으면 블루프린트가 매우 효율적인 도구**라는 것을 알 것이다.                
또한 기획자나 아티스트가 간단한 코드를 직접 짜서 불필요한 시간 낭비를 줄임.                 
                    
한 프레임 몇 번 호출안되는 코드는 그냥 조금 느려도 상관 없기 때문에 블루프린트를 사용해도 괜찮다.                           

references : [Blueprints vs. C++: How They Fit Together and Why You Should Use Both](https://youtu.be/VMZftEVDuCE), [https://zhuanlan.zhihu.com/p/92268112](https://zhuanlan.zhihu.com/p/92268112)              