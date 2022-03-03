---
layout: post
title:  "Unreal Engine4의 이해가 안가는 코드.. ( FRenderTarget::ReadFloat16Pixels )"
date:   2022-03-03
categories: UnrealEngine4 UE4
---

입사해서 사용할 언리얼 엔진을 공부하고 있다. ( 곧 언리얼 엔진 엔진 프로그래머로 입사한다. )                      
언리얼 엔진을 사용해본 적이 지난 인턴 기간 3개월 밖에 없어서 급하게 공부하고 있다.           
언리얼 엔진 공식 문서를 쭉 읽어보고 토이 프로젝트를 하기로 결정했다.         
예전부터 [게임 "갓오브워4"의 Interactive한 바람이나 식생의 움직임에 대한 GDC 영상](https://youtu.be/MKX45_riWQA)을 보고 언젠가는 만들어보고 싶었는데 언리얼 엔진을 새로 배운 기념으로 만들어보기로 했다.          
입사까지 시간이 얼마 남지 않아 밤을 새며 만들고 있다.        

필자가 작성 중인 코드는 [여기서](https://github.com/SungJJinKang/UE4_Interactive_Wind_and_Vegetation_in_God_of_War) 볼 수 있다                            

-----------------------                

현재 바람의 방향의 Vector 값의 데이터를 가지는 텍스쳐 ( UTextureRenderTarget2D, RGB 값에 바람 Vector를 쓰는 방식 )가 있고 GPU를 통해서 바람 Vector를 그리는 방식이다.       
GPU를 사용하기 때문에 매우 빠르게 연산할 수 있다.                      

이해가 안가는 코드는 바람 텍스쳐의 디버깅을 위해 텍스쳐 값을 DRAM으로 읽어오는 과정에 있었다.            
         
```
FRenderTarget::ReadFloat16Pixels(TArray<FFloat16Color>& OutputBuffer,ECubeFace CubeFace)
```
           
이 함수를 통해 GPU에서 DRAM으로 텍스쳐 데이터를 읽어온다. ( CPU 게임 스레드에서 블록이 되지만 어차피 디버깅용이라 느려도 상관없다. )            
![gifff](https://user-images.githubusercontent.com/33873804/156438399-b6e9e75f-f73f-48f2-bbc5-83ba9268ca47.gif)          
그리고 읽어온 텍스쳐 값으로 위와 같이 3D 공간에 화살표를 그린다. ( DRAM으로 읽어오지 않고 GPU에서 바로 화살표를 그리려고 했는데 코드를 찾기도 귀찮고 어차피 디버깅용이라 그냥 대충 짜기로 했다. )                
        
문제는 "FRenderTarget::ReadFloat16Pixels" 함수 내부 코드에 있었다.      
이 함수는 매개변수로 텍스쳐의 색깔 데이터를 담을 변수를 레퍼런스로 받아서 색깔 데이터를 써준다.        
근데 내부 코드를 보자.         

``` 
bool FRenderTarget::ReadFloat16Pixels(TArray<FFloat16Color>& OutputBuffer,ECubeFace CubeFace)
{
	// Copy the surface data into the output array.
	OutputBuffer.Empty();
	OutputBuffer.AddUninitialized(GetSizeXY().X * GetSizeXY().Y);
	return ReadFloat16Pixels((FFloat16Color*)&(OutputBuffer[0]), CubeFace);
}
```          

이상함이 안느껴지는가?....          

**OutputBuffer를 "TArray::Empty" 함수로 내부 버퍼 ( Capacity ( Slack ) )를 힙 할당 해제해버리고 다시 "TArray::AddUninitialized"로 내부 버퍼를 다시 힙 할당**한다.       
필자의 경우에는 매프레임 이 함수를 호출해야하기 때문에 매 프레임 힙 재할당이 발생하는 것이다.                      
이해가 안갔다.      
어차피 "GetSizeXY().X * GetSizeXY().Y"만큼의 Element 개수가 필요하면 **그냥 "TArray::SetNumUninitialized" 함수를 사용하면 되는데 왜 힙 할당 해제를 하고 다시 힙 할당을 받아오는가.........** ( "TArray::SetNumUninitialized"를 사용하면 현재 Capacity ( Slack ) )보다 요구하는 Element 개수가 적으면 힙 할당을 하지 않고 내부 Element 개수의 숫자 값만 바꾸어주면된다. )           
    
TArray::Empty 함수를 호출하는 이유를 알 수 없었다.            
FFloat16Color 타입은 그냥 RGBA 값을 가지는 클래스로 따로 소멸자에서 하는 일도 없다. ( TArray::Empty 함수가 Element들의 소멸자를 호출해주는데 소멸자를 호출할 목적으로 이렇게 코드를 썼을리도 없다는 것이다. )                    

그럼 **불필요한 메모리 낭비를 막기 위한 목적인가?**                
그것도 아니다. 그럴 이유라면 **"TArray::SetNumUninitialized" 함수를 호출할 때 매개변수 "bAllowShrinking"에 true를 넘겨주어서 내부 버퍼를 재할당**하면된다.                          
그럼 **만약 매개변수 OutputBuffer의 현재 Capacity ( Slack )과 필요로하는 Element 개수가 같다면 힙 재할당을 피할 수 있다.**        
어찌되었든 "TArray::SetNumUninitialized" 함수를 사용하는 것이 이득이다.          
                
----------------------         
               
Raw 포인터를 매개변수로 받는 함수 ( 힙 재할당이 없는 )도 있지만 proteced 함수여서 내가 접근할 수 없었다.          
```
protected : 
bool FRenderTarget::ReadFloat16Pixels(FFloat16Color* OutImageData,ECubeFace CubeFace)
```        
             
------------------
          
바로 옆 "FRenderTarget::ReadPixels" 함수 ( 거의 똑같은 기능을 하는 )에는 "TArray::SetNumUninitialized" 함수를 사용한다.     
```
OutImageData.Reset();
FReadSurfaceContext Context =
{
    this,
    &OutImageData,
    InRect,
    InFlags
};

ENQUEUE_RENDER_COMMAND(ReadSurfaceCommand)(
    [Context](FRHICommandListImmediate& RHICmdList)
    {
        RHICmdList.ReadSurfaceData(
            Context.SrcRenderTarget->GetRenderTargetTexture(),
            Context.Rect,
            *Context.OutData,
            Context.Flags
            );
    });
FlushRenderingCommands();

return OutImageData.Num() > 0;

------>             

FORCEINLINE void ReadSurfaceData(FRHITexture* Texture,FIntRect Rect,TArray<FColor>& OutData,FReadSurfaceDataFlags InFlags)
{
    QUICK_SCOPE_CYCLE_COUNTER(STAT_RHIMETHOD_ReadSurfaceData_Flush);
    LLM_SCOPE(ELLMTag::Textures);
    ImmediateFlush(EImmediateFlushType::FlushRHIThread);  
    GDynamicRHI->RHIReadSurfaceData(Texture,Rect,OutData,InFlags);
}

------>     

virtual void RHIReadSurfaceData(FRHITexture* Texture, FIntRect Rect, TArray<FLinearColor>& OutData, FReadSurfaceDataFlags InFlags)
{
    TArray<FColor> TempData;
    RHIReadSurfaceData(Texture, Rect, TempData, InFlags);
    OutData.SetNumUninitialized(TempData.Num()); <--- !!!
    for (int32 Index = 0; Index < TempData.Num(); ++Index)
    {
        OutData[Index] = TempData[Index].ReinterpretAsLinear();
    }
}
```


-------------------------------------          

내가 이해하지 못하는 무언가가 있는지 모르겠다.          
일단은 내가 보기에는 이러하다...........         

일단은 언리얼 엔진 리포지터리에 [Pull Request](https://github.com/EpicGames/UnrealEngine/pull/8953)를 날려보아야겠다.           





