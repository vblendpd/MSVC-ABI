###### CONTENT
This article introduces the underlying structure of the 64-bit Application Binary Interface (ABI) for the C++ language in the Windows environment. In particular, it provides a low-level analysis of:
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

The material presented here is specific for Microsoft Windows using Visual Studio C++ compiler and x86 architecture. The content focuses on 64-bit binaries reporting differences against 32-bit counterparts whenever possible. This article assumes you're familiar with the Visual Studio environment and have some command of Assembly language and hardware architecture.

###### PREFACE
Each Assembly code fragment is taken from a 64-bit Release build with optimization and stack security check disabled. The reason to turn off optimization for Release build is solely for educational purposes, not a good practice in production code because it prevents the compiler from performing compile-time calculations and dozens of other things. Although most of the upcoming examples seem quite complex from an Assembly standpoint, they're quite simple instead. Eventually, the compiler will remove a function call entirely. We don't want that for now because we need to analyze step by step how parameters are propagated to functions and other details. We'll turn on optimization whenever they won't be relevant for the current topic. For now, in order to generate the similar machine code you'll see through the article, you need to turn off `Optimization` and `Inline Function Expansion` from the project settings:
```
Properties > C/C++ > Optimization
```
Both can be disabled from the command line using `/Od` and `/Ob0`. You can also explicitly tell the compiler not to inline a function using Microsoft specifier:
```C++
__declspec(noinline) int GetValue(int x)
{
  x++;
  return x*2 + 1;
}
```
The inline mechanism is relative to the calling point. Since the `GetValue` call won't be replaced with its instructions, a parent function might get smaller and can itself be marked inline by the compiler. Hence, better to disable globally from project settings.

The compiler may add extra code for stack security checks (buffer overruns) to detect overflow attacks. The code writes a value called security cookie on the stack when a function frame is allocated, and checks if the same value has not changed prior to return. A different value means that the return address may have been overwritten, perhaps by malicious code. This topic is out of the scope of this article, and instead of ignoring extra Assembly code, we can prevent the compiler from generating it. You can find the option `Security Check` to turn it off here:
```
Properties > C/C++ > Code Generation
```
The command line option to disable the security check is `/GS-`.

Most of the Visual Studio windows you will use here are only visible during debugging. To open any of them, you must hit a breakpoint first. Then you can open debugging windows from the `Debug > Windows` menu. Default shortcuts:
```
Registers   (Alt+5)
Memory      (Alt+6)
Call Stack  (Alt+7)
Disassembly (Alt+8)
```
I recommend at the beginning to disable source code and symbol names used in the Disassembly view by default. Right-click over the Disassembly window and disable `Show Source Code` and `Show Symbol Names.` By doing so, the Disassembly output will change from the first to the second one:
```
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
```
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
It's more difficult to figure out what the CPU is reading from memory, but on the other hand, you get a better grasp of how variables are stored around the stack. Data is referenced by adding an offset from a base address and not using its name. The same goes for the source code: hiding the corresponding C++ statements makes Assembly code harder to read, but at the same time, you practice recognizing Assembly patterns.

In case the interested memory block is outside the range displayed in Memory or Disassembly view, you can copy the address from the appropriate register (from Register Window) and paste it to the `Address` textbox located at the top of the view. You can also type in a register name in the textbox to scroll the view to the desired address.
