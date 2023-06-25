---
layout: post
title:  "언리얼 엔진4의 Exception Handling, Crash Reporter(Windows 플랫폼)"
date:   2023-09-24
tags: [ComputerScience, UE]
---          

언리얼 엔진 4.27, Windows 기준

--------------------------          
             
사전 지식           
              
C++ Exception : "throw std::exception("")와 같은 C++ Exception들을 말함         
Asynchronous Exceptions : "Floating point exceptions", "Access Violation"이나 "Division by zero"와 같은 하드웨어에서 throw하는 Exception                   
"Structured Exception Handling(SEH) :  Asynchronous Exceptions을 Handling하는 것             
               
--------------------------         

언리얼 엔진4 구현상 대부분의 경우 C++ Exception Handling을 수행하지 않는다.            
ModuleRules.cs의 bEnableExceptions 변수의 기본 값이 false이다.               
C++ Exception(EX. throw std::exception(""))을 사용하는 "써드 파티 라이브러리를 사용하는 모듈"이나 UnrealHeaderTool들에 한해 활성화되어 있다.           
```c++
ModuleRules.cs

public class ModuleRules
{
	...
	...
	...

	/// <summary>
	/// Enable exception handling
	/// </summary>
	public bool bEnableExceptions = false;

	...
	...
	...
}

-------------------

VCToolChain.cs

// Enable C++ exceptions when building with the editor or when building UHT.
if (CompileEnvironment.bEnableExceptions)
{
	// Enable C++ exception handling, but not C exceptions.
	Arguments.Add("/EHsc"); // !!
}
else
{
	// This is required to disable exception handling in VC platform headers.
	Arguments.Add("/D_HAS_EXCEPTIONS=0");
}
```
MSVC의 "/EHsc" 플래그는 try{} 블록 내에 C++ Exception이 발생할 가능성이 있는 코드에 대해 Exception 필터링을 위한 코드를 생성한다.(플래그 중 'c'의 의미는 extern "C"는 절대로 C++ Exception을 발생시키지 않는다고 간주한다 [참고](https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically)) 그리고 "Exception 필터링"은 Exception을 Handling하는 동안 스택을 unwind할 때 C++ 로컬 오브젝트의 소멸자가 호출되는 것을 보장해준다.(소멸자 호출 코드를 생성해준다)([RHII](https://en.cppreference.com/w/cpp/language/raii) !)                           
여기서 "발생할 가능성이 있는 코드에 대해 Exception 필터링을 위한 코드를 생성한다"이라고 말했는데, 이 의미는 컴파일러가 컴파일 중 코드를 분석하면서 C++ Exception이 절대로 발생하지 않는다는 것이 확인이 되는 경우에는 Exception Handling 코드를 생성하지 않는다는 의미이다.         
C++ Exception뿐만아니라 "Structured Exception Handling(SEH)"를 수행하는 "/EHa" 플래그도 있다. C++ Exception이 발생할 가능성이 있는 경우에만 Exception 필더링 코드를 생성하는 "/EHsc"와 달리, "/EHa"는 "Access Violation"와 같은 언제나 발생할 수 있는 Exception을 Handling하기 때문에 조건에 상관없이 항상 Exception 필터링 코드를 생성한다. ( [참고](https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically) )                                 
                
위에서 "언리얼 엔진4 구현상 기본적으로는 C++ Exception Handling을 수행하지 않는다."고 말했는데 Access Violation과 같은 Asynchronous Exceptions들을 Handling하기 위한 SEH는 수행합니다.          
그럼 "/EHa" 플래그와 와 try, catch문을 사용하나 싶지만, 이 경우 Asynchronous Exceptions뿐만 아니라 C++ Exceptions까지도 Handling하기 때문에, 우리 Asynchronous Exceptions에 대한 Handling만을 원하므로 "__try", "__except"라는 윈도우 특화된 키워드를 사용합니다.           
언리얼 엔진의 윈도우 플랫폼 코드를 보면 몇몇 곳에서 "__try", "__except"을 사용하시는 것을 보실 수 있을겁니다.         
"__try", "__except" 사용시 장점이 있는데 바로 발생한 Exception에 대한 정보들을 알 수 있고 그에 따라 여러 방식으로 Exception을 핸들링할 수 있다는 것입니다. 이는 아래에서 더 자세히 설명하겠습니다.      
          
"/EHa" 플래그 없이 "__try", "__except"를 사용할시 Stack unwinding시 로컬 C++ 오브젝트의 소멸자가 호출된다는 것이 보장이 되지는 않습니다.          
사실 Asynchronous Exceptions가 발생했다는 것 자체가 프로그램 상태가 정상이 아닌 상태라는 의미인데 이 상태로 빌빌거리며 프로그램을 계속 진행시켜도 결국 뒤에서 문제가 터지기 마련이고 뒤에서 터진다면 버그 추적이 더 어려워질 것입니다.      
그래서 언리얼 엔진은 Asynchronous Exceptions 발생시 Crash에 대한 리포트 후 프로그램을 종료하는 방식으로 SEH를 수행합니다.                      
         
아래는 언리얼 엔진4에서 "__try", "__except"를 사용하는 코드들입니다.  
```c++
uint32 FRunnableThreadWin::GuardedRun()
{
	...
	...
	...

#if UE_BUILD_DEBUG
	if (true && !GAlwaysReportCrash)
#else
	if (bNoExceptionHandler || (FPlatformMisc::IsDebuggerPresent() && !GAlwaysReportCrash))
#endif // UE_BUILD_DEBUG
	{
		ExitCode = Run();
	}
	else
	{
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try // !!
#endif // !PLATFORM_SEH_EXCEPTIONS_DISABLED
		{
			ExitCode = Run();
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except (ReportCrash( GetExceptionInformation() ))
		{

		...
		...
		...
}

virtual uint32 FRenderingThread::Run(void) override
{
	FMemory::SetupTLSCachesOnCurrentThread();
	FPlatformProcess::SetupRenderThread();

#if PLATFORM_WINDOWS
	bool bNoExceptionHandler = FParse::Param(FCommandLine::Get(), TEXT("noexceptionhandler"));
	if ( !bNoExceptionHandler && (!FPlatformMisc::IsDebuggerPresent() || GAlwaysReportCrash))
	{
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try // !!
#endif
		{
			RenderingThreadMain( TaskGraphBoundSyncEvent );
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except(FlushRHILogsAndReportCrash(GetExceptionInformation()))
		{
		...
		...
		...
}


LAUNCH_API int32 LaunchWindowsStartup( HINSTANCE hInInstance, HINSTANCE hPrevInstance, char*, int32 nCmdShow, const TCHAR* CmdLine )
{
	...
	...
	...

	// When we're running embedded, assume that the outer application is going to be handling crash reporting
#if UE_BUILD_DEBUG
	if (GUELibraryOverrideSettings.bIsEmbedded || !GAlwaysReportCrash)
#else
	if (GUELibraryOverrideSettings.bIsEmbedded || bNoExceptionHandler || (FPlatformMisc::IsDebuggerPresent() && !GAlwaysReportCrash))
#endif
	{
		// Don't use exception handling when a debugger is attached to exactly trap the crash. This does NOT check
		// whether we are the first instance or not!
		ErrorLevel = GuardedMain( CmdLine );
	}
	else
	{
		// Use structured exception handling to trap any crashes, walk the the stack and display a crash dialog box.
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try // !!
#endif
 		{
			GIsGuarded = 1;
			// Run the guarded code.
			ErrorLevel = GuardedMainWrapper( CmdLine );
			GIsGuarded = 0;
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( GEnableInnerException ? EXCEPTION_EXECUTE_HANDLER : ReportCrash( GetExceptionInformation( ) ) )
		{
	...
}


/**
 * The inner exception handler catches crashes/asserts in native C++ code and is the only way to get the correct callstack
 * when running a 64-bit executable. However, XAudio2 doesn't always like this and it may result in no sound.
 */
#ifdef _WIN64
	bool GEnableInnerException = true;
#else
	bool GEnableInnerException = false;
#endif

/**
 * The inner exception handler catches crashes/asserts in native C++ code and is the only way to get the correct callstack
 * when running a 64-bit executable. However, XAudio2 doesn't like this and it may result in no sound.
 */
LAUNCH_API int32 GuardedMainWrapper( const TCHAR* CmdLine )
{
	int32 ErrorLevel = 0;
	if ( GEnableInnerException )
	{
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
	 	__try // !!
#endif
		{
			// Run the guarded code.
			ErrorLevel = GuardedMain( CmdLine );
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( ReportCrash( GetExceptionInformation() ), EXCEPTION_CONTINUE_SEARCH )
		{
			// Deliberately do nothing but avoid warning C6322: Empty _except block.
			(void)0;
		}
#endif
```        
특히 스레드, 프로세스의 Entry 부분에서부터 __try, __except으로 코드를 감싸 SEH를 수행하는 것을 볼 수 있습니다.          
      
위에서 "__try", "__except" 사용시 장점이 있는데 바로 발생한 Exception에 대한 정보들을 알 수 있고 그에 따라 여러 방식으로 Exception을 핸들링할 수 있다는 장점이 있다고 말했는데 아래의 msdn 코드 예시를 보시면 알 수 있습니다.
```c++
// exceptions_try_except_Statement.cpp
// Example of try-except and try-finally statements
#include <stdio.h>
#include <windows.h> // for EXCEPTION_ACCESS_VIOLATION
#include <excpt.h>

int filter(unsigned int code, struct _EXCEPTION_POINTERS *ep)
{
    puts("in filter.");
    if (code == EXCEPTION_ACCESS_VIOLATION)
    {
        puts("caught AV as expected.");
        return EXCEPTION_EXECUTE_HANDLER;
    }
    else
    {
        puts("didn't catch AV, unexpected.");
        return EXCEPTION_CONTINUE_SEARCH;
    };
}

int main()
{
    int* p = 0x00000000;   // pointer to NULL
    puts("hello");
    __try
    {
        puts("in try");
        __try
        {
            puts("in try");
            *p = 13;    // causes an access violation exception;
        }
        __finally
        {
            puts("in finally. termination: ");
            puts(AbnormalTermination() ? "\tabnormal" : "\tnormal");
        }
    }
    __except(filter(GetExceptionCode(), GetExceptionInformation()))
    {
        puts("in except");
    }
    puts("world");
}


출력

hello
in try
in try
in filter.
caught AV as expected.
in finally. termination:
        abnormal
in except
world
```
각각의 코드에 대한 자세한 설명은 [여기](https://learn.microsoft.com/ko-kr/cpp/cpp/try-except-statement?view=msvc-170)를 참고해주시면 됩니다.        

앞에서 언리얼 엔진은 기본적으로 C++ Exception에 대한 Handling을 수행하지 않는다고 말했는데 그 이유는 여러가지가 있는데 코드 유지보수(언리얼 엔진의 코드가 워낙 방대하다보니 Exception 발생시 이를 일일이 처리/대응 코드를 유지 보수하기 어려움, 언리얼 엔진은 그냥 프로그램을 종료시킴), Exception Handling 코드로 인한 바이너리 사이즈가 커짐등이 대표적입니다. 그리고 Exception 발생이 이를 Handling하는 것보다는 그냥 바로 죽여서 원인을 찾고 픽스하는 것이 코드 유지보수 차원에서 더 유리할 수 있습니다.                
위의 이유들로 언리얼 엔진은 Exception Handling보다는 check, ensure문으로 문제를 곧 바로 알려주는 방식을 사용합니다.       
             
간혹 Exception Handling을 수행하는 경우 그렇지 않은 경우에 비해 성능적으로 느리다는 얘기가 있는데, 정확히는 "Exception 발생시"에 상대적으로 느리다가 맞습니다.       
일단 현대에 대부분의 컴파일러들은 Zero Cost Exception 모델을 가지고 있기 때문에 Exception이 발생하지 않는 상황에서는 Exception Handling이 활성화되어 있는 환경과 그렇지 않은 환경에서의 런타임 비용은 거의 동일합니다(x86 환경에서는 try문 진입시 exception handler 주소를 스택에 저장하기 동작을 런타임에 수행하기 때문에 상대적으로 느립니다. 반면 x64 환경에서 최근의 컴파일러들은 exception 주소에 대한 exception handler 주소를 일종의 룩업 테이블에 저장하기 때문에 추가적으로 발생하는 런타임 비용이 매우 쌉니다.)             
                          
Exception이 발생한 상황에서는 Exception Handling을 수행하는 경우에 그렇지 않은 경우보다 비용이 상당히 클 수 있습니다          
다만 Exception이 발생한 상황에서의 비용도 환경에 따라 다릅니다. 
(환경, 조건과 상관 없이 Exception 필터링을 위한 명령어들이 추가되므로 바이너리 사이즈가 약간 늘어단다는 단점은 있습니다.)
(레퍼런스 : [1](https://stackoverflow.com/a/16785259), [2](https://stackoverflow.com/a/4574319), [3](https://stackoverflow.com/questions/4414027/visual-c-unmanaged-code-use-eha-or-ehsc-for-c-exceptions))                
             
--------------------          
        
이제 언리얼 엔진4의 Exception Handling 코드를 좀 더 자세히 살펴보겠습니다.              




-------------------


EngineUnhandledExceptionFilter의 주석 자세히 읽어보면 좋을 듯
SEH나 C++ Exception으로 처리되지 않는 예외들
[SetUnhandledExceptionFilter](https://learn.microsoft.com/ko-kr/windows/win32/api/errhandlingapi/nf-errhandlingapi-setunhandledexceptionfilter)         

---------------------

UE4 Crash Report
SetUnhandledExceptionFilter로 UnhandledException Handling Function(함수명 : EngineUnhandledExceptionFilter) 등록

FCrashReportingThread 생성해서 CrashEvent 기다림
Exception 발생시 해당 스레드가 UnhandledException Handling Function 실행 -> UnhandledException Handling Function에서 CrashEvent 등록 후 FCrashReportingThread가 Report를 완료하기를 대기함
CrashEvent 등록되면 FCrashReportingThread는 그 이벤트를 받아서 Crash Report 수행(혹시나 Crash Report 동작 코드에서 크래시가 날 수 있으니 해당 코드들은 SEH로 감싸져있음)       
Crash Report 동작은 별도 CrashReportClient 프로세스를 생성해서 크래시 관련 데이터를 해당 프로세스에 전달해서 CrashReportClient가 Report를 수행

에디터는 CrashReportClient가 에디터 시작 시점부터 계속돔(USE_CRASH_REPORTER_MONITOR=1)
ReportCrashForMonitor 확인
 
-----------------------

크래시 발생시 GMalloc은 FGenericPlatformMallocCrash로 대체됨
```c++
void* FGenericPlatformMallocCrash::Malloc( SIZE_T Size, uint32 Alignment )
{
	const uint32 Size32 = (uint32)Size;
	if( Alignment > 16 )
	{
		UE_DEBUG_BREAK();
		FPlatformMisc::LowLevelOutputDebugString( TEXT( "Alignment > 16 is not supported\n" ) );
	}

	if( IsOnCrashedThread() )
	{

	...
	...
	...
}

bool FGenericPlatformMallocCrash::IsOnCrashedThread() const
{
	// Suspend threads other than the crashed one to prevent serious memory errors.
	// Only the crashed thread can do anything meaningful from here anyway.
	if( CrashedThreadId == FPlatformTLS::GetCurrentThreadId() )
	{
		return true;
	}
	else
	{
		FPlatformProcess::SleepInfinite(); // 크래시가 발생한 스레드 외의 스레드
		return false;
	}
}
```

references : [https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically](https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically), [https://stackoverflow.com/a/16785259](https://stackoverflow.com/a/16785259), [https://stackoverflow.com/a/4574319](https://stackoverflow.com/a/4574319), [https://stackoverflow.com/a/7249442](https://stackoverflow.com/a/7249442), [https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170](https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170), [https://kuaaan.tistory.com/435](https://kuaaan.tistory.com/435)