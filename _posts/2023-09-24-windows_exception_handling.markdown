---
layout: post
title:  "Windows Exception Handling(작성 중)"
date:   2023-09-24
tags: [ComputerScience, UE]
---          
             

언리얼 엔진4 구현상 기본적으로 bEnableExceptions 기본 값은 false이다(대부분의 모듈에서 false)     
```c++
// Enable C++ exceptions when building with the editor or when building UHT.
if (CompileEnvironment.bEnableExceptions)
{
	// Enable C++ exception handling, but not C exceptions.
	Arguments.Add("/EHsc");
}
else
{
	// This is required to disable exception handling in VC platform headers.
	Arguments.Add("/D_HAS_EXCEPTIONS=0");
}
```
EHsc :
a
Enables standard C++ stack unwinding. Catches both structured (asynchronous) and standard C++ (synchronous) exceptions when you use catch(...) syntax. /EHa overrides both /EHs and /EHc arguments.

s(!!)
Enables standard C++ stack unwinding. Catches only standard C++ exceptions when you use catch(...) syntax. Unless /EHc is also specified, the compiler assumes that functions declared as extern "C" may throw a C++ exception.

c(!!)
When used with /EHs, the compiler assumes that functions declared as extern "C" never throw a C++ exception. It has no effect when used with /EHa (that is, /EHca is equivalent to /EHa). /EHc is ignored if /EHs or /EHa aren't specified.


--------------------

standard C++ exceptions : std::throw~
automatically clean up local objects

vs

SEH=structured (asynchronous) exception(that are raised by the hardware(hardware faults). Things like floating point exceptions, division by zero, and the all-mighty access violation exception.)
```
Specifying /EHa and trying to handle all exceptions by using catch(...) can be dangerous. In most cases, asynchronous exceptions are unrecoverable and should be considered fatal. Catching them and proceeding can cause process corruption and lead to bugs that are hard to find and fix.

Even though Windows and Visual C++ support SEH, we strongly recommend that you use ISO-standard C++ exception handling (/EHsc or /EHs). It makes your code more portable and flexible. There may still be times you have to use SEH in legacy code or for particular kinds of programs. It's required in code compiled to support the common language runtime (/clr), for example. For more information, see Structured exception handling.
```

------------------------

 To implement SEH without specifying /EHa, you may use the __try, __except, and __finally syntax
 https://learn.microsoft.com/en-us/cpp/cpp/structured-exception-handling-c-cpp?view=msvc-170
 __try, __except를 이용해서 SEH 구현 가능
 
```
If you use SEH in a C++ program that you compile by using the /EHa or /EHsc option, destructors for local objects are called, but other execution behavior might not be what you expect. For an illustration, see the example later in this article. In most cases, instead of SEH we recommend that you use ISO-standard C++ exception handling. By using C++ exception handling, you can ensure that your code is more portable, and you can handle exceptions of any type.
```

EngineUnhandledExceptionFilter의 주석 자세히 읽어보면 좋을 듯

[SetUnhandledExceptionFilter](https://learn.microsoft.com/ko-kr/windows/win32/api/errhandlingapi/nf-errhandlingapi-setunhandledexceptionfilter)         

---------------------
UE4 Crash Report
SetUnhandledExceptionFilter로 UnhandledException Handling Function(함수명 : EngineUnhandledExceptionFilter) 등록

FCrashReportingThread 생성해서 CrashEvent 기다림
Exception 발생시 해당 스레드가 UnhandledException Handling Function 실행 -> UnhandledException Handling Function에서 CrashEvent 등록 후 FCrashReportingThread가 Report를 완료하기를 대기함
CrashEvent 등록되면 FCrashReportingThread는 그 이벤트를 받아서 Crash Report 수행(혹시나 Crash Report 동작 코드에서 크래시가 날 수 있으니 해당 코드들은 SEH로 감싸져있음)
 
references : [https://stackoverflow.com/questions/4573536/msvc-ehsc-vs-eha-synchronous-vs-asynchronous-exception-handling](https://stackoverflow.com/questions/4573536/msvc-ehsc-vs-eha-synchronous-vs-asynchronous-exception-handling), [https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically](https://learn.microsoft.com/en-us/cpp/build/reference/eh-exception-handling-model?view=msvc-170#set-the-option-in-visual-studio-or-programmatically)