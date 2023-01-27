---
layout: post
title:  "자녀 클래스만 Polymorphic 클래스인 경우 this 참조시 발생하는 문제"
date:   2021-10-21
tags: [C++]
---

자녀 클래스만 Polymorphic 클래스인 경우 **Non Polymorphic 부모 클래스안에서 참조하는 this**와 **자녀 Polymorphic 클래스에서 참조하는 this**의 **주소가 다르다!!**             

```
+------------------------------------+ <---- `this` value for `Derived ( Polymorphic Class )`
| VMT pointer introduced by Derived  |
+------------------------------------+ <---- `this` value for `Base ( Non Polymorphic Class )`
| Base data                          |
+------------------------------------+
| Derived data                       |
+------------------------------------+
```

그래서 혹시나 **부모 클래스에서 this를 이용해서 자녀 클래스 타입으로 캐스팅을 한다면** 절대로 reinterpret_cast를 사용하면 안된다.         
**반드시 dynamic_cast를 사용하여서 위의 vtable pointer의 사이즈를 고려해서 offset을 추가하여 캐스팅** 해준다.           

reference : [https://stackoverflow.com/questions/11593783/mismatch-of-this-address-when-base-class-is-not-polymorphic-but-derived-is](https://stackoverflow.com/questions/11593783/mismatch-of-this-address-when-base-class-is-not-polymorphic-but-derived-is)         

