---
layout: post
title:  "자녀 클래스만 Polymorphic 클래스인 경우 this 참조시 발생하는 문제"
date:   2021-10-21
categories: ComputerScience C++
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

reference : [https://stackoverflow.com/questions/11593783/mismatch-of-this-address-when-base-class-is-not-polymorphic-but-derived-is](https://stackoverflow.com/questions/11593783/mismatch-of-this-address-when-base-class-is-not-polymorphic-but-derived-is)         

