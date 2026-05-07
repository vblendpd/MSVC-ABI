###### CONTENT
This article introduces the underlying structure of the 64-bit Application Binary Interface (ABI) for the C++ language in the Windows environment. Specifically, it provides a low-level analysis of:
```
Stack
Built-In Types
Data Alignment and Packing
64-bit Calling Conventions
One Definition Rule (ODR)
Name Mangling
Objects Memory Layout
Run-Time Type Identification (RTTI)
C Run-Time Libraries (CRT)
Exception Handling
```
Each chapter will be made available as a separate markdown file:

[Stack](Chapter/Stack.md)\
[Built-in Types](Chapter/Types.md)\
[Data Alignment and Packing](Chapter/Alignment.md)\
...\
...

The material presented here is specific to Microsoft Windows using the Visual Studio C++ compiler and the x86 architecture. The content focuses on 64-bit binaries, highlighting differences compared to their 32-bit counterparts whenever possible. This article assumes familiarity with the Visual Studio environment and a basic understanding of Assembly language and hardware architecture.

###### PREFACE
Each Assembly code fragment is taken from a 64-bit Release build with optimization and stack security checks disabled. The reason for disabling optimization in a Release build is purely for educational purposes. This is not a recommended practice in production code, as it prevents the compiler from performing compile-time calculations and other optimizations. Although most of the upcoming examples may seem complex from an Assembly standpoint, they are actually quite simple. In practice, the compiler might entirely remove a function call. For now, we want to analyze step by step how parameters are propagated to functions and other details. Optimization will be enabled whenever it is not relevant to the current topic. To generate machine code similar to what is shown in this article, you need to disable `Optimization` and `Inline Function Expansion` in the project settings:
```
Properties > C/C++ > Optimization
```
Both can also be disabled from the command line using `/Od` and `/Ob0`. You can explicitly instruct the compiler not to inline a function using the Microsoft-specific specifier:
```c++
__declspec(noinline) int GetValue(int x)
{
  x++;
  return x * 2 + 1;
}
```
The inline mechanism is relative to the calling point. Since the `GetValue` call will not be replaced with its instructions, a parent function might become smaller and could itself be marked as inline by the compiler. Therefore, it is better to disable inlining globally from the project settings.

The compiler may add extra code for stack security checks (buffer overruns) to detect overflow attacks. This code writes a value called a security cookie to the stack when a function frame is allocated and verifies that the value has not changed before returning. A different value indicates that the return address may have been overwritten, potentially by malicious code. This topic is beyond the scope of this article. Instead of ignoring the extra Assembly code, we can prevent the compiler from generating it. You can disable the `Security Check` option here:
```
Properties > C/C++ > Code Generation
```
The command line option to disable the security check is `/GS-`.

Most of the Visual Studio windows used here are only visible during debugging. To open any of them, you must hit a breakpoint first. Then, you can access debugging windows from the `Debug > Windows` menu. Default shortcuts:
```
Registers   (Alt+5)
Memory      (Alt+6)
Call Stack  (Alt+7)
Disassembly (Alt+8)
```
At the beginning, I recommend disabling source code and symbol names in the Disassembly view by default. Right-click in the Disassembly window and disable `Show Source Code` and `Show Symbol Names.` By doing so, the Disassembly output will change from the first to the second example:
```c++
int Compute(int a, int b)
{
00007FF6C3F31000  mov dword ptr [rsp+10h],edx
00007FF6C3F31004  mov dword ptr [rsp+8],ecx
00007FF6C3F31008  sub rsp,18h
    int t = a * 2;
00007FF6C3F3100C  mov eax,dword ptr [a]
00007FF6C3F31010  shl eax,1
00007FF6C3F31012  mov dword ptr [rsp],eax
    return t + b;
00007FF6C3F31015  mov eax,dword ptr [b]
00007FF6C3F31019  mov ecx,dword ptr [rsp]
00007FF6C3F3101C  add ecx,eax
00007FF6C3F3101E  mov eax,ecx

}
00007FF6C3F31020  add rsp,18h
00007FF6C3F31024  ret
```
```c++
00007FF6C3F31000  mov dword ptr [rsp+10h],edx
00007FF6C3F31004  mov dword ptr [rsp+8],ecx
00007FF6C3F31008  sub rsp,18h
00007FF6C3F3100C  mov eax,dword ptr [rsp+20h]
00007FF6C3F31010  shl eax,1
00007FF6C3F31012  mov dword ptr [rsp],eax
00007FF6C3F31015  mov eax,dword ptr [rsp+28h]
00007FF6C3F31019  mov ecx,dword ptr [rsp]
00007FF6C3F3101C  add ecx,eax
00007FF6C3F3101E  mov eax,ecx
00007FF6C3F31020  add rsp,18h
00007FF6C3F31024  ret
```
While it becomes more difficult to determine what the CPU is reading from memory, this approach provides a better understanding of how variables are stored on the stack. Data is referenced by adding an offset from a base address rather than using its name. Similarly, hiding the corresponding C++ statements makes the Assembly code harder to read, but it helps you practice recognizing Assembly patterns.

If the memory block of interest is outside the range displayed in the Memory or Disassembly view, you can copy the address from the appropriate register (in the Register Window) and paste it into the `Address` textbox located at the top of the view. You can also type a register name in the textbox to scroll the view to the desired address.
