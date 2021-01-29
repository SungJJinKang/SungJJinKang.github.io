---
layout: post
title:  "C# Dictionary에서 key를 enum이나 struct으로 사용 시 매우 주의해야할 점"
date:   2018-03-29
categories: C#
---
Dictionary key 값으로 enum이나 struct을 사용하면 알게 모르게 사용하면서 엄청난 양의 가비지가 발생한다.   

그 이유는 enum이나 struct은 Equals, GethashCode가 구현되지 않았기 때문에 Dictionary의 Contain 함수를 사용할 때 비교를 위해 부모 클래스 형인 Object로 박싱하여 Object끼리 비교를 해야하기 때문이다. 예를 들면 A라는 딕셔너리가 있고 A.ConatinKey(키값) 이나 A[키값]을 사용 할 때 모두 내부적으로는 비교구문이 실행된다. 비교를 위해서는 비교 함수를 사용하는데 이때 자신이 만든 enum , struct은 Equals, GethashCode가 구현이 되어 있지 않아 박싱을 해야하고 박싱을 할 때 가비지가 발생한다.   

이러한 문제는 프로그램이 커질 수록 더 많은 양의 가비지를 발생시켜 나중에는 문제가 될 가능성이 많다.   
이를 해결하기 위해서는 IEqualityComparer<T> 인터페이스를 상속받는 클래스를 선언하여 Dictionary 인스턴스를 생성할 때 생성자로 넣어주면 된다.   

아래와 같이 struct(Vector3) , enum(MoviePoint)에 대한 Comparer 클래스를 생성해주면 된다.   

```c#
public class Vector3Comparer : IEqualityComparer<Vector3>
{
    public bool Equals(Vector3 x, Vector3 y)
    {
        return x.x == y.x && x.y == y.y && x.z == y.z;
    }

    public int GetHashCode(Vector3 vector)
    {
        return vector.x.GetHashCode() ^ vector.y.GetHashCode() << 2 ^ vector.z.GetHashCode() >> 2;
    }
}

public class MoviePointComparer : IEqualityComparer<MoviePoint>
{
    public bool Equals(MoviePoint x, MoviePoint y)
    {
        return (int)x == (int)y;
    }



    public int GetHashCode(MoviePoint obj)
    {
        return ((int)obj).GetHashCode();
    }
}

Dictionary<Vector3, bool> dic = new Dictionary<Vector3, bool>(new Vector3Comparer());
Dictionary<MoviePoint, bool> dic = new Dictionary<MoviePoint, bool>(new MoviePointComparer ());
```