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
(사실 표준 "try", "catch" 키워드도 windows에서는 내부적으로 "__try", "__except"을 이용해 구현됩니다.)           
          
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
앞서 본대로 언리얼 엔진의 스레드, 프로세스의 Entry 부분에는 __try, __except로 코드를 감싸서 SHE를 수행합니다.       
```c++
LaunchWindows.cpp

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
	 	__try
#endif
		{
			// Run the guarded code.
			ErrorLevel = GuardedMain( CmdLine ); // 엔진의 static main 함수
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( ReportCrash( GetExceptionInformation() ) )
		{
			// Deliberately do nothing but avoid warning C6322: Empty _except block.
			(void)0;
		}
#endif
	}
	else
	{
		// Run the guarded code.
		ErrorLevel = GuardedMain( CmdLine );
	}
	return ErrorLevel;
}
```
그럼 GuardedMain 함수가 호출하는 함수에서 엔진의 여러 동작을 수행하다 Access Violation이 발생하였다고 가정해봅시다.        
그럼 앞서 말한대로 __except에서 Asynchronous exception을 받아서 handling을 수행합니다. ReportCrash 함수의 결과에 따라 Exception이 발생한 명령어로 다시 돌아갈지, exception filtering 코드를 수행할지(다만 이 케이스에서는 (void)0이지만요)가 결정됩니다.       
         
ReportCrash 함수는 아래와 같이 구현되어 있습니다.        
```c++
int32 ReportCrash( LPEXCEPTION_POINTERS ExceptionInfo )
{
#if !NOINITCRASHREPORTER
	// Only create a minidump the first time this function is called.
	// (Can be called the first time from the RenderThread, then a second time from the MainThread.)
	if (GCrashReportingThread)
	{
		if (FPlatformAtomics::InterlockedIncrement(&ReportCrashCallCount) == 1) // 
		{
			GCrashReportingThread->OnCrashed(ExceptionInfo);
		}

		// Wait 60s for the crash reporting thread to process the message
		GCrashReportingThread->WaitUntilCrashIsHandled(); // 크래시 리포팅 스레드가 크래시의 리포팅을 완료하기를 기다립니다.
	}
#endif

	return EXCEPTION_EXECUTE_HANDLER;
}
```
NOINITCRASHREPORTER 매크로는 CrashReportClient 바이너리에 한해 1로 정의되어 있습니다.          
밑에서 설명하겠지만 언리얼 엔진은 크래시가 발생한 프로세스(플레이 중인 게임)에서 크래시를 리포팅하는 것이 아닌, 별도 CrashReportClient 프로세스를 생성해서 해당 프로세스의 크래시 관련 정보를 넘겨서 CrashReportClient 프로세스가 크래시 리포팅을 처리하는 방식으로 구현되어 있습니다.         
이유는 추측컨데 크래시가 발생한 프로세스(플레이 중인 게임)은 크래시가 발생하였다보니 상태가 메롱할 것입니다. OOM(Out of memory)로 인한 크래시가 발생했을 수도 있고 여러 알 수 없는 크래시로 프로세스의 상태가 안 좋을텐데 이 프로세스로 리포팅을 처리하다가는 리포팅 중 프로세스가 죽거나, 리포팅을 제대로 처리하지 못할 가능성이 있기 떄문에 별도 프로세스를 생성해서 해당 프로세스가 크래시를 리포팅하도록 하는 것이라고 생각합니다.          
밑에서 더 자세히 설명하겠지만 에디터 환경에서는 에디터 실행 시점부터 CrashReportClient 프로세스를 생성해서 에디터가 죽었는지, 행이 발생했는지 등등 여러 문제들을 검사하구요, 에디터가 아닌 경우(ex. 패키징된 클라이언트)에는 크래시 발생시 CrashReportClient 프로세스를 생성해서 해당 프로세스에 크래시 관련된 정보를 넘겨줍니다.        
              
ReportCrash에는 GCrashReportingThread라는 CrashReporting 스레드라는 개념이 등장합니다.             
이 스레드의 동작을 간단히 설명하면 엔진 초기화 단계에서 생성되어 크래시 발생시 셋팅되는 Event의 시그널을 대기하다 시그널이 오면 크래시 정보를 수집하고 CrashReportClient 프로세스를 생성하여 크래시 관련 정보를 넘겨주는 역할을 합니다.           
크래시 발생시 크래시가 발생한 스레드에서 이 Event에 Signal을 보냅니다.        
            
ReportCrashCallCount가 0이었는지 체크하는 이유는 여러 스레드에서 동시에 크래시가 발생할 수도 있기 때문에 하나의 크래시만을 처리하기 위함입니다.       
```c++
// 크래시가 발생한 스레드에서 호출
/** The thread that crashed calls this function which will trigger the CR thread to report the crash */
FORCEINLINE void FCrashReportingThread::OnCrashed(LPEXCEPTION_POINTERS InExceptionInfo)
{
	ExceptionInfo = InExceptionInfo;
	CrashingThreadId = GetCurrentThreadId(); // 크래시가 발생한 스레드(Caller 스레드)의 Thread ID를 셋팅해줍니다.
	CrashingThreadHandle = GetCurrentThread();
	SetEvent(CrashEvent); // CrashReporting 스레드가 시그널을 오기를 기다리고 있는 CrashEvent 이벤트입니다.
}

// CrashReporting 스레드의 Entry 함수
/** Main loop that waits for a crash to trigger the report generation */
FORCENOINLINE uint32 FCrashReportingThread::Run()
{
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
	__try
#endif
	{
		while (StopTaskCounter.GetValue() == 0)
		{
			if (WaitForSingleObject(CrashEvent, 500) == WAIT_OBJECT_0) // CrashEvent 시그널을 기다립니다.
			{
				ResetEvent(CrashHandledEvent);
				HandleCrashInternal();

				ResetEvent(CrashEvent);
				// Let the thread that crashed know we're done.
				SetEvent(CrashHandledEvent);

				break;
			}
			
			// CrashClientHandle는 CrashReportClient 프로세스의 Handle입니다.
			// CrashReportClient 프로세스가 이미 동작하고 있는 경우 이 분기를 탑니다.
			if (CrashClientHandle.IsValid() && !FPlatformProcess::IsProcRunning(CrashClientHandle))
			{
				// The crash monitor (CrashReportClient) died unexpectedly. Collect the exit code for analytic purpose.
				int32 CrashMonitorExitCode = 0;
				if (FPlatformProcess::GetProcReturnCode(CrashClientHandle, &CrashMonitorExitCode))
				{
					FGenericCrashContext::SetOutOfProcessCrashReporterExitCode(CrashMonitorExitCode);
					FPlatformProcess::CloseProc(CrashClientHandle);
					CrashClientHandle.Reset();
				}
			}
		}
	}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
	__except(EXCEPTION_EXECUTE_HANDLER) // CrashReporting 스레드가 리포팅 도중에 크래시가 발생할 수도 있기 때문에 이에 대응합니다.
	{
		// The crash reporting thread crashed itself. Exit with a code that the out-of-process monitor will be able to pick up and report into analytics.
		::exit(ECrashExitCodes::CrashReporterCrashed);
	}
#endif
	return 0;
}
```

이제 FCrashReportingThread::HandleCrashInternal 함수에서 크래시에 대한 정보를 수집하고 리포팅할 단계입니다.
```c++
/** Handles the crash */
FORCENOINLINE void FCrashReportingThread::HandleCrashInternal()
{
	// Stop the heartbeat thread so that it doesn't interfere with crashreporting
	FThreadHeartBeat::Get().Stop();

	// Then try run time crash processing and broadcast information about a crash.
	FCoreDelegates::OnHandleSystemError.Broadcast();

	if (GLog)
	{
		//Panic flush the logs to make sure there are no entries queued. This is
		//not thread safe so it will skip for example editor log.
		GLog->PanicFlushThreadedLogs();
	}
	
	// Get the default settings for the crash context
	ECrashContextType Type = ECrashContextType::Crash;
	const TCHAR* ErrorMessage = TEXT("Unhandled exception");
	TCHAR ErrorMessageLocal[UE_ARRAY_COUNT(GErrorExceptionDescription)];
	int NumStackFramesToIgnore = 2;

	void* ContextWrapper = nullptr;

	// If it was an assert or GPU crash, allow overriding the info from the exception parameters
	if (ExceptionInfo->ExceptionRecord->ExceptionCode == AssertExceptionCode && ExceptionInfo->ExceptionRecord->NumberParameters == 1)
	{
		const FAssertInfo& Info = *(const FAssertInfo*)ExceptionInfo->ExceptionRecord->ExceptionInformation[0];
		Type = ECrashContextType::Assert;
		ErrorMessage = Info.ErrorMessage;
		NumStackFramesToIgnore += Info.NumStackFramesToIgnore;
	}
	else if (ExceptionInfo->ExceptionRecord->ExceptionCode == GPUCrashExceptionCode && ExceptionInfo->ExceptionRecord->NumberParameters == 1)
	{
		const FAssertInfo& Info = *(const FAssertInfo*)ExceptionInfo->ExceptionRecord->ExceptionInformation[0];
		Type = ECrashContextType::GPUCrash;
		ErrorMessage = Info.ErrorMessage;
		NumStackFramesToIgnore += Info.NumStackFramesToIgnore;
	}
	// Generic exception description is stored in GErrorExceptionDescription
	else if (ExceptionInfo->ExceptionRecord->ExceptionCode != EnsureExceptionCode)
	{
		// When a generic exception is thrown, it is important to get all the stack frames
		NumStackFramesToIgnore = 0;
		CreateExceptionInfoString(ExceptionInfo->ExceptionRecord, ErrorMessageLocal, UE_ARRAY_COUNT(ErrorMessageLocal));
		ErrorMessage = ErrorMessageLocal;

		// TODO: Fix race conditions when writing GErrorExceptionDescription (concurrent threads can read/write it)
		FCString::Strncpy(GErrorExceptionDescription, ErrorMessageLocal, UE_ARRAY_COUNT(GErrorExceptionDescription));
	}

#if USE_CRASH_REPORTER_MONITOR
	if (CrashClientHandle.IsValid() && FPlatformProcess::IsProcRunning(CrashClientHandle))
	{
		// If possible use the crash monitor helper class to report the error. This will do most of the analysis
		// in the crash reporter client process.
		ReportCrashForMonitor(
			ExceptionInfo,
			Type,
			ErrorMessage,
			NumStackFramesToIgnore,
			CrashingThreadHandle,
			CrashingThreadId,
			CrashClientHandle,
			&SharedContext,
			CrashMonitorWritePipe,
			CrashMonitorReadPipe,
			EErrorReportUI::ShowDialog
		);
	}
	else
#endif
	{
		// Not super safe due to dynamic memory allocations, but at least enables new functionality.
		// Introduces a new runtime crash context. Will replace all Windows related crash reporting.
		FWindowsPlatformCrashContext CrashContext(Type, ErrorMessage);

		// Thread context wrapper for stack operations
		ContextWrapper = FWindowsPlatformStackWalk::MakeThreadContextWrapper(ExceptionInfo->ContextRecord, CrashingThreadHandle);
		CrashContext.SetCrashedProcess(FProcHandle(::GetCurrentProcess()));
		CrashContext.CapturePortableCallStack(NumStackFramesToIgnore, ContextWrapper);
		CrashContext.SetCrashedThreadId(CrashingThreadId);
		CrashContext.CaptureAllThreadContexts();

		// Also mark the same number of frames to be ignored if we symbolicate from the minidump
		CrashContext.SetNumMinidumpFramesToIgnore(NumStackFramesToIgnore);

		// First launch the crash reporter client.
#if WINVER > 0x502	// Windows Error Reporting is not supported on Windows XP
		if (GUseCrashReportClient)
		{
			ReportCrashUsingCrashReportClient(CrashContext, ExceptionInfo, EErrorReportUI::ShowDialog);
		}
		else
#endif		// WINVER
		{
			CrashContext.SerializeContentToBuffer();
			WriteMinidump(GetCurrentProcess(), GetCurrentThreadId(), CrashContext, MiniDumpFilenameW, ExceptionInfo);
		}
	}

	const bool bGenerateRuntimeCallstack =
#if UE_LOG_CRASH_CALLSTACK
		true;
#else
		FParse::Param(FCommandLine::Get(), TEXT("ForceLogCallstacks")) || FEngineBuildSettings::IsInternalBuild() || FEngineBuildSettings::IsPerforceBuild() || FEngineBuildSettings::IsSourceDistribution();
#endif // UE_LOG_CRASH_CALLSTACK
	if (bGenerateRuntimeCallstack)
	{
		const SIZE_T StackTraceSize = 65535;
		ANSICHAR* StackTrace = (ANSICHAR*)GMalloc->Malloc(StackTraceSize);
		StackTrace[0] = 0;
		// Walk the stack and dump it to the allocated memory. This process usually allocates a lot of memory.
		if (!ContextWrapper)
		{
			ContextWrapper = FWindowsPlatformStackWalk::MakeThreadContextWrapper(ExceptionInfo->ContextRecord, CrashingThreadHandle);
		}
		
		FPlatformStackWalk::StackWalkAndDump(StackTrace, StackTraceSize, 0, ContextWrapper);
		
		if (ExceptionInfo->ExceptionRecord->ExceptionCode != EnsureExceptionCode && ExceptionInfo->ExceptionRecord->ExceptionCode != AssertExceptionCode)
		{
			CreateExceptionInfoString(ExceptionInfo->ExceptionRecord, GErrorExceptionDescription, UE_ARRAY_COUNT(GErrorExceptionDescription));
			FCString::Strncat(GErrorHist, GErrorExceptionDescription, UE_ARRAY_COUNT(GErrorHist));
			FCString::Strncat(GErrorHist, TEXT("\r\n\r\n"), UE_ARRAY_COUNT(GErrorHist));
		}

		FCString::Strncat(GErrorHist, ANSI_TO_TCHAR(StackTrace), UE_ARRAY_COUNT(GErrorHist));

		GMalloc->Free(StackTrace);
	}

	// Make sure any thread context wrapper is released
	if (ContextWrapper)
	{
		FWindowsPlatformStackWalk::ReleaseThreadContextWrapper(ContextWrapper);
	}

#if !UE_BUILD_SHIPPING
	FPlatformStackWalk::UploadLocalSymbols();
#endif
	}
};
```


아래 전부 번역해야함.
```c++
/**
 * Fallback for handling exceptions that aren't handled elsewhere.
 *
 * The SEH mechanism is not very well documented, so to start with, few facts to know:
 *   - SEH uses 'handlers' and 'filters'. They have different roles and are invoked at different state.
 *   - Any unhandled exception is going to terminate the program whether it is a benign exception or a fatal one.
 *   - Vectored exception handlers, Vectored continue handlers and the unhandled exception filter are global to the process.
 *   - Exceptions occurring in a thread doesn't automatically halt other threads. Exception handling executes in thread where the exception fired. The other threads continue to run.
 *   - Several threads can crash concurrently­.
 *   - Not all exceptions are equal. Some exceptions can be handled doing nothing more than catching them and telling the code to continue (like some user defined exception), some
 *     needs to be handled in a __except() clause to allow the program to continue (like access violation) and others are fatal and can only be reported but not continued (like stack overflow).
 *   - Not all machines are equal. Different exceptions may be fired on different machines for the same usage of the program. This seems especially true when
 *     using the OS 'open file' dialog where the user specific extensions to the Windows Explorer get loaded in the process.
 *   - If an exception handler/filter triggers another exception, the new inner exception is handled recursively. If the code is not robust, it may retrigger that inner exception over and over.
 *     This eventually stops with a stack overflow, at which point the OS terminates the program and the original exception is lost.
 *
 * Usually, when an exception occurs, Windows executes following steps (see below for unusual cases):
 *     1- Invoke the vectored exception handlers registered with AddVectoredExceptionHandler(), if any.
 *         - In general, this is too soon to handle an exception because local structured exception handlers did not execute yet and many exceptions are handled there.
 *         - If a registered vectored exception handler returns EXCEPTION_CONTINUE_EXECUTION, the vectored continue handler(s), are invoked next (see number 4 below)
 *         - If a registered vectored exception handler returns EXCEPTION_CONTINUE_SEARCH, the OS skip this one and continue iterating the list of vectored exception handlers.
 *         - If a registered vectored exception handler returns EXCEPTION_EXECUTE_HANDLER, in my tests, this was equivalent to returning EXCEPTION_CONTINUE_SEARCH.
 *         - If no vectored exception handlers are registered or all registered one return EXCEPTION_CONTINUE_SEARCH, the structured exception handlers (__try/__except) are executed next.
 *         - At this stage, be careful when returning EXCEPTION_CONTINUE_EXECUTION. For example, continuing after an access violation would retrigger the exception immediatedly.
 *     2- If the exception wasn't handled by a vectored exception handler, invoke the structured exception handlers (the __try/__except clauses)
 *         - That let the code manage exceptions more locally, for the Engine, we want that to run first.
 *         - When the filter expression in __except(filterExpression) { block } clause returns EXCEPTION_EXECUTE_HANDLER, the 'block' is executed, the code continue after the block. The exception is considered handled.
 *         - When the filter expression in __except(filterExpression) { block } clause returns EXCEPTION_CONTINUE_EXECUTION, the 'block' is not executed and vectored continue exceptions handlers (if any) gets called. (see number 4 below)
 *         - When the filter expression in __except(filterExpression) { block } clause returns EXCEPTION_CONTINUE_SEARCH, the 'block' is not executed and the search continue for the next __try/__except in the callstack.
 *         - If all unhandled exception filters within the call stack were executed and all of them returned returned EXCEPTION_CONTINUE_SEARCH, the unhandled exception filter is invoked. (see number 3 below)
 *         - The __except { block } allows the code to continue from most exceptions, even from an access violation because code resume after the except block, not at the point of the exception.
 *     3- If the exception wasn't handled yet, the system calls the function registered with SetUnhandedExceptionFilter(). There is only one such function, the last to register override the previous one.
 *         - At that point, both vectored exception handlers and structured exception handlers have had a chance to handle the exception but did not.
 *         - If this function returns EXCEPTION_CONTINUE_SEARCH or EXCEPTION_EXECUTE_HANDLER, by default, the OS handler is invoked and the program is terminated.
 *         - If this function returns EXCEPTION_CONTINUE_EXECUTION, the vectored continue handlers are invoked (see number 4 below)
 *     4- If a handler or a filter returned the EXCEPTION_CONTINUE_EXECUTION, the registered vectored continue handlers are invoked.
 *         - This is last chance to do something about an exception. The program was allowed to continue by a previous filter/handler, effectively ignoring the exception.
 *         - The handler can return EXCEPTION_CONTINUE_SEARCH to observe only. The OS will continue and invoke the next handler in the list.
 *         - The handler can short cut other continue handlers by returning EXCEPTION_CONTINUE_EXECUTION which resume the code immediatedly.
 *         - In my tests, if a vectored continue handler returns EXCEPTION_EXECUTE_HANDLER, this is equivalent to returning EXCEPTION_CONTINUE_SEARCH.
 *         - By default, if no handlers are registered or all registered handler(s) returned EXCEPTION_CONTINUE_SEARCH, the program resumes execution at the point of the exception.
 *
 * Inside a Windows OS callback, in a 64-bit application, a different flow than the one described is used.
 *    - 64-bit applications don't cross Kernel/user-mode easily. If the engine crash during a Kernel callback, EngineUnhandledExceptionFilter() is called directly. This behavior is
 *      documented by various article on the net. See: https://stackoverflow.com/questions/11376795/why-cant-64-bit-windows-unwind-user-kernel-user-exceptions.
 *    - On early versions of Windows 7, the kernel could swallow exceptions occurring in kernel callback just as if they never occurred. This is not the case anymore with Win 10.
 *
 * Other SEH particularities:
 *     - A stack buffer overflow bypasses SEH entirely and the application exits with code: -1073740791 (STATUS_STACK_BUFFER_OVERRUN).
 *     - A stack overflow exception occurs when not enough space remains to push what needs to be pushed, but it doesn't means it has no stack space left at all. The exception will be reported
 *       if enough stack space is available to call/run SEH, otherwise, the app exits with code: -1073741571 (STATUS_STACK_OVERFLOW)
 *     - Fast fail exceptions bypasse SEH entirely and the application exits with code: -1073740286 (STATUS_FAIL_FAST_EXCEPTION) or 1653 (ERROR_FAIL_FAST_EXCEPTION)
 *     - Heap corruption (like a double free) is a special exception. It is likely only visible to Vectored Exception Handler (VEH) before possibly beeing handled by Windows Error Reporting (WER).
 *       A popup may be shown asking to debug or exit. The application may exit with code -1073740940 (STATUS_HEAP_CORRUPTION) or 255 (Abort) depending on the situation.
 *
 * The engine hooks itself in the unhandled exception filter. This is the best place to be as it runs after structured exception handlers and
 * it can be easily overriden externally (because there can only be one) to do something else.
 */
LONG WINAPI EngineUnhandledExceptionFilter(LPEXCEPTION_POINTERS ExceptionInfo)
```


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

references : [https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically](https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically), [https://stackoverflow.com/a/16785259](https://stackoverflow.com/a/16785259), [https://stackoverflow.com/a/4574319](https://stackoverflow.com/a/4574319), [https://stackoverflow.com/a/7249442](https://stackoverflow.com/a/7249442), [https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170](https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170), [https://kuaaan.tistory.com/435](https://kuaaan.tistory.com/435), [https://stackoverflow.com/a/7049836](https://stackoverflow.com/a/7049836)