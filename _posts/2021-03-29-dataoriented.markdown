---
layout: post
title:  "Data Oriented Design ( DOTS )"
date:   2021-03-29
categories: ComputerScience
---
게임을 개발하다 보니 필연적으로 특정 오브젝트들은 반복적으로 접근해야하는 일이 생긴다.    
예를 들면 매 루프마다 엔티티에 접근 하고 매 루프마다 엔티티의 RendererComponent에 접근한다.     
대개 이런 접근은 for문 내에서 연속적으로 접근한다. 아래와 같이 말이다.    
```c++
for(unsigned int i = 0 ; i < EntityCount ; i++)
{
   Entity[i]->DoSomeThing();
}
```

```c++
std::vector<Entity*> SpawnedEntities;
void CreateNewEntity()
{
   SpawnedEntities.push_back(new Entity());
}
```

```c++
struct EntityBlock
{
   Entity mEntityPool[50];
   unsigned int mCreatedEntityCount;
}
std::vector<EntityBlock*> EntityBlockPool; 
void AddEntityBlockPool()
{
   EntityBlockPool.push_back(new EntityBlock());
}

std::vector<Entity*> mSpawnedEntities;
Entity* CreateNewEntity()
{
   EntityBlock* lastEntityBlock = EntityBlockPool.back();
   lastEntityBlock->mCreatedEntityCount++
   Entity* newEntity = &(lastEntityBlock->mEntityPool[lastEntityBlock->mCreatedEntityCount-1]);
   mSpawnedEntities.push_back(newEntity);
}

for(unsigned int i = 0 ; i < EntityCount ; i++)
{
   mSpawnedEntities[i]->DoSomeThing(); // Cache hitting probability is increased!!!!
}
```

Renderer Component나, Transform Component에도 적용 가능

references :        
https://en.wikipedia.org/wiki/Data-oriented_design