---
layout: post
title:  "Data Oriented Design에 대해, 그리고 매우 강력하고 이해하기 쉬운 예제"
date:   2021-04-04
categories: ComputerScience
---

일반적으로 Object Oriented Design, OOT는 현실 세계가 돌아가는 방식대로 코드를 작성하는 것이다.       
Dictionary를 작성한다고 가정해보자, Dictionary는 Key와 Value가 있고 Key를 통해 Value를 찾는 Structure이다.    
```
---------------------
|   Key   |  Value  |
|---------|---------|
|         |         |  
|---------|---------|   --->  Find Key -> Get Value
|         |         |
|---------|---------|
|         |         |
|---------|---------|

```
위의 그림이 현실 세계가 돌아가는 대로 생각을 한 방식이다.      
Key와 Value는 연관되어 있으니 서로를 한 묶음으로 생각할 것이다.       
그래서 아래와 같이 코드를 짜기 쉽다.     
```c++
struct Data
{
   int Key;
   int Value;
}
Data DictionaryData[];
```
그렇지만 이건 현실 세상이 돌아가는 방식이고 우리는 프로그래머다.      
프로그래머는 데이터를 다루는 직업이다. 데이터를 어떻게 효율적으로 다룰지를 생각해야한다.             
프로그래머는 아래와 같이 생각해야한다.     
```
-----------                                     -----------
|   Key   |                                     |  Value  |
|---------|                                     |---------|
|         |                                     |         | 
|---------|    ->  Find Key And Get Index ->    |---------|   ->   Get Value  
|         |                                     |         |
|---------|                                     |---------|
|         |                                     |         |
|---------|                                     |---------|
```
```c++
struct Dictionary
{
   int Key[];
   int Value[];
}
Dictionary DictionaryData;
```
Key와 Value는 별개의 데이터이다.    
그래서 현실 세계에서 Key와 Value과 관련이 되어 있다는 데 중심을 두기 보다는 어떻게 하면 빠르게 데이터를 찾을 수 있을 지를 고민하고 이렇게 코드를 짠다.   
바로 캐시를 이용하는 것이다.                

Data Oriented Design에 관심이 많다면 [이 강의](https://youtu.be/rX0ItVEVjHc) [이 강의2](https://youtu.be/yy8jQgmhbAU)를 꼭 보기 바란다.  ( 나중에 내가 직접 자막을 달아 볼까 생각 중이다. )

-----------------------

이제 Data oriented Design의 매우 강력한 예를 보여 줄 것이다.

```c++
Quat mQuat;

void TranslateNode(Node* nodeT, int count, const Vector3& d)
{
   for( unsigned int i = 0 ; i < count ; i++ )
   {
      node->mPosition += mQuat * d;
   }
}
```

```c++
struct NodeTranslate
{
   Vector3 mPositon;
   Quat mQuat;
};
void TranslateNode(NodeTranslate* nodeTranslate, int count, const Vector3& d)
{
   for( unsigned int i = 0 ; i < count ; i++ )
   {
      nodeTranslate->mPosition += nodeTranslate->mQuat * d;
   }
}
```

위의 코드의 문제는 무엇일까??    
자

매번 새롭게 캐쉬를 써야함

새로운 방법은 4개의 Entity들은 같은 CacheLine에 포함되어서 4개의 Entity의 mLocalPoint는 캐쉬에서 바로 가져옴
references :        
https://en.wikipedia.org/wiki/Data-oriented_design       
https://youtu.be/rX0ItVEVjHc         
