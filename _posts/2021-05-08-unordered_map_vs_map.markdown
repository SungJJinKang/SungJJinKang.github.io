---
layout: post
title:  "std::map vs std::unordered_map ( MSVC STL )"
date:   2021-05-08
categories: C++
---

이렇게 차이가 나는 이유는 std::map과 std::unordered_map이 look-up을 하는 방식에서 차이가 있다.      

std::map은 트리 형태로 데이터를 배치하여 찾으려는 key값과 현재 위치한 node의 key값을 비교하여 그 비교 함수의 결과에 따라 트리의 왼쪽 자식 노드로 갈지 오른쪽 자식 노드로 갈지를 결정하여 이를 반복해 원하는 key값의 node를 찾는 방식이다. 시간 복잡도로는 O(n)의 시간 복잡도를 가진다.                     
std::unordered_map은 hash table을 이용하는 방식이다. 우선 key값을 가지고 hash value를 계산한 후 그 hash value를 인덱스로 사용하여 bucket array의 특정 element 접근한다. 그럼 이 array의 element는 linked list로 같은 hash value를 가지는 key-value pair가 저장된다. hash collision이 발생하는 경우 시간이 더 걸릴 수 있지만 기본적으로는 O(1)의 복잡도를 가진다.       

여기까지만 들으면 hash value를 계산하는 방법이 당연히 빠를 것 같아 보인다. 하지만 그렇지 않다.    
일반적으로는 std::unordered_map이 std::map보다 look-up시 빠르지만 hash value를 계산하는 hash function에서 많은 시간이 소모되는 경우 std::map보다 더 느린 경우가 생긴다.       

MSVC STL에서 std::unordered_map은 hash value를 구할 때는 Fnv1a이라는 알고리즘이 사용되는 데 이 알고리즘은 unsigned char*의 포인터들을 1씩 더해가며 포인터 값을 게속해서 xor을 하는 방식으로 hash 값을 구한다. 즉 아래 함수에서 보이는 parameter _Count가 hash 함수 성능에 중요한 영향을 준다.      
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
그럼 이 _Count는 어떻게 정해질까? 여기서 std::unordered_map을 사용할 시 key값에 따른 성능의 차이가 나뉘어진다. 일반적인 integer, float는 값의 주소를 가지고 hash를 만든다. 주소는 64bit 환경에서는 8byte일테니 총 8번의 루프를 통해서 hash value를 계산할 수 있다. 포인터의 경우는 그 포인터가 가지고 있는 주소를 가지고 8번의 루프를 거쳐 hash value가 결정된다.    
그럼 integer나 float, 포인터의 경우에는 std::unordered_map을 써도 나쁘지 않아 보인다. 기껏해야 8번의 루프로 hash value를 구하면 그 후에는 O(1)만에 원하는 key-value를 찾을 수 있으니깐 말이다.      

하지만 key값을 std::string으로 사용한 경우 얘기가 달라진다. std::string의 경우에는 string의 문자의 개수만큼 루프를 돌아서 hash value를 구할 수 있다. 그럼 std::string 문자열의 길이가 100인 경우 hash value를 구하기 위해서 100번의 루프를 돌아야 한다는 것이다.      
그럼 여기서 개발자는 std::string을 key값으로 사용할 때 std::map을 사용해야할지, std::unordered_map을 사용해야할지를 정해야한다.

이해를 돕기 위해 std::map이 어떻게 작동하는지 간단히 설명하겠다.           
std::map은 hash값을 계산할 필요는 없지만 tree구조의 node들을 이동하며 key값을 비교하여 왼쪽 자식 노드를 택할지, 오른쪽 자식 노드를 택할지를 결정한다. 대걔 이렇게 node를 한 단계식 찾아가는 동작으로 인해 O(n)의 시간 복잡도를 가져 std::map이 std::unordered_map보다 느리다고 여겨진다.             
그런데 std::string을 key값으로 사용하는 경우 얘기가 달라진다.          
두 문자열을 비교하는데는 strcmp가 쓰이는 데 이 함수에 중요한 점이 있다.       
(msvc에서는 std::string의 길이가 small size optimization의 Buffer 사이즈보다 작은 경우에는 그냥 std::string의 길이로 key값을 비교한다.)  
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
strcmp는 index를 하나씩 올려가며 첫 문자부터 끝 문자까지 한 문자씩 비교를 하지만 다른 문자를 만나는 순간 바로 비교 결과를 return한다는 것이다.         
이 말은 즉 key값으로 사용된 문자열 중 앞쪽에 위치한 문자들이 서로 다르면 두 문자열을 비교하는 함수를 n번의 루프가 필요없이 빨리 끝낼 수 있다는 것이다.
```
아래의 두 문자열을 총 18번의 loop 후 비교 결과를 return 할 수 있다.
"aaaaaaaaaaaaaaaaab"
"aaaaaaaaaaaaaaaaac"

반면 아래의 두 문자열은 한번의 loop만으로 비교 결과를 return 할 수 있다. 
"baaaaaaaaaaaaaaaaa"
"caaaaaaaaaaaaaaaaa"
```

그러면 이제 우리는 std::string을 key값으로 사용하는 경우 std::map과 std::unordered_map 중 어떤 경우가 빠를지 어느 정도 예측을 할 수 있다.      
문자열이 짧은 경우는 당연히 hash value를 금방 구할 수 있어 std::unordered_map이 압도적으로 빠르다.       
문자열이 긴 경우는 두 가지 경우로 나뉜다. 두 문자열에서 문자열의 앞쪽에 다른 문자가 있는 경우에는 hash value를 계산하는 데 O(n)이 소요되는 std::unordered_map보다 std::map이 더 빠를 수 있다. 아무리 std::map이 tree구조라 하여도 한번의 비교로 어떤 자식 노드를 탐색해야하는지가 결정이 되면 std::unordered_map이 hash value 계산이 끝나기도 전에 std::map은 원하는 key-value를 찾을 수도 있을 것이다.            
반면 두 문자열이 거의 비슷하다 뒤쪽에서 차이가 나는 경우는 std::unordered_map이 빠를 것이다.     

일상에서 인간이 흔히 쓰는 문자열에는 prefix 즉 문자열 중 앞쪽에 위치하는 문자들이 같은 경우가 많다고 한다.    
예를 들면 "look up", "look for"이 있는 것 같다(?). 그래서 일상적인 글에서 키 값을 추출해서 사용한다면 std::unordered_map이 더 빠를 것이라고 생각한다.

극단적인 비교를 하였지만 이렇게 std::map과 std::unordered_map이 key값으로 int를 사용한 경우, pointer를 사용한 경우, std::string을 사용한 경우를 내부적으로 어떤 비교 함수가 사용되고 어떤 해시 함수가 사용하는 지를 알아둔다면 성능을 위해 어떤 container을 사용해야할지에 큰 도움을 줄 것이라 생각한다.     

아래의 이미지는 문자열이 긴 경우 다른 문자가 문자열의 처음에 등장하는 경우, 마지막에 등장하는 경우를 std::map과 std::unordered_map의 경우를 비교한 성능 비교도이다. ( 그래프 막대가 짧을 수록 더 빠르다는 의미이다. ) 
주목해야할 점은 긴 문자열의 첫 문자가 다른 std::string을 key로 사용한 경우 look-up 성능이 std::map이 std::unordered_map에 비해 더 좋다는 것이다.
![comparison](https://user-images.githubusercontent.com/33873804/117540476-a8250900-b04a-11eb-9434-a2d5476f902b.png)
