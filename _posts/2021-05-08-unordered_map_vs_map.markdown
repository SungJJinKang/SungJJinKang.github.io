---
layout: post
title:  "std::map vs std::unordered_map ( MSVC STL )"
date:   2021-05-08
categories: C++
---

**std::map은 트리 형태로 데이터를 배치하여 찾으려는 key값과 현재 위치한 node의 key값을 비교하여 그 비교 함수의 결과에 따라 트리의 왼쪽 자식 노드로 갈지 오른쪽 자식 노드로 갈지를 결정하여 이를 반복해 원하는 key값의 node를 찾는 방식이다.** 시간 복잡도로는 O(n)의 시간 복잡도를 가진다.                     
**std::unordered_map은 hash table을 이용하는 방식이다. 우선 key값을 가지고 hash value를 계산한 후 그 hash value를 인덱스로 사용하여 bucket array의 특정 element 접근한다.** 그럼 이 array의 element는 linked list로 같은 hash value를 가지는 key-value pair가 저장된다. hash collision이 발생하는 경우 시간이 더 걸릴 수 있지만 기본적으로는 O(1)의 복잡도를 가진다. ( hash table은 [libcxx의 hash table 구현](https://github.com/llvm-mirror/libcxx/blob/master/include/__hash_table_)을 참고하라 )             

여기까지만 들으면 hash value를 계산하는 방법이 당연히 빠를 것 같아 보인다. 하지만 그렇지 않다.    
일반적으로는 **std::unordered_map이 std::map보다 look-up시 빠르지만 hash value를 계산하는 hash function에서 많은 시간이 소모되는 경우 std::map보다 더 느린 경우가 생긴다.**         

key값으로 pointer나 일반 정수를 사용하는 경우에는 std::unordered_map의 hash value 계산이 빠르다. 그냥 값에다가 hash table의 bucket의 개수만큼을 나머지 연산해주면 되기 때문이다.            

반면 문자열의 경우에는 hash value를 구하는 것이 O(n)의 시간복잡도를 가진다. 문자열의 경우에는 다른 위치에 저장되었지만 같은 문자로 구성된 문자열의 경우 동일한 hash value를 가져야하기 때문이다. MSVC STL에서 문자열의 hash value를 구할 때는 Fnv1a이라는 알고리즘이 사용되는 데 이 알고리즘은 unsigned char*의 포인터들을 1씩 더해가며 포인터 값을 게속해서 xor을 하는 방식으로 hash 값을 구한다. 즉 아래 함수에서 보이는 parameter _Count 문자열의 길이가 hash 함수 성능에 중요한 영향을 준다.      
```
_NODISCARD inline size_t _Fnv1a_append_bytes(size_t _Val, const unsigned char* const _First,
    const size_t _Count) noexcept { // accumulate range [_First, _First + _Count) into partial FNV-1a hash _Val
    for (size_t _Idx = 0; _Idx < _Count; ++_Idx) {
        _Val ^= static_cast<size_t>(_First[_Idx]);
        _Val *= _FNV_prime;
    }

    return _Val;
}
``` 

그래서 문자열을 key값으로 쓸 때는 std::map이 빠를까? std::unordered_map이 빠를까??     
이를 이해하기 위해서 그 전에 std::map의 key값을 문자열로 사용한 경우 내부적으로 어떻게 동작하는지를 우선 알아보자.     

이해를 돕기 위해 std::map이 어떻게 작동하는지 간단히 설명하겠다.           
**std::map은 hash값을 계산할 필요는 없지만 tree구조의 node들을 이동하며 key값을 비교하여 왼쪽 자식 노드를 택할지, 오른쪽 자식 노드를 택할지를 결정한다. 대걔 이렇게 node를 한 단계식 찾아가는 동작으로 인해 O(n)의 시간 복잡도를 가져 std::map이 std::unordered_map보다 느리다고 여겨진다.**                
아래는 흔히 볼 수 있는 문자열 비교 함수이다.       
```
_NODISCARD static _CONSTEXPR17 int compare(_In_reads_(_Count) const _Elem* _First1,
    _In_reads_(_Count) const _Elem* _First2, size_t _Count) noexcept /* strengthened */ {
    // compare [_First1, _First1 + _Count) with [_First2, ...)
    for (; 0 < _Count; --_Count, ++_First1, ++_First2) {
        if (*_First1 != *_First2) {
            return *_First1 < *_First2 ? -1 : +1;
        }
    }

    return 0;
}
```
문자열 비교 함수는 index를 하나씩 올려가며 첫 문자부터 끝 문자까지 한 문자씩 비교를 하지만 다른 문자를 만나는 순간 바로 비교 결과를 return한다는 것이다. ( std::string 비교에서는 std::string이 문자열 길이를 알고 있기 때문에 가장 먼저 문자열 길이를 비교해서 길이가 다르면 비교 연산을 끝낸다. )                  
이 말은 즉 **key값으로 사용된 문자열 중 앞쪽에 위치한 문자들이 서로 다르면 두 문자열을 비교하는 함수를 n번의 루프가 필요없이 빨리 끝낼 수 있다는 것이다.**      
```
아래의 두 문자열을 총 18번의 loop 후 비교 결과를 return 할 수 있다.
"aaaaaaaaaaaaaaaaab"
"aaaaaaaaaaaaaaaaac"

반면 아래의 두 문자열은 한번의 loop만으로 비교 결과를 return 할 수 있다. 
"baaaaaaaaaaaaaaaaa"
"caaaaaaaaaaaaaaaaa"
```

그러면 이제 우리는 문자열을 key값으로 사용하는 경우 std::map과 std::unordered_map 중 어떤 경우가 빠를지 어느 정도 예측을 할 수 있다.      
문자열이 짧은 경우는 당연히 hash value를 금방 구할 수 있어 std::unordered_map이 압도적으로 빠르다.       
문자열이 아주 긴 경우는 두 가지 경우로 나뉜다. **문자열 간의 문자열 길이가 서로 다르거나, 문자열의 앞쪽에 다른 문자가 있는 경우에는 hash value를 계산하는 데 O(n)이 소요되는 std::unordered_map보다 std::map이 더 빠를 수 있다.** 아무리 std::map이 tree구조라 하여도 한번의 비교로 어떤 자식 노드를 탐색해야하는지가 결정이 되면 std::unordered_map이 hash value 계산이 끝나기도 전에 std::map은 원하는 key-value를 찾을 수도 있을 것이다.     

예를 들어보면 100개의 문자로 이루어진 문자열 10개이 있다. 그리고 10개의 문자열 모드 첫번째 문자가 서로 다르다.      
이 경우 map의 경우는 최대 문자 4개만 비교(map은 red black tree로 이루어져 하위 노드의 balance가 유지된다)하면 바로 원하는 key값을 가진 노드를 찾을 수 있다.        
반면 unordered_map의 경우는 hash value를 구하기 위해서 100번의 xor 연산을 해야한다. 그 후에도 문자열간 hash value의 충돌이 있는 경우 linked list 형태의 노드를 타고 들어가 원하는 key값을 가진 데이터를 찾아야한다.       

반면 두 문자열이 거의 비슷하다 뒤쪽에서 차이가 나는 경우는 std::unordered_map이 빠를 것이다.     

일상에서 인간이 흔히 쓰는 문자열에는 prefix 즉 문자열 중 앞쪽에 위치하는 문자들이 같은 경우가 많다고 한다.    
예를 들면 "look up", "look for"이 있는 것 같다(?). 그래서 일상적인 글에서 키 값을 추출해서 사용한다면 std::unordered_map이 더 빠를 것이라고 생각한다.

극단적인 비교를 하였지만 이렇게 std::map과 std::unordered_map이 key값으로 int를 사용한 경우, pointer를 사용한 경우, std::string을 사용한 경우를 내부적으로 어떤 비교 함수가 사용되고 어떤 해시 함수가 사용하는 지를 알아둔다면 성능을 위해 어떤 container을 사용해야할지에 큰 도움을 줄 것이라 생각한다.     

아래의 이미지는 문자열이 긴 경우 다른 문자가 문자열의 처음에 등장하는 경우, 마지막에 등장하는 경우를 std::map과 std::unordered_map의 경우를 비교한 성능 비교도이다. ( 그래프 막대가 짧을 수록 더 빠르다는 의미이다. ) 
주목해야할 점은 긴 문자열의 첫 문자가 다른 std::string을 key로 사용한 경우 look-up 성능이 std::map이 std::unordered_map에 비해 더 좋다는 것이다.
![comparison](https://user-images.githubusercontent.com/33873804/117540476-a8250900-b04a-11eb-9434-a2d5476f902b.png)
