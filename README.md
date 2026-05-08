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
[64-bit Calling Conventions](Chapter/callingc.md)\
[Name Mangling](Chapter/nmangling.md)\
...\
...

The material presented here is specific to Microsoft Windows using the Visual Studio C++ compiler and x86 architecture. The content focuses on 64-bit binaries, noting differences from their 32-bit counterparts wherever applicable. This article assumes you are familiar with the Visual Studio environment and have a working knowledge of Assembly language and hardware architecture.

###### PREFACE
Each Assembly code fragment is taken from a 64-bit Release build with optimization and stack security checks disabled. Disabling optimization for a Release build is done solely for educational purposes and is not a good practice in production code, as it prevents the compiler from performing compile-time calculations and numerous other optimizations. Although most of the upcoming examples may seem quite complex from an Assembly standpoint, they are in fact quite simple. Eventually, the compiler will remove a function call entirely. We do not want that for now, because we need to analyze step by step how parameters are propagated to functions and other details. Optimization will be re-enabled whenever it is not relevant to the current topic. For now, in order to generate similar machine code to what you will see throughout the article, you need to turn off `Optimization` and `Inline Function Expansion` from the project settings:
```
Properties > C/C++ > Optimization
```
Both can be disabled from the command line using `/Od` and `/Ob0`. You can also explicitly tell the compiler not to inline a function using the Microsoft-specific `__declspec` specifier:
```C++
__declspec(noinline) int GetValue(int x)
{
  x++;
  return x*2 + 1;
}
```
The inlining decision is made relative to the call site. Since the `GetValue` call will not be replaced with its instructions, the calling function may become smaller and can itself be marked as inline by the compiler. It is therefore preferable to disable this globally from the project settings.

The compiler may add extra code for stack security checks (against buffer overruns) to detect overflow attacks. It writes a value known as the security cookie onto the stack when a function frame is allocated, and checks whether that value has not changed prior to return. A different value indicates that the return address may have been overwritten, perhaps by malicious code. This topic is outside the scope of this article, and rather than ignoring the extra Assembly code, we can prevent the compiler from generating it. You can find the `Security Check` option to turn it off here:
```
Properties > C/C++ > Code Generation
```
The command line option to disable the security check is `/GS-`.

Most of the Visual Studio windows you will use here are only visible during debugging. To open any of them, you must first hit a breakpoint. Then you can open debugging windows from the `Debug > Windows` menu. Default shortcuts:
```
Registers   (Alt+5)
Memory      (Alt+6)
Call Stack  (Alt+7)
Disassembly (Alt+8)
```
I recommend initially disabling the source code and symbol names displayed by default in the Disassembly view. Right-click on the Disassembly window and disable `Show Source Code` and `Show Symbol Names`. By doing so, the Disassembly output will change from the first to the second form:
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
It becomes more difficult to determine what the CPU is reading from memory, but on the other hand, you gain a better understanding of how variables are laid out on the stack. Data is referenced by offset from a base address rather than by name. The same applies to the source code: hiding the corresponding C++ statements makes Assembly code harder to read, but at the same time, it helps you practice recognizing Assembly patterns.

If the memory block of interest is outside the range displayed in the Memory or Disassembly view, you can copy the address from the appropriate register (from the Registers window) and paste it into the `Address` textbox located at the top of the view. You can also type a register name into the textbox to scroll the view to the desired address.
