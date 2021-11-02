---
layout: post
title:  "C++에서 C# 함수 호출하기"
date:   2021-11-01
categories: ComputerScience GameEngine
---

리플랙션 시스템을 게임 엔진에 넣으면서 이 리플랙션 관련 데이터를 자동으로 추출해주는 툴이 필요하다고 생각이 들었다.           
이러한 툴들은 보통 C#으로 짜는 경우가 많다고 들어서 필자도 C#으로 코드를 짠 후 dll파일로 뽑아서 C++ 코드에서 Load해서 사용하기로 결정하였다.    

핵심은 [DllExport](https://github.com/3F/DllExport)라는 툴을 사용하는 것이다.       
이 툴을 이용해 managed 언어의 코드를 unmanaged 코드에서 실행할 수 있게 해준다. ( 그 반대도 가능하다 )            

이 툴을 사용하는 구체적인 방법은 여기서 설명하지는 않을 것이다. 깃허브에 들어가면 잘 설명되어 있다.           


C# 코드
```
[DllExport]
static public int c_Generate_clReflect_data(IntPtr stringPtr) // wchar_t*를 전달
{
    int result = 0;
    try
    {
        String args = Marshal.PtrToStringAuto(stringPtr); // C++에서 넘어오는 문자열 데이터를 C#의 유니코드 포맷에 맞게 변환, C#의 managed 되는 메모리에 복사.
```

C++ 코드에서 위에서 추출해낸 dll을 로드
```
	mLibrary = LoadLibrary( dll 파일 경로 );
```

C++에서 로드한 DLL의 export된 함수를 호출
```
template <typename RETURN_TYPE, typename ... Args>
bool SmartDynamicLinking::CallFunctionWithReturn(const char* const functionName, RETURN_TYPE& returnValue, Args&&... args)
{
    typedef RETURN_TYPE(__cdecl* functionType)(Args...);

    functionType function = reinterpret_cast<functionType>(_GetProcAddress(functionName));

    bool IsSuccess = false;
    if(function != nullptr)
    {
        __try
        {
            returnValue = function(std::forward<Args>(args)...);
            IsSuccess = true;
        }
        __except (filter(GetExceptionCode(), GetExceptionInformation()))
        {
            //... yey!
        }
        
        
    }

    return IsSuccess;
}
```

C#
```
using (var clScanConariL = new ConariL(dllPath, CallingConvention.Cdecl) )
{
    string arvs = GetClScanArgv(_clScanParameter);
    NativeString<CharPtr> unmanagedStringArgv = new NativeString<CharPtr>(arvs);

    result = clScanConariL.DLR.c_clscan<int>(unmanagedStringArgv); // managed 언어의 유니코드 문자열을 unmanaged 언어의 ASCII 문자열로 전달하기

    unmanagedStringArgv.Dispose();
}
```

[소스코드](https://github.com/SungJJinKang/DoomsEngine/tree/main/Doom3/Source/Core/DynamicLinkingHelper)