---
layout: post
title:  "Vector3 * Matrix4x4를 빠르게 하는 매우 Tricky한 방법"
date:   2021-04-08
categories: ComputerScience
---

이 글은 내 게임엔진에만 해당된다. 물론 다른 사람들에게도 해당될 수 있다.        
게임을 만들다보면 Vector3와 Matrix4x4를 곱해야 하는 일이 매우 매우 자주 생긴다.         
예를 들면 Position과 ModelMatrix를 곱하는 일 말이다.        
이 연산은 거의 매 프레임마다 수십 아니 수 천번 필요한 연산이다. 왜냐하면 모든 엔티티에 매 렌더링마다 연산을 해주어야하기 때문이다.    
물론 따로 저장을 해두었다가 Position, Rotation, Scale이 바뀐 경우에만 연산을 해줄 수도 있다. 어쨌든 기본적으로 모든 오브젝트가 움직인다는 가정하에 매 프레임마다 수 천번의 연산이 필요하다.      

실제로 프로파일링을 해보면 엄청난 시간을 잡아먹는다. 그럼 어떻게 빠르게 할 수 있을까???     
바로 SIMD를 이용하는 방법이다.       
```c++
template <>
[[nodiscard]] FORCE_INLINE Vector<4, float> operator*(const Vector<4, float>& vector4) const noexcept
{
	Vector<4, float> Result{nullptr};

	const M128F* A = reinterpret_cast<const M128F*>(this);
	const M128F* B = reinterpret_cast<const M128F*>(vector4.data());
	M128F* R = reinterpret_cast<M128F*>(&Result);

	// First row of result (Matrix1[0] * Matrix2).
	*R = M128F_MUL(M128F_REPLICATE(*B, 0), A[0]);
	*R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 1), A[1], *R);
	*R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 2), A[2], *R);
	*R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 3), A[3], *R);

	return Result;
}
```
위와 같이 SIMD연산을 이용해서 매우 빠르게 Vector4와 Matrix4x4를 곱해줄 수 있다.       
아니 근데 내가 가지고 있는 Position은 Vector3이다. 그럼 어떻게 곱한다는 말인가....       

parameter로 받은 vector3로 Vector4를 Construct하고 곱셈 연산을 하고 다시 Vector3를 Construct해야할 까?? 이렇게 두 번의 Constructor을 호출하는 건 생각보다 아주 느리다. 특히 렌더링 같은 부분에서는 조금이라도 명령어 수를 줄이는 게 중요한 데 Construct 호출하고 하는 데 필요한 명령어가 생각보다 많아 별로 추천하지 않는다. 그렇다면 그냥 SIMD 없이 일반적으로 일일이 곱해서 더해줘야하는 데 너무 느리다.      
아래의 함수가 SIMD 없이 scalar한 연산인 데 게임에서 쓰기에는 너무 느리다.        
```c++
template <typename X>
[[nodiscard]] FORCE_INLINE constexpr Vector<3, X> operator*(const Vector<3, X>& vector3) const noexcept
{
	return Vector<3, X>
	{
		this->columns[0][0] * vector3[0] + this->columns[1][0] * vector3[1] + this->columns[2][0] * vector3[2] + this->columns[3][0],
		this->columns[0][1] * vector3[0] + this->columns[1][1] * vector3[1] + this->columns[2][1] * vector3[2] + this->columns[3][1],
		this->columns[0][2] * vector3[0] + this->columns[1][2] * vector3[1] + this->columns[2][2] * vector3[2] + this->columns[3][2],
	};
}	
```

그렇다면 그냥 *reinterpret_cast<Vector4*>(&vector4) 해서 곱하면 될까??        
안된다!!! 왜냐면 Vector3 뒤에 어떤 데이터가 있는지도 모르고 뒤 데이터가 float형 값 1이여야 정상적으로 연산을 할 수 있다.         

그래서 조금 Tricky한 방법을 사용하려 한다.                
Vector3 뒤에 float형 변수를 하나 넣는 것이다!!!        
그럼 우리가 필요한건 우선 16byte에 align이 되어 있어야 하고 뒤에 float형 padding이 와야한다.                 
```c++
struct alignas(16) AlignedTempVec3
{
    Vector<3, float> Vec3;
    const float padding{ 1.0f };
};
inline static AlignedTempVec3 _AlignedTempVec3_Parameter{};
inline static AlignedTempVec3 _AlignedTempVec3_Result{};

template <>
[[nodiscard]] FORCE_INLINE Vector<3, float> operator*(const Vector<3, float>& vector) const noexcept
{
    std::memcpy(&_AlignedTempVec3_Parameter, &vector, sizeof(Vector<3, float>));

    const M128F* A = reinterpret_cast<const M128F*>(this);
    const M128F* B = reinterpret_cast<const M128F*>(&_AlignedTempVec3_Parameter);
    M128F* R = reinterpret_cast<M128F*>(&_AlignedTempVec3_Result);

    // First row of result (Matrix1[0] * Matrix2).
    *R = M128F_MUL(M128F_REPLICATE(*B, 0), A[0]);
    *R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 1), A[1], *R);
    *R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 2), A[2], *R);
    *R = M128F_MUL_AND_ADD(M128F_REPLICATE(*B, 3), A[3], *R);

    return _AlignedTempVec3_Result.Vec3;
}
```
복잡해 보이는가?? 성능을 위해서 이 정도는 감당하자.        
우선 AlignedTempVec3이라는 struct을 만들었는 데 처음 오는 변수가 Vector3이고 뒤에 float형 padding이 하나 온다. 또한 16byte에 align되어 SIMD 연산을 할 수 있게 만들었다.      
일반적으로 Matrix4x4와 Vector3를 곱할 때 parameter로 넘어오는 vector3가 16byte에 align되어 있다고 가정하는건 멍청한 짓이다.       
높은 확률로 16byte에 not align되어 있을 가능성이 높다.(그렇다고 align된 vector3 받는건 실수할 가능성이 높고 매우 귀찮아진다.)         
그래서 일단은 parameter로 받는 vector3에는 제약을 두지 않았다.     
그럼 곱하는 함수 내에서 해결해야한다.       
그래서 AlignedTempVec3 struct를 만들어 이 align된 struct로 parameter의 데이터를 복사해 넘겨서 SIMD 연산을 하는 것이다.       
연산 중간에 필요한 align된 변수 AlignedTempVec3를 임시로 그때그떄 할당하기 보다는 미리 할당해두어 최대한(!) 연산 속도를 줄이려 노력하였다.            
