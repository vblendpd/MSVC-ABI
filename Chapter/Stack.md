> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### THE RUN-TIME STACK
The run-time stack is one of the fundamental pillars of program execution. Understanding how the stack operates at a low-level is optional for writing software. The compiler translates high-level code to machine instructions that manipulate the run-time stack. However, most of the upcoming topics are heavily based on it, so it is worth a look.

Every time a program starts execution, the system creates a system object representing the program running instance: a process. A process contains information about the virtual address space assigned to the process, the main executable, other modules, threads of execution, and more. A portion of the virtual address space is reserved for threads stack. To support code execution, each thread in the process has at least one stack, but that's seldom the case in practice. Some operations (e.g., creating files) are kernel jobs and are not allowed in user space. In such cases, transitioning from user space to kernel space requires the thread to have two stacks to support execution in both levels.\
All threads in a process share the same address space. It means they're free to write incorrectly to memory locations that don't belong to them. For that reason, on thread creation, the system ensures that each thread will have its contiguous portion of virtual memory to use as a stack that won't overlap with others. We'll only analyze the main user-mode thread (the main thread is the one that will eventually invoke the main entry point).
> Windows Internals book is the best reference to familiarize yourself with system details.

The low-level run-time stack differs from the homonym Last-In-First-Out (LIFO) data structure. The CPU indeed provides support to manipulate it in a stack-like fashion using `push` and `pop` instructions. Still, at the same time, the CPU is free to access other addresses in the stack, not limited to stack-top or resize it without the aid of those instructions, with some limitations we'll see shortly.

###### THE FRAME
Every function takes a portion of the stack to store local variables and more. The amount of the stack reserved for such purpose is called `stack frame`, otherwise known as `activation record`. Caller and callee functions (I will sometime name them parent and child functions) work together to construct and release the stack frame. Consider the following function call:
```cpp
int Func(int x, int y, int f)
{
  int t = x*y
  return t+f;
}

int x = Func(5,6,2);
```
The stack frame for 32-bit and 64-bit builds might look like this:
```
    32-bit:                        64-bit:
    0137F7A8  0137f7c8             000000B71D39F888  00000000
    0137F7AC  002710fe             000000B71D39F88C  9c002400
ESP 0137F7B0  0027102f         RSP 000000B71D39F890  0fb421a8
    0137F7B4 |00000005| [x]        000000B71D39F894  00007ff7
    0137F7B8 |00000006| [y]        000000B71D39F898  0fb4148d
    0137F7BC |00000002| [f]        000000B71D39F89C  00007ff7
    0137F7C0  00000000             000000B71D39F8A0  0000001f
EBP 0137F7C4  0137f80c             000000B71D39F8A4  00000000
    0137F7C8  002711fa             000000B71D39F8A8  00000001
    0137F7CC  00000001             000000B71D39F8AC  00000000
```
In 32-bit code, some of the available calling conventions use the stack to pass parameters to functions, while others use a combination of stack and CPU registers. In a 64-bit build, depending on the number and type of parameters, some of these values can be propagated to functions using registers, and any remaining parameter will use the stack. That's why you can spot `x` `y` `f` in a 32-bit frame while they're missing for a 64-bit one (assuming the same parameters haven't been stored far from RSP for some reason).

A snapshot of the stack at a given time identifies the frames of currently executing functions that have not yet terminated, often called `call stack`. The high-level abstraction of these frames stacked on top of each other can be inspected during debugging using `Call Stack (Alt+7)` window:
```
ConsoleApplication1.exe!F3()
ConsoleApplication1.exe!DoWork(int y)
ConsoleApplication1.exe!Compute(int a, int b)
ConsoleApplication1.exe!main()
[External Code]
Kernel32.dll!
```
The stack usage is not limited to function parameters management. Non-volatile registers (registers whose content must be preserved across function calls), local variables, and return addresses are all stored in the stack. We'll look at more details in the calling conventions chapter.

The stack starts from a base address and grows toward a lower address. Some representations draw higher addresses going up; others draw the same going down. It's just a convention, and you're free to choose the one you feel comfortable working with. I will use the same as Visual Studio Memory view since most memory snapshots are taken from there.

###### FRAME POINTER
In 32-bit programs, base-pointer register EBP also called frame-pointer, points to the base of the current frame. At the same time, ESP always holds the address of the stack-top, which is the address of the last value added on top of the stack or the new address of the stack-top after a resize operation. The CPU accesses the data in the stack, adding an offset to the base address stored in EBP. In 64-bit programs, the corresponding base pointer and stack-top are RBP and RSP. RBP may not be used, and the system runs the code by manipulating the stack using RSP only. In a 32-bit optimized build, the frame-pointer might be omitted, and the data in the stack is referenced using an offset from ESP instead. The option `Omit Frame Pointers` to discard the use of base-pointer register can be found in:
```
Properties > C/C++ > Optimization
```
Note that frame-pointer omission is an optimization and doesn't take effect if optimization is disabled (the same we disabled at the beginning with `/Od`). In 64-bit programs, we rarely have this kind of control, and RBP is omitted by default, so the `Omit Frame Pointer` option is ignored. The stack alignment requirements in 64-bit programs eliminate the need for dynamic stack alignment and frame-pointer register. The omission of the frame-pointer has implications in stack walking and unwinding, but other solutions are possible without frame-pointer information. These are rare events. Hence a slower solution that doesn't employ a frame-pointer is an acceptable trade-off.

Without the burden of maintaining base-pointer information, a function will generally execute faster because no time is spent setting-up base-pointer during the prologue and epilogue. When EBP or RBP is not employed as a frame-pointer register, the same register is available for general-purpose use but, most of the time, not used at all. The base-pointer register falls into the non-volatile category, and the CPU must save its content across function calls. Thus, whenever possible, the compiler avoids additional `push` and `pop` (or memory accesses in general).

The compiler identifies cases where a frame-pointer cannot be omitted, either in 32-bit or 64-bit, and bring it back into action when required. A typical example is dynamic stack allocation, using the `alloca` function. In this case, the frame-pointer register comes back into play for a while and is abandoned again as soon as the function terminates. To exemplify how this works, we will consider a 64-bit example for static and dynamic stack allocation, specifically how they differ in allocating and releasing stack space. For this example, I enable optimization for smaller code generation. The first example allocates an array statically:
```cpp
int DoDomething(int a, int b)
{
  volatile int data[4];

  for(int i = 0; i < 4; i++)
    data[i] = i;

  return a+b;
}
```
```
RSP = 000000902087FDB0
RBP = 0000000000000000

00007FF6FBA21000  sub rsp,18h ;prologue
...
...
00007FF6FBA2101C  add rsp,18h ;epilogue
00007FF6FBA21020  ret
```
The content of RSP and RBP just before the call is indicated above. We ignore the details of function calls and other processing since they are not currently relevant.\
The first instruction is to resize the stack by 24 bytes (18h) to make space for local variables. RSP is left untouched in between prologue-epilogue except for addressing variables. The epilogue shrinks the stack by the same amount to clean local variables. RBP was not used all the time; its content is still 0. Now let's see what changes if we dynamically allocate space on the stack:
```cpp
int DoSomething(int a, int b)
{
  volatile int* data = (int*)alloca(16);

  for(int i = 0; i < 4; i++)
    data[i] = i;

  return a+b;
}
```
```
RSP = 000000ACC1D4FB80
RBP = 0000000000000000

00007FF7369F1000  push rbp     ;prologue
00007FF7369F1002  mov  rbp,rsp

;alloca(16)
00007FF7369F1005  mov  eax,dword ptr [rsp] ;alloca code
00007FF7369F1008  sub  rsp,10h             ;alloca code
...
...
00007FF7369F1026  mov  rsp,rbp ;epilogue
00007FF7369F1029  pop  rbp
00007FF7369F102A  ret
```
Both prologue and epilogue look different now. The function begins by saving RBP onto the stack. Then it copies the value from RSP to RBP, establishing a new base-pointer for what's coming next. Next is the inline implementation of alloca. There is no call to alloca since we realize the call has been replaced with instructions that resize the stack for 16 bytes (10h) as specified in C++ code. The epilogue will restore RSP (from EBP) and RBP (from the stack). RBP is now 0 again (specifically, the previous value).

The reason why we need RBP in the latter case is straightforward. Since the compiler cannot predict how much space will be allocated (the alloca parameter may be computed at run-time, hence the name `dynamic allocation`), it needs to save information about the current frame so the whatever-space can be released by moving RSP back to its previous position. Holding RSP onto the stack to restore it later will not work for the same reason: to pop the last value back into RSP, you first need to move RSP to the address where its previous value has been pushed. Calculating that offset at compile-time is impossible for the same reason because the amount of dynamically allocated space can't be known in advance. In addition to that, and more important, data in the stack is accessed by adding an offset from RSP so the same register cannot alter its content in between prologue-epilogue, or the data will not correspond to the intended one (instructions that read data at RSP+32, etc. are already encoded). RBP-relative addressing is used to avoid wizardry or inefficient solutions: offsets from RBP remain valid for the entire function lifetime, no matter what happens to RSP.

###### ENDIANNESS AND MEMORY DISPLAY
We're dealing with little-endian architecture, so the last significant byte stored at the corresponding memory address is the rightmost one. Suppose you're looking at some piece of memory:
```
000000B71D39F894 |00000002|
000000B71D39F898  00000000
000000B71D39F89C  00007ff7
000000B71D39F8A0  0000001f
```
The address-byte relationships for the highlighted integer are:
```
000000B71D39F894  02
000000B71D39F895  00
000000B71D39F896  00
000000B71D39F897  00
```
You can inspect the stack content in the Memory window. You can use its toolbar to change the number of columns for the displayed data. With right-click menu is possible to change other properties. This article uses:
```
1 Column
4-byte integer
Hexadecimal Display
No Text
```
With a 1-byte integer display and multiple columns, the least significant byte is drawn on the left. The previous memory block using a 1-byte integer and 4 columns will change to this:
```
000000B71D39F894  02 00 00 00
000000B71D39F898  00 00 00 00
000000B71D39F89C  f7 7f 00 00
000000B71D39F8A0  1f 00 00 00
```
The endianness display can be changed directly with right-click menu, turning on or off `Bid Endian`, but remember that's only for display purposes. Whenever you paste a memory address to `Address` textbox, it's expected to be a little-endian address.

###### SIZE AND ALLOCATION
In a previous paragraph, I mentioned the CPU's ability to resize the stack without using `push` and `pop` instructions. This resize mechanism is constrained to the available reserved space described below.

The default per-thread reserved stack space in user mode is set to 1 MB. The reserved space is the maximum amount of space the stack can consume during its lifetime. You can set a different amount in many ways, and it's limited to the available virtual address range.\
Using `/F N` specifying N in bytes to the command line in:
```
Properties > C/C++ > Command Line
```
A better alternative using linker options `Stack Reserve Size` and `Stack Commit Size` you can find here:
```
Properties > Linker > System
```
These values are embedded and inspectable in the binary file. Open `Visual Studio Command Prompt` from `Tools` menu, reach the directory where your executable is located, and run:
```
dumpbin /headers program.exe
```
The size of stack reserve and stack commit is listed in hexadecimal, along optional header values:
```
OPTIONAL HEADER VALUES
          ...
          ...
            8160 DLL characteristics
                   High Entropy Virtual Addresses
                   Dynamic base
                   NX compatible
                   Terminal Server Aware
          100000 size of stack reserve
            1000 size of stack commit
          100000 size of heap reserve
            1000 size of heap commit
```
Commit size is the initial amount of space the system allocates during thread creation. The minimum commit size is 1 page (typically 4096 bytes). Commit size can also be changed per thread if created using API functions, passing the desired amount as `dwStackSize`:
```cpp
HANDLE CreateThread(
  LPSECURITY_ATTRIBUTES   lpThreadAttributes,
  SIZE_T                  dwStackSize,
  LPTHREAD_START_ROUTINE  lpStartAddress,
  __drv_aliasesMem LPVOID lpParameter,
  DWORD                   dwCreationFlags,
  LPDWORD                 lpThreadId
);
```
Creating a new thread doesn't affect the reserved space for existing ones. The value dwStackSize, however, may affect the reserved stack space for the newly created thread. If you set this value to 0, the system will use the default (the one specified in linker settings), and nothing goes unnoticed. Specifying a higher value than default will round up the reserved space to a multiple of 1 MB. For example: if the default reserved space is left untouched (hence still 1 MB), and you specify 1.2 MB, the system will reserve 2 MB space for the stack you're creating. Similarly, specifying 3.5 MB, the system will reserve 4 MB space for the same stack. Since the main thread cannot be created manually with API calls, it will always use the values embedded into the executable.

Values are always rounded up to the nearest multiple of the page size. A value of 71000 is not valid because it corresponds to 17.3 pages and is not an integral multiple of the page size. The system will round up to 73728 bytes corresponding to 18 pages.

So the entire reserved space is not allocated when the program starts. The system will commit more space on demand as the stack grows. The commit space never shrinks and gets deallocated once the thread terminates. If the thread is almost using its entire room, requesting more space outside the reserved boundary will crash the process. Replicating this crash is very easy. Set the reserved space to 2 MB and run this program:
```cpp
#include <iostream>

void Func()
{
  std::cout << "Press ENTER to allocate more stack space" << std::endl;
  std::cin.get();

  char t[524288];
  Func();
}

void main()
{ 
  Func();
}
```
In case you can't make this program crash, try to reference the array using std::cout or mark it volatile (the compiler might remove the unused array even with optimizations off).\
The compiler generates code for stack allocation and recursive calls, but at the same time, it's smart enough to figure out that the function never returns and will yield a warning:
```
warning C4717: 'Func': recursive on all control paths, function will cause runtime stack overflow
```
Executing the program will end up in a crash after a few allocations, most likely without seeing any error message if you run it from a console, otherwise under the debugger, a message will report the stack overflow exception:
```
Unhandled exception at 0x00007FF60CB121B7 in ConsoleApplication1.exe: 0xC00000FD:
Stack overflow (parameters: 0x0000000000000001, 0x000000C4B3F53000).
```
The stack growing mechanism is implemented using guard pages. A guard page has special protection allocated below the last committed ones. Whenever the stack requires more space than committed pages can offer, trying to cross the boundary where the guard pages are stored, force the system to commit more space (reclaiming the space taken by current guard pages) and setup again guard pages below the newly committed ones. The commit on-demand continues till there is no more space for another guard page, and the stack overflow exception arises.

[VMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/vmmap) is a free utility from the SysInternals suite to inspect the stack growing behavior. To understand how the system handles memory allocations we need to modify the previous example to print more information:
```cpp
#include <Windows.h>
#include <iostream>
#include <iomanip>

void Func()
{
  CONTEXT ctx{0};
  ctx.ContextFlags = CONTEXT_CONTROL;

  //roughly get RSP position
  GetThreadContext(GetCurrentThread(), &ctx);

  std::cout << "RSP ~" << std::hex << std::setw(16) << std::setfill('0') << ctx.Rsp << std::endl;
  std::cout << " Press ENTER to allocate more stack space" << std::endl;
  std::cin.get();

  char t[204800];
  Func();
}

void main()
{
  std::cout << "Current thread ID: " << GetCurrentThreadId() << std::endl;
  Func();
}
```
For this example, I use 2 MB of reserved size and 512 KB of commit size. Run the program and select it in VMMap (click `Show All Processes` if the executable is not in the list). VMMap highlights stack information in orange color. If you wonder why there are many stacks, the reason is the default process thread pool. After a period of inactivity, those threads will be gone. Locate the corresponding stack with the same ID printed to the console and expand the list. In my case, the thread ID was 5920:
```
Address               Type           Size     Committed  Protection        Details
__________________________________________________________________________________________
[-]000000EA05E00000    Thread Stack  2,048 K  524 K      Read/Write/Guard  Thread ID: 5920
     000000EA05E00000  Thread Stack  1,524 K             Reserved
     000000EA05F7D000  Thread Stack     12 K   12 K      Read/Write/Guard
     000000EA05F80000  Thread Stack    512 K  512 K      Read/Write
```
Understanding what's happening here is straightforward. 2 MB of memory have been reserved from 000000EA05E00000 to 000000EA06000000:
```
000000EA06000000-000000EA05E00000 = 200000h = 2097152 bytes = 2 MB
```
The stack starts from the end, growing toward the base address (remember, the stack grows toward lower addresses). The list reports 3 blocks of memory located within the same contiguous 2 MB range. Each of them has a different base address.

512 KB block (the bottom one) is the range of committed memory currently used by the stack. Since it started, to the point of waiting for user input, the stack is consuming 512 KB. It doesn't mean the stack uses that amount exactly because the space is allocated using the granularity established in the system, and the stack-top is somewhere inside that range. Also, remember that the committed space never shrinks even when functions return releasing their frame.

12 KB block is the amount used for the guard page mechanism (note `Guard` in `Protection` column). A page is typically 4 KB, so the system is using 3 pages to watch the boundary. Every time the stack tries to grow touching the guard pages, 12 KB minimum will be committed, and 3 other guard pages will be placed under the newly committed ones. We don't need to care much about the details, and we only need to remember that the system is watching the border at 000000EA05F80000 to check if the stack tries to grow over that address.

1,524 KB is the space left out of 2 MB, and still not used. Its protection says `Reserved` as it's reserved, not allocated virtual space.

From my program output, RSP is roughly at 000000EA05FCD5E8, and its distance to the not-yet-committed space (guard zone) is:
```
000000EA05FCD5E8-000000EA05F80000 = ~300 KB
```
This 300 KB of space is within the committed range, so the stack can use it without crossing the border the system is watching. Return to the running program and press ENTER to allocate 200 KB more. Then press F5 in VMMap to refresh the view. The latest allocation shouldn't trigger memory commit, and all information remains the same:
```
Address               Type           Size     Committed  Protection        Details
__________________________________________________________________________________________
[-]000000EA05E00000    Thread Stack  2,048 K  524 K      Read/Write/Guard  Thread ID: 5920
     000000EA05E00000  Thread Stack  1,524 K             Reserved
     000000EA05F7D000  Thread Stack     12 K   12 K      Read/Write/Guard
     000000EA05F80000  Thread Stack    512 K  512 K      Read/Write
```
But stack-top is getting closer to the border. Grab the new RSP value and calculate the distance again:
```
000000EA05F9B088-000000EA05F80000 = ~100 KB
```
This time the committed space isn't enough to hold another 200 KB. It's inevitable during the next allocation, and the stack will touch the guard pages. Press ENTER to perform another allocation and refresh the VMMap view again:
```
Address               Type           Size     Committed  Protection        Details
__________________________________________________________________________________________
[-]000000EA05E00000    Thread Stack  2,048 K  524 K      Read/Write/Guard  Thread ID: 5920
     000000EA05E00000  Thread Stack  1,424 K             Reserved
     000000EA05F64000  Thread Stack     12 K   12 K      Read/Write/Guard
     000000EA05F67000  Thread Stack    612 K  612 K      Read/Write
```
The base address for the 2 MB block is still the same and will not change till the thread exists. The commit size changed from 512 KB to 612 KB, so the system allocated 100 KB more for the stack. Estimating this 100 KB cannot be precise in this context because it's based on the exact location of RSP and space taken by local variables and other detail. The block of 612 KB now starts at the previous address minus the amount of newly allocated space:
```
000000EA05F67000 = 000000EA05F80000-19000h(100 KB)
```
12 KB of guard space is sitting again below the committed one:
```
000000EA05F64000 = 000000EA05F67000-3000h(12 KB)
```
This allocation procedure continues till the stack requests space below the address 000000EA05E00000 and raises the stack overflow exception. Dividing the program into functions helps to manage the stack. Not only does the program become more modular and readable, but by using functions, it's possible to perform computation and return, releasing the allocated frame and limiting the chances of running out of space.\
As a general recommendation, it's better to allocate a manageable amount of space to minimize the chances to overflow, especially during recursive calls. You can increase the reserved size if you have many overflow occurrences. Start as small as possible for the initial commit size and let the system commit space on-demand because too much-unused commit space is a waste of resources if never used.

###### ALIGNMENT
The stack must be aligned to the current platform word size. For 32-bit code, ESP aligns to 4-byte boundaries. It means ESP always holds an address multiple of 4. Pushing a `char` or a `short` onto the stack won't alter ESP by sizeof(char) or sizeof(short). Those values are promoted to 32-bit integers before the push. The function below highlights this behavior:
```cpp
int Test(char c, short s)
{
  return static_cast<int>(c) + static_cast<int>(s);
}

auto x = Test('a',5);
```
The procedure call and the stack right just before the invocation may look like this:
```
009E1014  push 5
009E1016  push 61h
009E1018  call 09E1000h

    010FF96C  009e127e
ESP 010FF970  00000000
    010FF974  010ff9bc
```
From the Assembly fragment, there is no way to figure out that 5 and 97 are 8-bit and 16-bit values. Both seem 32-bit integer and are pushed onto the stack using 4-byte alignment:
```
    push 5:                           push 61h:
    010FF968  00f1f000            ESP 010FF968 |00000061| [c]
ESP 010FF96C |00000005| [s]           010FF96C  00000005  [s]
    010FF970  00000000                010FF970  00000000
    010FF974  010ff9bc                010FF974  010ff9bc
```
ESP always holds an address multiple of 4; also, both items are 4-byte aligned (another chapter will discuss the data alignment). Different story when the same items are part of a structure allocated in memory, they can be aligned to non-multiple-of-4 addresses because their natural alignment is smaller than 4:
```cpp
struct Test
{
  short s = 5;
  char  c = 'a';
};
```
```
00009FFF  00cfff44
0000A000 |**610005| [c][s]
0000A004  ccff0000
```
`push` instructions in Assembly code behave differently depending on the operand and the current platform. It's still possible to misalign the stack in 32-bit and 64-bit code. Take a look at this:
```asm
;32-bit:
push ecx ;OK push the content of ecx onto the stack taking 4 bytes
push 1   ;OK push a 32-bit integer onto the stack still taking 4 bytes
push al  ;ERROR byte register cannot be push operand
push ax  ;OK but push a 16-bit value onto the stack misalign ESP to 2-byte boundary
```
```asm
;64-bit:
push rcx ;OK push the content of RCX onto the stack taking 8 bytes
push 1   ;OK push a 64-bit integer onto the stack still taking 8 bytes
push eax ;ERROR invalid operand. 32-bit register cannot be encoded in 64-bit push instructions
push al  ;ERROR byte register cannot be push operand
push ax  ;OK but push a 16-bit value onto the stack misalign RSP to 2-byte boundary
```
Pushing 16-bit values is always bad because it pushes exactly 2 bytes. This is one reason why the compiler extends the values to larger types. Consider the promotion a sort of padding to keep the stack alignment consistent (alignment and padding are covered in deep later). At a high level, this happens transparently to the programmer: the compiler pushes a `char` as 32-bit integer, but at the same time, it generates instructions to read the lower 8-bit part whenever the `char` is needed for computations.

For 64-bit programs, the stack keeps the alignment to 16-byte boundaries with some exceptions. We need to distinguish between leaf and non-leaf functions to understand the conditions that alleviate keeping the stack aligned. A leaf procedure is a function that:
* Doesn't alter RSP content
* Doesn't call other functions
* Doesn't allocate stack space
* Doesn't save non-volatile registers
* Doesn't use exception-handling mechanisms

A non-leaf function can do everything listed above and requires a function frame (the non-leaf is called frame-function in this case). A non-leaf function that calls another function must align the stack to 16-byte boundary before executing the call. The call pushes the return address onto the stack taking 8 bytes making the stack-top unaligned again; hence the callee must re-align the stack (except for leaf functions that are not required to do so). We'll see all of these details in the proper chapter.
