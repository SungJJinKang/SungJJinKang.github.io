---
layout: post
title:  "std::function은 왜 느릴까?"
date:   2021-05-20
categories: C++
---

우선 std::function을 왜 사용하는지를 알아야한다.    
이유는 바로 polymorphism을 지원하기 위함이다. 템플릿을 사용하지 않고 어떤 함수 타입을 가진 다양한 오브젝트를 저장 혹은 전달하기 위해 사용된다.    
polymorphism을 지원한다는 점에서 **virtual 함수를 호출할 때와 std::function을 사용할 때 내부적으로 거의 같은 원리로 작동한다.**     
둘다 함수 inlining이 되지 않고 함수 포인터를 통해 함수를 호출한다는 점이 virtual함수와 std::function의 특징이다.        

std::function은 callable object의 함수 타입만 알지 callable object의 사이즈는 알지 못한다. 그래서 불가피하게 new를 통해 dynamic allocation을 해서 포인터로 해당 callable object의 copy를 가지고 있어야한다.     
이렇게 dynamic alloaction 과정에서 성능저하가 생긴다.

그리고 성능 하락의 가장 큰 원인은 **std::function에 저장된 function은 고정된 것이 아니기 때문에 컴파일러가 함수를 inling할 수 없다.**      
이렇게 inlining 되지 못한 함수는 virtual함수를 호출하는 것과 inling이 가능함 lambda보다 상대적으로 느릴 수 밖에 없다.    
callable object의 함수 길이가 짧은 경우에는(3~5 instruction) lamba보다는 분명히 느릴 것이다. 그렇지만 함수 길이가 긴 경우에는(Instruction이 매우 긴 경우) 상대적으로 둘 사이 성능 차이는 크게 두드러지지 않는다.      

왠만하면 **std::function보다는 template을 통해 lambda를 사용하는 것을 지향**해야한다.      
아래 코드를 보자.      
```c++
#include <functional>

template <typename F>
void __attribute__((noinline)) use_lambda(F const & f) { // !!!!!!!!!!!!!!
    auto volatile a = f(13); // call f
    // ....
    auto volatile b = f(7); // call f again
}

void __attribute__((noinline)) use_func(
        std::function<int(int)> const & f) {
    auto volatile a = f(11); // call f
    // ....
    auto volatile b = f(17); // call f again
}

int main() {
    int x = 123;
    auto f = [&](int y){ return x + y; };
    use_lambda(f); // Pass lambda
    use_func(f); // Pass function
}
```
    
**std::function**은 lambda를 직접 호출하지 못하고 **lambda의 포인터를 std::function 내부적으로 저장했다가 호출하는 것이기 때문에 컴파일러는 std::function 내부 함수포인터에서 어떤 함수를 호출할지를 알 수 없기 때문에 inlining을 하지 못한다.**            
반면 **use_lambda**를 호출하는 경우에는 **lambda의 타입을 템플릿 매개변수로 받아 직접 lambda를 호출하고 있다. 이 경우 컴파일러는 inlining을 한다.**         
**템플릿 매개변수를 사용하여 컴파일 타임에 어떤 함수를 호출할지가 확정되기 때문에 컴파일러가 inlining을 할 수 있는 것이다.**          
                   
----------------------                
                       
나중에 알게된 것은 이 이유 말고도 힙할당의 문제가 있다. 흔히 람다를 사용할 때 Capture를 하는데 이 Capture한 오브젝트를 std::function에 저장하기 위해서는 결국 힙할당이 필요한데 여기 드는 비용이 큰 것이다. 물론 std::string 처럼 Small Size 버퍼가 내부적으로 있어서 일정 사이즈보다 작은 경우 힙할당을 하지는 않지만 그 버퍼 사이즈가 크지 않다. 그래서 [협업에서는 이 Small Size 버퍼를 늘려서 자제 std::function을 사용한다.](https://youtu.be/tD4xRNB0M_Q?t=1725)         

references : [https://stackoverflow.com/questions/18453145/how-is-stdfunction-implemented](https://stackoverflow.com/questions/18453145/how-is-stdfunction-implemented), [https://stackoverflow.com/questions/5057382/what-is-the-performance-overhead-of-stdfunction](https://stackoverflow.com/questions/5057382/what-is-the-performance-overhead-of-stdfunction), [https://stackoverflow.com/questions/18608888/c11-stdfunction-slower-than-virtual-calls](https://stackoverflow.com/questions/18608888/c11-stdfunction-slower-than-virtual-calls), [https://stackoverflow.com/questions/67615330/why-stdfunction-is-too-slow-is-cpu-cant-utilize-instruction-reordering](https://stackoverflow.com/questions/67615330/why-stdfunction-is-too-slow-is-cpu-cant-utilize-instruction-reordering), 
