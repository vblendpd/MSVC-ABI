> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### CALLING CONVENTIONS
A calling convention is a binary protocol (or a set of rules) that describes how function calls are implemented at the low level. Functions are called procedures in Assembly language. When translating from C++ to machine code, the compiler uses this set of rules to pass arguments to the child function (callee), return values to the parent (caller), restore execution after the call site, and so on.

The compiler implements different calling conventions, and some of them are platform-specific. Whenever the current platform does not recognize the specified calling convention, the compiler will ignore it and use the default one. This auto-selection is particularly relevant for 64-bit programs, which only accept `__fastcall` regardless of which convention is specified (except for `__vectorcall`, which is also available in 64-bit). Some 32-bit conventions are now deprecated and no longer used (`__pascal` `__fortran` `__syscall`). In Visual Studio project settings, there is an option that allows you to change the current default `Calling Convention`:
```
Properties > C/C++ > Advanced
```
* **__cdecl**\
The default calling convention for C/C++. Pushes parameters from right to left onto the stack, and the caller cleans up the stack.
* **__fastcall**\
The default calling convention for 64-bit programs. Arguments are made available to functions using registers whenever possible. The rest are pushed onto the stack from right to left. In 32-bit, the callee cleans up the stack; in 64-bit, the caller is responsible for cleaning up the stack.
* **__stdcall**\
Win32 API calls use this convention. Parameters are pushed from right to left onto the stack, and the callee cleans up the stack.
* **__vectorcall**\
Supported in native code only if SSE2 or above is enabled. This convention uses more registers than fastcall, and its primary purpose is to speed up functions that operate on several floating-point or vector values.

There is a special calling convention called `__thiscall` used for member functions. Parameters are still processed from right to left, and the callee cleans up the stack. The `this` pointer is passed via ECX in 32-bit or RCX in 64-bit. `__thiscall` is ignored in 64-bit code because register-passing of arguments is already the default (fastcall), but the number of available registers for parameters in this case is N-1, since the first register is occupied by `this`.

You can specify a different calling convention directly in a function declaration. Assuming the default convention is `cdecl`, the following functions will be compiled as `cdecl` and `stdcall` respectively:
```cpp
void Func1(){}

void __stdcall Func2(){}
```
This is valid for 32-bit only. In 64-bit, the compiler transparently replaces them with `fastcall`. The calling convention specifier takes effect in function declarations only. Adding it to the function definition is redundant. Using it only in a function definition will not work. Examples for 32-bit:
```cpp
//assume default __cdecl
int GetValue();

//error: different type modifier
int __stdcall GetValue() {
  return 10;
}

class Util
{
public:
  //assumed default __stdcall
  int GetValue() const;
};

//error: different type modifier
int __fastcall Util::GetValue() const {
  return 1;
}

class Vector
{
public:
  int __stdcall GetX() const;
  int __stdcall GetY() const;

  int x = 0;
  int y = 0;
};

//use __stdcall as specified in the function declaration
int Vector::GetX() const {
  return x;
}

//superfluous
int __stdcall Vector::GetY() const {
  return y;
}
```
The current calling convention defines how values are made available to procedures. Some conventions push arguments onto the stack, while others use a combination of stack and CPU registers. Strings and arrays (built-in and user-defined types) are passed using pointers to the first element and never as immediate values. Vector types `__m128` `__m256` `__m512` have different rules. Passing vector types by value will eventually use SIMD registers, while passing references or pointers to vector types will use general-purpose registers (or the stack if no registers are available). There is a special case of vector type commonly known as a vector aggregate. A vector aggregate is a POD type containing homogeneous vector types:
```cpp
//OK homogeneous
struct VA1
{
  __m128 a;
  __m128 b;
};

//OK homogeneous
struct VA2
{
  __m256 a;
  __m256 b;
  __m256 c;
  __m256 d;
};

//OK homogeneous
struct VA3
{
  __m128 v[3];
};

//not a VA because mixes __m128 and __m256
struct VA4
{
  __m128 a;
  __m256 b;
};
```
When using `vectorcall`, the compiler can propagate these structures by copying all individual members to different SIMD registers. We will take a look at a few examples later.

Built-in types are returned to the caller using a register whenever the data fits in one. Integer values (or types convertible to integers) of length `1` `2` `4` bytes return using EAX in 32-bit. This includes enumerators, pointers, and references whose size fits into EAX. For 64-bit code, RAX is the default register, and the list extends to `1` `2` `4` `8` byte integer values. User-defined objects that fit into a register can return using a register only if they are a Plain-Old-Data (POD) type. This means the object must not have:

* User-defined constructors
* Destructor
* Assignment operator
* Base class
* Virtual functions
* Non-static protected members
* Non-static private members
* Non-static reference members

>Due to the evolution of the language, the old concept of POD is now obsolete, and std::is_pod should no longer be the first choice. A POD now defines a trivial type with standard layout, which means no C++ features that are incompatible with C.

User-defined objects returned from functions are also subject to alignment and packing rules that determine whether they are eligible for register return. For example, in 64-bit code, of all the following objects, only `A` `B` `C` `D` are eligible to return using RAX:
```cpp
//sizeof(A) = 8
struct A
{
  char c[4];
  int  i;
};

//sizeof(B) = 8
#pragma pack(2)
struct B
{
  char c1[2];
  int  i;
  char c2[2];
};
#pragma pack()

//sizeof(C) = 8
//POD can have normal member functions
struct C
{
  char c1;
  char c2;
  int  i;
  void Func(){}
};

//sizeof(D) = 8
//POD can have pointer members
struct D
{
  int* ptr = &x;
};

//sizeof(E) = 12
//POD but too big for RAX
struct E
{
  char c1[2];
  int  i;
  char c2[2];
};

//sizeof(F) = 12
//POD but too big for RAX
struct F
{
  char            c1;
  alignas(4) char c2;
  int             i;
};

//sizeof(G) = 8
//not a POD because of the reference member
struct G
{
  int& ref = x;
};

//sizeof(Base) = 1
struct Base
{
  char c1;
};

//sizeof(Derived) = 8
//not a POD because of inheritance
struct Derived : public Base
{
  char c2;
  int  i;
}
```
Any user-defined object that does not fit into a register will be returned by copy (these objects are typically temporary objects when optimization is disabled). The caller allocates space for the temporary object and passes the return address for that temporary object onto the stack (or moves it to ECX/RCX if register passing is available) prior to calling the child function. The callee fills that storage with the temporary object. Once execution returns to the caller, the object is copied from temporary storage to the destination (which typically corresponds to the local variable in the parent function). With the help of some pseudo-code and a pseudo-stack-frame, we can see this mechanism in action:
```cpp
struct XYZ
{
  int x = 1;
  int y = 2;
  int z = 3;
};

XYZ Create()
{
  XYZ t;
  return t;
}

void main()
{
  auto local_xyz = Create();
}
```
According to what has been discussed above, `main` should pass the address of the temporary XYZ object being returned from the callee function. These bytes will be copied to `local_xyz` at some point later. Suppose the current frame looks like this:
```
RSP 000000DF321AF820  00000000
    000000DF321AF824  00000000
    000000DF321AF828  00000000
    000000DF321AF82C  00000000
    000000DF321AF830  00000000  [local_xyz.x]
    000000DF321AF834  00000000  [local_xyz.y]
    000000DF321AF838  00000000  [local_xyz.z]
    000000DF321AF83C  00000000
    000000DF321AF840  00000000  [temp.x]
    000000DF321AF844  00000000  [temp.y]
    000000DF321AF848  00000000  [temp.z]
```
The `main` function has allocated stack space for local variables and more. The object returned from `Create` is a temporary object (`temp`). The compiler places this data in the corresponding 12 bytes shown above. The other 12 bytes for `local_xyz` reside at a separate location. The function moves the address of the temp object to RCX and calls `Create`:
```asm
lea  rcx, [rsp+20h] ;rsp+20h = 000000DF321AF840 = &__temp
call Create
```
The callee uses the address in RCX to fill the temp bytes:
```
000000DF321AF83C  00000000
000000DF321AF840 |00000001| [temp.x]
000000DF321AF844 |00000002| [temp.y]
000000DF321AF848 |00000003| [temp.z]
```
When execution returns to `main`, the temp object is copied to `local_xyz` using string instructions:
```asm
mov rdi, 000000DF321AF830 ;destination = &local_xyz
mov rsi, 000000DF321AF840 ;source = &temp
mov ecx, 0Ch              ;amount = 12 bytes
rep movsb
```
```
000000DF321AF830 |00000001| [local_xyz.x]
000000DF321AF834 |00000002| [local_xyz.y]
000000DF321AF838 |00000003| [local_xyz.z]
000000DF321AF83C  00000000
000000DF321AF840  00000001  [temp.x]
000000DF321AF844  00000002  [temp.y]
000000DF321AF848  00000003  [temp.z]
```
Most of the upcoming examples move return values around (built-in or user-defined objects) as seen in the XYZ example. Return Value Optimization (RVO) removes the intermediate temporary object by copying the return object directly to the destination. In the previous example, the callee would fill `local_xyz` storage directly. For built-in types, there is little to no overhead. For complex and expensive-to-copy objects, there may be a non-negligible performance cost if RVO does not take place.

In 32-bit code, 64-bit registers are unavailable, but 64-bit integers are valid data types and are passed to functions using 2 push instructions (to push the low and high 32-bit parts). The same type is returned to the caller using the register pair EDX:EAX (high part in EDX and low part in EAX). The use of register pairs is not limited to 64-bit integers. POD objects that fit into 64 bits can use EDX:EAX as well (or RAX in 64-bit). The same rule does not apply in 64-bit: neither 128-bit integers nor 128-bit POD types are returned using RDX:RAX. In both 32-bit and 64-bit builds, the maximum size for a POD type to be returned in a register is 64 bits (vector types returning via SIMD registers is a separate case).

We can summarize the return mechanism for 32-bit and 64-bit:
```C++
struct S32  { int a; }
struct S64  { int a; int b;}
struct F2   { float f[2]; }
struct F4   { float f[4]; }
struct CHAR4{ char c[4]; }
struct CHAR8{ char c[8]; }
struct Vec2 { int x; int y; }
struct Vec3 { int x; int y; int z; }
struct C4S2 { char c[4]; short s[2]; }
enum class Color{}
enum class Color2 : long long{}
```
32-bit:
```
AL      = char, unsigned char, __int8
AX      = short, unsigned short, __int16, char16_t
EAX     = int, unsigned int, long, unsigned long, char*, unsigned int*, unsigned int&, S32, CHAR4, Vec2*, Vec3&
EDX:EAX = long long, unsigned long long, __int64, Color2, S64, Vec2, CHAR8, F2, C4S2
XMM0    = float, double
XMM0    = __m128
YMM0    = __m256
ZMM0    = __m512
STACK   = Vec3 //Address of temp return object provided by the caller in ECX
STACK   = F4   //Address of temp return object provided by the caller in ECX
```
64-bit:
```
AL    = char, unsigned char, __int8
AX    = short, unsigned short, __int16, char16_t
EAX   = int, unsigned int, long, unsigned long, S32, CHAR4
RAX   = long long, unsigned long long, __int64, Color2, S64, Vec2, CHAR8, F2, C4S2, Vec2*, int*, char&
XMM0  = float, double
XMM0  = __m128
YMM0  = __m256
ZMM0  = __m512
STACK = Vec3 //Address of temp return object provided by the caller in RCX
STACK = F4   //Address of temp return object provided by the caller in RCX
```
Sometimes the Assembly output reveals discrepancies. Consider this small code fragment and its corresponding machine code:
```cpp
unsigned char Test()
{
  return 65;
}

auto x = Test(); 
```
```asm
mov  al,41h  
ret

call 00007FF7DBAB1000  
mov  byte ptr [rsp+20h],al
```
The 8-bit value is sent to the caller using AL. The caller uses the AL content to initialize `x`. Let us return a `short` type instead:
```cpp
unsigned short Test()
{
  return 65;
}

auto x = Test(); 
```
```asm
mov  eax,41h  
ret

call 00007FF744041000  
mov  word ptr [rsp+20h],ax
```
According to the return table, the value 65 must go to AX. We observe that the entire 32-bit register is used instead. This is perfectly fine because whenever the code needs the 16-bit value, it knows where to find it. In fact, the caller uses AX to initialize `x`, addressing the memory location using `WORD PTR` to write only 16 bits to `x`'s storage.
> Stack alignment is covered in the Stack chapter

The compiler sometimes sign-extends or zero-extends values using `MOVSX` and `MOVZX` instructions. `MOVSX` fills the remaining portion of the first operand with the sign bit of the second operand. `MOVZX` fills the remaining bits with zero:
```
EAX |........ ........ ........ 11110010|

MOVSX EAX, AL

EAX |11111111 11111111 11111111 11110010|
```
```
EAX |........ ........ ........ 11110010|
 
MOVZX EAX, AL
 
EAX |00000000 00000000 00000000 11110010|
```
In addition, it is worth remembering that instructions operating on 8-bit or 16-bit operands affect only the 8-bit or 16-bit portion of a register (in both 32-bit and 64-bit code). When using 32-bit operands in 64-bit mode, those instructions automatically zero-extend the upper part of the register:
```asm
xor rax, rax
not rax         ;RAX = FFFFFFFFFFFFFFFF

mov al, 1       ;RAX = FFFFFFFFFFFFFF01
mov ax, 0AAAAh  ;RAX = FFFFFFFFFFFFAAAA
mov eax, 1      ;RAX = 0000000000000001
```
These sign-extending and zero-extending operations have applications in out-of-order execution. Instructions are decoded and broken into micro-operations that can execute in parallel. The CPU can identify unrelated instructions (e.g., when the next instruction does not depend on the result of the current one). To achieve this, the CPU implements a mechanism called register renaming, and the notion of a register must be virtualized to some degree (in practice, a register may not be implemented as a single, fully independent physical entity). Whenever an instruction uses part of a register and the upper portion is left untouched, the CPU cannot determine whether that untouched portion, resulting from a prior operation, will be used again shortly. `MOVZX` tells the CPU that a register (or part of it) can be allocated for internal use without affecting subsequent instructions.

###### 64-BIT FASTCALL EXAMPLE 1
In 64-bit programs, fastcall uses a total of 4 registers for parameter passing. Any remaining parameters go onto the stack from right to left, and the caller cleans up the stack. The available registers are: `RCX` `RDX` `R8` `R9` for integers and types interpretable as integers. `XMM0` `XMM1` `XMM2` `XMM3` are used for floating-point values (single-precision and double-precision). Vector types do not use SIMD registers. A vector type, whether passed by value, reference, or pointer, is passed as a pointer using one of the 4 integer registers, or via the stack if no registers are available. A smaller register can be used whenever the type is smaller than the accommodating register (e.g., ECX for a 32-bit integer, R8W for a 16-bit integer, R9B for an 8-bit integer, etc.). The choice of register for a given parameter depends on the type and position of that parameter. The convention uses a position-based mapping according to the following scheme:
```
     RCX      RDX      R8       R9          STACK...
     XMM0     XMM1     XMM2     XMM3        STACK...
```
```
     ECX      EDX
Func(int,     int)

     RCX      EDX
Func(int*,    enum)

     ECX      XMM1     R8D
Func(int,     float,   int)

     XMM0     XMM1     R8D      R9D
Func(float,   float,   int,     int)

     ECX      XMM1     R8D      XMM3        STACK
Func(int,     float,   int,     float,      int)

     RCX      XMM1     R8       XMM3        STACK
Func(__m128&, float,   __m256*, float,      int)

     RCX      XMM1     R8D      XMM3        STACK
Func(char&,   float,   int,     double,     int)

     ECX      EDX      XMM2     XMM3        STACK    STACK
Func(char,    int,     double,  float,      double,  char) 

     XMM0     EDX      R8D      R9          STACK    STACK
Func(float,   int,     char,    __m256,     double,  __m256)

     RCX      XMM1     R8       R9D         STACK    STACK
Func(float*,  float,   __m128,  char,       int,     int)

     RCX      RDX      R8       R9          STACK    STACK
Func(Data*,   Data&,   __m128,  __m256,     int*,    unsigned char)

     ECX      EDX      XMM2     R9          STACK
Func(char,    char,    float,   const int&, __m128)

      RCX     RDX      XMM2     XMM3
Func(__int64, int[][4],float,   float)

     RCX      EDX      R8D      R9          STACK    STACK
Func(__m128,  int,     char,    __int64,    __int32, float)

     ECX      EDX      R8       R9D         STACK 
Func(int,     int,     __m256,  char8_t,    int[][4])
```
8-bit and 16-bit literals can be encoded directly into instructions and use the smallest available register. Non-literals are zero-extended to 32-bit integers. Here is a typical example:
```cpp
void Func(char a, short b, char c, short d)
{}
```
```
     CL   DX   R8B   R9W
Func('a', 10,  'b',  11);

mov  r9w,0Bh
mov  r8b,62h
mov  dx, 0Ah
mov  cl, 61h
```
```
char  c = 'a';
short s = 10;

     ECX  EDX  R8D  R9D
Func(c,   s,   c,   s);

movzx r9d,word ptr [rbp+4]
movzx r8d,byte ptr [rbp]
movzx edx,word ptr [rbp+4]
movzx ecx,byte ptr [rbp]
```
Registers in the following list are considered volatile and may be freely used or overwritten:
```
RAX
RCX
RDX
R8
R9
R10
R11
XMM0/YMM0
XMM1/YMM1
XMM2/YMM2
XMM3/YMM3
XMM4/YMM4
XMM5/YMM5
```
This means you cannot rely on the content of any of them being preserved after invoking a procedure. Consider the following fragment:
```asm
mov  rdx, 1
mov  rcx, 16
call [__imp_malloc] ;allocate 16 bytes
```
After the call, RCX and RDX may no longer contain 16 and 1 respectively.\
Registers in the following list are non-volatile, and their content must be preserved across function calls. Any procedure that wishes to use a non-volatile register must save its content somewhere (the stack is typically used for this) and restore it before returning.
```
RBX
RBP
RSP
RSI
RDI
R12
R13
R14
R15
XMM6/YMM6
XMM7/YMM7
XMM8/YMM8
XMM9/YMM9
XMM10/YMM10
XMM11/YMM11
XMM12/YMM12
XMM13/YMM13
XMM14/YMM14
XMM15/YMM15
```
The ABI specifies that the caller must allocate 32 bytes of home space (also called shadow space) on the stack before calling the child function, regardless of the type and number of parameters. The home space sits above the return address and is available as scratch space for `RCX` `RDX` `R8` `R9`. The rationale for saving these registers to the corresponding stack locations is not immediately obvious. Consider what happens when a parameter is passed through a register and not saved on the stack: registers are scarce resources with a reasonable chance of having their value overwritten during computation. Once a procedure returns to the caller, it is often very difficult to reconstruct the original parameters, as no copies remain on the stack, which turns memory dump analysis into a significant challenge. Parameters copied to the stack, on the other hand, are easier to identify (provided nothing has overwritten them).\
I have observed this home space only partially utilized in debug builds. In a Release build, the compiler will not generate instructions to save those registers, as unnecessary memory accesses are avoided. The ABI only requires that the space be allocated; its actual usage is left to the discretion of the compiler or programmer.

We now analyze the first example, in which RBP is unused.
```cpp
int Compute(int a, int b, int c, int d, float e)
{
  return a + b + c + d + static_cast<int>(e);
}

void main()
{
  int x = Compute(1,2,3,4,5.0f);    
}
```
```asm
RIP 00007FF7AC5E1030  sub   rsp,48h
    00007FF7AC5E1034  movss xmm0,dword ptr [00007FF7AC5E2DD0h]  
    00007FF7AC5E103C  movss dword ptr [rsp+20h],xmm0  
    00007FF7AC5E1042  mov   r9d,4  
    00007FF7AC5E1048  mov   r8d,3  
    00007FF7AC5E104E  mov   edx,2  
    00007FF7AC5E1053  mov   ecx,1  
    00007FF7AC5E1058  call  00007FF7AC5E1000  
    00007FF7AC5E105D  mov   dword ptr [rsp+30h],eax  
    00007FF7AC5E1061  xor   eax,eax  
    00007FF7AC5E1063  add   rsp,48h  
    00007FF7AC5E1067  ret
```
```
    0000008EE698FCA4  00000000
RSP 0000008EE698FCA8  ac5e1258
    0000008EE698FCAC  00007ff7
    0000008EE698FCB0  00000000
```
We start at RIP. The function resizes the stack to make space for 72 bytes:
```
    0000008EE698FC5C  00000000
RSP 0000008EE698FC60  00000001
    0000008EE698FC64  00000000
    0000008EE698FC68  00000000
    0000008EE698FC6C  9c002400
    0000008EE698FC70  ac5e21a8
    0000008EE698FC74  00007ff7
    0000008EE698FC78  ac5e1495
    0000008EE698FC7C  00007ff7
    0000008EE698FC80  0000001f
    0000008EE698FC84  00000000
    0000008EE698FC88  00000001
    0000008EE698FC8C  00000000
    0000008EE698FC90  00000000
    0000008EE698FC94  00000000
    0000008EE698FC98  00000000
    0000008EE698FC9C  00000000
    0000008EE698FCA0  00000000
    0000008EE698FCA4  00000000
```
According to the parameter-passing scheme, the first 4 arguments can use registers. `ECX` `EDX` `R8` `R9` accommodate `a` `b` `c` `d`, while `e` will use the stack. The floating-point value is first loaded into an XMM register and then copied to its stack location at RSP+32:
```
    0000008EE698FC5C  00000000
RSP 0000008EE698FC60  00000001
    0000008EE698FC64  00000000
    0000008EE698FC68  00000000
    0000008EE698FC6C  9c002400
    0000008EE698FC70  ac5e21a8
    0000008EE698FC74  00007ff7
    0000008EE698FC78  ac5e1495
    0000008EE698FC7C  00007ff7
    0000008EE698FC80 |40a00000| [e] [RSP+32]
    0000008EE698FC84  00000000
    0000008EE698FC88  00000001
    0000008EE698FC8C  00000000
    0000008EE698FC90  00000000
    0000008EE698FC94  00000000
    0000008EE698FC98  00000000
    0000008EE698FC9C  00000000
    0000008EE698FCA0  00000000
    0000008EE698FCA4  00000000
```
Then the remaining parameters are moved to their corresponding registers from right to left, as shown in the Assembly code. The `call` instruction pushes the return address onto the stack (taking 8 bytes, since addresses in 64-bit are 8 bytes long) and transfers control to `Compute` by loading its first instruction address, 00007FF7AC5E1000, into RIP.
```
    0000008EE698FC54  00000000
RSP 0000008EE698FC58 |ac5e105d| [ret_main]
    0000008EE698FC5C |00007ff7| [ret_main]
    0000008EE698FC60  00000001
    0000008EE698FC64  00000000
    0000008EE698FC68  00000000
    0000008EE698FC6C  9c002400
    0000008EE698FC70  ac5e21a8
    0000008EE698FC74  00007ff7
    0000008EE698FC78  ac5e1495
    0000008EE698FC7C  00007ff7
    0000008EE698FC80  40a00000  [e] [RSP+40]
    0000008EE698FC84  00000000
    0000008EE698FC88  00000001
    0000008EE698FC8C  00000000
    0000008EE698FC90  00000000
    0000008EE698FC94  00000000
    0000008EE698FC98  00000000
    0000008EE698FC9C  00000000
    0000008EE698FCA0  00000000
    0000008EE698FCA4  00000000
```
```asm
RIP 00007FF7AC5E1000  mov       dword ptr [rsp+20h],r9d
    00007FF7AC5E1005  mov       dword ptr [rsp+18h],r8d
    00007FF7AC5E100A  mov       dword ptr [rsp+10h],edx
    00007FF7AC5E100E  mov       dword ptr [rsp+8],ecx
    00007FF7AC5E1012  mov       eax,dword ptr [rsp+10h]
    00007FF7AC5E1016  mov       ecx,dword ptr [rsp+8]
    00007FF7AC5E101A  add       ecx,eax
    00007FF7AC5E101C  mov       eax,ecx
    00007FF7AC5E101E  add       eax,dword ptr [rsp+18h]
    00007FF7AC5E1022  add       eax,dword ptr [rsp+20h]
    00007FF7AC5E1026  cvttss2si ecx,dword ptr [rsp+28h]
    00007FF7AC5E102C  add       eax,ecx
    00007FF7AC5E102E  ret
```
The code begins by copying data from registers to the stack at the highlighted locations below:
```
    0000008EE698FC54  00000000
RSP 0000008EE698FC58  ac5e105d  [ret_main]
    0000008EE698FC5C  00007ff7  [ret_main+
    0000008EE698FC60 |00000001| [a] [RSP+8]
    0000008EE698FC64  00000000
    0000008EE698FC68 |00000002| [b] [RSP+16]
    0000008EE698FC6C  9c002400
    0000008EE698FC70 |00000003| [c] [RSP+24]
    0000008EE698FC74  00007ff7
    0000008EE698FC78 |00000004| [d] [RSP+32]
    0000008EE698FC7C  00007ff7
    0000008EE698FC80  40a00000  [e] [RSP+40]
    0000008EE698FC84  00000000
    0000008EE698FC88  00000001
    0000008EE698FC8C  00000000
    0000008EE698FC90  00000000
    0000008EE698FC94  00000000
    0000008EE698FC98  00000000
    0000008EE698FC9C  00000000
    0000008EE698FCA0  00000000
    0000008EE698FCA4  00000000
```
The CPU loads `b` into EAX and `a` into ECX, performs the addition, and stores the result in ECX, which is immediately copied to EAX. `c` and `d` are then added to EAX. The floating-point value at RSP+40 is converted to a 32-bit integer in ECX and added to EAX, which now holds the final result. Since a 32-bit integer is returned to the caller using EAX, the return value is already in the correct place. The `ret` instruction pops the return address 00007FF7AC5E105D from the stack top into RIP:
```
    0000008EE698FC54  00000000
    0000008EE698FC58  ac5e105d  [ret_main]
    0000008EE698FC5C  00007ff7  [ret_main]
RSP 0000008EE698FC60  00000001  [a] [RSP+8]
    0000008EE698FC64  00000000
    0000008EE698FC68  00000002  [b] [RSP+16]
    0000008EE698FC6C  9c002400
    0000008EE698FC70  00000003  [c] [RSP+24]
    0000008EE698FC74  00007ff7
    0000008EE698FC78  00000004  [d] [RSP+32]
    0000008EE698FC7C  00007ff7
    0000008EE698FC80  40a00000  [e] [RSP+40]
    0000008EE698FC84  00000000
    0000008EE698FC88  00000001
    0000008EE698FC8C  00000000
    0000008EE698FC90  00000000
    0000008EE698FC94  00000000
    0000008EE698FC98  00000000
    0000008EE698FC9C  00000000
    0000008EE698FCA0  00000000
    0000008EE698FCA4  00000000
```
The stack is back to the frame it was in before the call. Since RIP now holds the return address (the address of the instruction immediately after the call), execution resumes here:
```
    00007FF7AC5E1058  call 00007FF7AC5E1000  
RIP 00007FF7AC5E105D  mov  dword ptr [rsp+30h],eax  
    00007FF7AC5E1061  xor  eax,eax  
    00007FF7AC5E1063  add  rsp,48h  
    00007FF7AC5E1067  ret
```
The remaining code copies the computed value to `x`'s storage, sets the return value to zero, and releases the previously allocated 72 bytes from the stack:
``` 
    0000008EE698FC78  00000004  [d] 
    0000008EE698FC7C  00007ff7  
    0000008EE698FC80  40a00000  [e]
    0000008EE698FC84  00000000  
    0000008EE698FC88  00000001  
    0000008EE698FC8C  00000000  
    0000008EE698FC90  00000000  
    0000008EE698FC94  00000000  
    0000008EE698FC98  00000000  
    0000008EE698FC9C  00000000  
    0000008EE698FCA0  00000000  
    0000008EE698FCA4  00000000  
RSP 0000008EE698FCA8  ac5e1258  
    0000008EE698FCAC  00007ff7  
    0000008EE698FCB0  00000000
```
The `main` function is about to terminate, and RSP is back to where it was before `main` began execution.

###### 64-BIT FASTCALL EXAMPLE 2
```cpp
struct Vector { int  x; int  y; };
struct Offset { int dx; int dy; };

Vector Compute(const Vector& v, Offset o)
{
  Vector ret;
  ret.x = v.x + o.dx;
  ret.y = v.y + o.dy;
  return ret;
}

void Func()
{
  Vector v;
  v.x = 5;
  v.y = 6;

  Offset o;
  o.dx = 2;
  o.dy = 3;

  auto r = Compute(v, o);
}

void main()
{
  Func();
}
```
We assume `Func` has already been invoked from `main`, so RSP is pointing to the return address for `main`. Disassembly output and current stack:
```asm
00007FF68D8B1040  sub  rsp,48h
00007FF68D8B1044  mov  dword ptr [rsp+28h],5
00007FF68D8B104C  mov  dword ptr [rsp+2Ch],6
00007FF68D8B1054  mov  dword ptr [rsp+20h],2
00007FF68D8B105C  mov  dword ptr [rsp+24h],3
00007FF68D8B1064  mov  rdx,qword ptr [rsp+20h]
00007FF68D8B1069  lea  rcx,[rsp+28h]
00007FF68D8B106E  call 00007FF68D8B1000
00007FF68D8B1073  mov  qword ptr [rsp+30h],rax
00007FF68D8B1078  mov  rax,qword ptr [rsp+30h]
00007FF68D8B107D  mov  qword ptr [rsp+38h],rax
00007FF68D8B1082  add  rsp,48h
00007FF68D8B1086  ret
```
```
RSP 000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
`Func` reserves 72 bytes of space:
```
RSP 000000E7F88FFCF0  00000000
    000000E7F88FFCF4  00000000
    000000E7F88FFCF8  00000000
    000000E7F88FFCFC  00000000
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000000
    000000E7F88FFD14  00000000
    000000E7F88FFD18  00000000
    000000E7F88FFD1C  00000000
    000000E7F88FFD20  00000001
    000000E7F88FFD24  00000000
    000000E7F88FFD28  00000000
    000000E7F88FFD2C  9c002400
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
`Vector` and `Offset` are trivial objects and are directly constructed on the stack without invoking constructors:
```
RSP 000000E7F88FFCF0  00000000
    000000E7F88FFCF4  00000000
    000000E7F88FFCF8  00000000
    000000E7F88FFCFC  00000000
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10 |00000002| [o.dx] [RSP+24]
    000000E7F88FFD14 |00000003| [o.dy]
    000000E7F88FFD18 |00000005| [v.x]  [RSP+28]
    000000E7F88FFD1C |00000006| [v.y]
    000000E7F88FFD20  00000001
    000000E7F88FFD24  00000000
    000000E7F88FFD28  00000000
    000000E7F88FFD2C  9c002400
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
The address of `Vector` (`&v` = RSP+40) calculated with the `lea` instruction goes into RCX. `Offset` is passed by value, meaning it must be copied. Since `Offset` is a POD type that fits within 64 bits, its components can be moved to RDX. The `call` instruction pushes the return address onto the stack and moves the new procedure address into RIP:
```
RSP 000000E7F88FFCE8 |8d8b1073| [ret_Func]
    000000E7F88FFCEC |00007ff6| [ret_Func]
    000000E7F88FFCF0  00000000
    000000E7F88FFCF4  00000000
    000000E7F88FFCF8  00000000
    000000E7F88FFCFC  00000000
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx] [RSP+24]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]  [RSP+28]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
    000000E7F88FFD24  00000000
    000000E7F88FFD28  00000000
    000000E7F88FFD2C  9c002400
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
```asm
RIP 00007FF68D8B1000  mov qword ptr [rsp+10h],rdx
    00007FF68D8B1005  mov qword ptr [rsp+8],rcx
    00007FF68D8B100A  sub rsp,18h
    00007FF68D8B100E  mov rax,qword ptr [rsp+20h]
    00007FF68D8B1013  mov eax,dword ptr [rax]
    00007FF68D8B1015  add eax,dword ptr [rsp+28h]
    00007FF68D8B1019  mov dword ptr [rsp],eax
    00007FF68D8B101C  mov rax,qword ptr [rsp+20h]
    00007FF68D8B1021  mov eax,dword ptr [rax+4]
    00007FF68D8B1024  add eax,dword ptr [rsp+2Ch]
    00007FF68D8B1028  mov dword ptr [rsp+4],eax
    00007FF68D8B102C  mov rax,qword ptr [rsp]
    00007FF68D8B1030  add rsp,18h
    00007FF68D8B1034  ret
```
The first two `mov` instructions save the `Offset` components to RSP+16 and the vector address to RSP+8:
```
RSP 000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
    000000E7F88FFCF0 |f88ffd18| [&v]   [RSP+8]
    000000E7F88FFCF4 |000000e7| [&v]
    000000E7F88FFCF8 |00000002| [o.dx] [RSP+16]
    000000E7F88FFCFC |00000003| [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
    000000E7F88FFD24  00000000
    000000E7F88FFD28  00000000
    000000E7F88FFD2C  9c002400
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
24 bytes are then allocated on the stack:
```asm
RSP 000000E7F88FFCD0  00000000
    000000E7F88FFCD4  00000000
    000000E7F88FFCD8  5ce939ce
    000000E7F88FFCDC  00007ff8
    000000E7F88FFCE0  00000000
    000000E7F88FFCE4  00000000
    000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
    000000E7F88FFCF0  f88ffd18  [&v]   [RSP+32]
    000000E7F88FFCF4  000000e7  [&v]
    000000E7F88FFCF8  00000002  [o.dx] [RSP+40]
    000000E7F88FFCFC  00000003  [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
```
Now the computation begins. The `x` component is read into EAX using the address stored at RSP+32. Since the vector address is the address of its first member, no additional offset is required to read `x`. The offset `dx` is added to EAX, and the result is stored at the stack top.
```
RSP 000000E7F88FFCD0 |00000007| [v.x+o.dx]
    000000E7F88FFCD4  00000000
    000000E7F88FFCD8  5ce939ce
    000000E7F88FFCDC  00007ff8
    000000E7F88FFCE0  00000000
    000000E7F88FFCE4  00000000
    000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
    000000E7F88FFCF0  f88ffd18  [&v]   [RSP+32]
    000000E7F88FFCF4  000000e7  [&v]
    000000E7F88FFCF8  00000002  [o.dx] [RSP+40]
    000000E7F88FFCFC  00000003  [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
```
The vector address is loaded again into RAX. This time a displacement of 4 is required to read `v.y` into EAX. The corresponding offset `o.dy` from RSP+44 is added to EAX, and the result is saved to RSP+4:
```
RSP 000000E7F88FFCD0  00000007  [v.x+o.dx]
    000000E7F88FFCD4 |00000009| [v.y+o.dy]
    000000E7F88FFCD8  5ce939ce
    000000E7F88FFCDC  00007ff8
    000000E7F88FFCE0  00000000
    000000E7F88FFCE4  00000000
    000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
```
The return value is a 64-bit POD type, so it can be returned in a register. The values `7` and `9` are copied from RSP into RAX. The second-to-last instruction reclaims the 24 bytes used by the procedure, releasing local variables and other data:
```
    000000E7F88FFCE4  00000000
RSP 000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
    000000E7F88FFCF0  f88ffd18  [&v]
    000000E7F88FFCF4  000000e7  [&v]
    000000E7F88FFCF8  00000002  [o.dx]
    000000E7F88FFCFC  00000003  [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
```
The `ret` instruction loads the address pointed to by RSP into RIP, returning execution to the next instruction after the call:
```
    000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
RSP 000000E7F88FFCF0  f88ffd18  [&v]
    000000E7F88FFCF4  000000e7  [&v]
    000000E7F88FFCF8  00000002  [o.dx]
    000000E7F88FFCFC  00000003  [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20  00000001
    000000E7F88FFD24  00000000
    000000E7F88FFD28  00000000
    000000E7F88FFD2C  9c002400
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
```
```asm
    00007FF68D8B106E  call 00007FF68D8B1000
RIP 00007FF68D8B1073  mov  qword ptr [rsp+30h],rax
    00007FF68D8B1078  mov  rax,qword ptr [rsp+30h]
    00007FF68D8B107D  mov  qword ptr [rsp+38h],rax
    00007FF68D8B1082  add  rsp,48h
    00007FF68D8B1086  ret
```
The temporary return value is copied to RSP+48 and used to initialize the local variable `r` at RSP+56:
```
    000000E7F88FFCE8  8d8b1073  [ret_Func]
    000000E7F88FFCEC  00007ff6  [ret_Func]
RSP 000000E7F88FFCF0  f88ffd18  [&v]
    000000E7F88FFCF4  000000e7  [&v]
    000000E7F88FFCF8  00000002  [o.dx]
    000000E7F88FFCFC  00000003  [o.dy]
    000000E7F88FFD00  00000002
    000000E7F88FFD04  00000000
    000000E7F88FFD08  5ceb1d56
    000000E7F88FFD0C  00007ff8
    000000E7F88FFD10  00000002  [o.dx]
    000000E7F88FFD14  00000003  [o.dy]
    000000E7F88FFD18  00000005  [v.x]
    000000E7F88FFD1C  00000006  [v.y]
    000000E7F88FFD20 |00000007| [temp_r.x] [RSP+48]
    000000E7F88FFD24 |00000009| [temp_r.y]
    000000E7F88FFD28 |00000007| [r.x]      [RSP+56]
    000000E7F88FFD2C |9c002409| [r.y]
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
    000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
```
The `add` instruction at 00007FF68D8B1082 releases 72 bytes from the stack:
```
    000000E7F88FFD30  00000000
    000000E7F88FFD34  00000000
RSP 000000E7F88FFD38  8d8b1099  [ret_main]
    000000E7F88FFD3C  00007ff6  [ret_main]
    000000E7F88FFD40  0000001f
```
`ret` then pops the address pointed to by RSP into RIP, returning control to the `main` function.

###### 64-BIT FASTCALL EXAMPLE 3
In this example, we will use a class object to inspect how member functions are invoked. The following object provides a simple utility for linear interpolation:
```cpp
class LI
{
public:
  LI() = default;

  LI(float a, float b)
  {
      this->a = a;
      this->b = b;
  }

  float Get(float t)
  {
      return a + t * (b - a);
  }
private:    
  float a = 0.0f;
  float b = 1.0f;
};

void main()
{
  LI li;

  float x = li.Get(0.5f);
}
```
Current frame and machine code for `main`:
```asm
    00007FF7E0301040  sub   rsp,38h
    00007FF7E0301044  lea   rcx,[rsp+28h]
    00007FF7E0301049  call  00007FF7E0301070
RIP 00007FF7E030104E  movss xmm1,dword ptr [00007FF7E0302DD0h]
    00007FF7E0301056  lea   rcx,[rsp+28h]
    00007FF7E030105B  call  00007FF7E0301000
    00007FF7E0301060  movss dword ptr [rsp+20h],xmm0
    00007FF7E0301066  xor   eax,eax
    00007FF7E0301068  add   rsp,38h
    00007FF7E030106C  ret
```
```
    0000009329D7F87C  00007ff7
RSP 0000009329D7F880  29d7f8a8
    0000009329D7F884  00000093
    0000009329D7F888  e03014c9
    0000009329D7F88C  00007ff7
    0000009329D7F890  0000001f
    0000009329D7F894  00000000
    0000009329D7F898  00000001
    0000009329D7F89C  00000000
    0000009329D7F8A0  00000000
    0000009329D7F8A4  00000000
    0000009329D7F8A8  00000000
    0000009329D7F8AC  3f800000
```
The function makes space for 56 bytes. Look at the code and determine what is stored at RSP+28h. It is used twice, each time immediately before a `call` instruction. If the second call is to `Get`, the first should be the constructor. You know that there is an implicit argument for each non-static member function, and that argument is the `this` pointer. I mentioned earlier that the `this` pointer is passed via ECX in 32-bit and RCX in 64-bit. The `this` pointer is what distinguishes a member function from a free global function. C++ has visibility rules that prevent invoking non-static member functions without an object instance. There is a way to circumvent this, and we will see it when we study object memory layouts in a later chapter. For the current example, we only need to know that the `this` pointer is passed implicitly by the compiler, transparently to the programmer.\
The `LI` object appears to be constructed at RSP+40, where you can spot its member values `0.0` and `1.0`:
```
this 0000009329D7F8A8  00000000 [li.a]
     0000009329D7F8AC  3f800000 [li.b]
```
The `this` value for the current `LI` object is exactly 0000009329D7F8A8, which corresponds to the address of `a` in this particular case (this is not always the case). Parameters must be propagated to the procedure, but something unusual will occur. According to the fastcall scheme:
```
    RCX      RDX       R8       R9       STACK...
    XMM0     XMM1      XMM2     XMM3     STACK...

Get(float)
```
The first floating-point argument would normally go to XMM0. However, with member functions, this is not the case. The `this` pointer counts as a parameter:
```
    RCX      RDX       R8       R9       STACK...
    XMM0     XMM1      XMM2     XMM3     STACK...

Get(this,    float)
```
Looking at the instructions at the RIP address: parameters are still set up from right to left — `t` goes to XMM1 first, then `this` (computed using the `lea` instruction) into RCX. The `call` instruction pushes the return address onto the stack and transfers control to `Get`:
```
    0000009329D7F874  00000000
RSP 0000009329D7F878 |e0301060| [ret_main]
    0000009329D7F87C |00007ff7| [ret_main]
    0000009329D7F880  29d7f8a8
    0000009329D7F884  00000093
    0000009329D7F888  e03014c9
```
```asm
RIP 00007FF7E0301000  movss  dword ptr [rsp+10h],xmm1
    00007FF7E0301006  mov    qword ptr [rsp+8],rcx
    00007FF7E030100B  mov    rax,qword ptr [rsp+8]
    00007FF7E0301010  mov    rcx,qword ptr [rsp+8]
    00007FF7E0301015  movss  xmm0,dword ptr [rax+4]
    00007FF7E030101A  subss  xmm0,dword ptr [rcx]
    00007FF7E030101E  movss  xmm1,dword ptr [rsp+10h]
    00007FF7E0301024  mulss  xmm1,xmm0
    00007FF7E0301028  movaps xmm0,xmm1
    00007FF7E030102B  mov    rax,qword ptr [rsp+8]
    00007FF7E0301030  movss  xmm1,dword ptr [rax]
    00007FF7E0301034  addss  xmm1,xmm0
    00007FF7E0301038  movaps xmm0,xmm1
    00007FF7E030103B  ret
```
The Assembly code above illustrates how a single copy of the `Get` function in memory can serve all objects of the class. The `this` pointer makes this possible by directing the function to the correct member variables. We will revisit this topic in the object memory layout chapter.

The code uses the object address from RCX to read `li.a` and `li.b`, and performs the interpolation using the `t` parameter from XMM1. The result is stored in XMM0 for the caller to retrieve. The function does not allocate or free stack space, as RSP is not modified in this procedure. The function returns to the caller with `ret` as usual:
```
    0000009329D7F874  00000000
    0000009329D7F878  e0301060  [ret_main]
    0000009329D7F87C  00007ff7  [ret_main]
RSP 0000009329D7F880  29d7f8a8
    0000009329D7F884  00000093
    0000009329D7F888  e03014c9
```
```asm
    00007FF7E0301040  sub   rsp,38h
    00007FF7E0301044  lea   rcx,[rsp+28h]
    00007FF7E0301049  call  00007FF7E0301070
    00007FF7E030104E  movss xmm1,dword ptr [00007FF7E0302DD0h]
    00007FF7E0301056  lea   rcx,[rsp+28h]
    00007FF7E030105B  call  00007FF7E0301000
RIP 00007FF7E0301060  movss dword ptr [rsp+20h],xmm0
    00007FF7E0301066  xor   eax,eax
    00007FF7E0301068  add   rsp,38h
    00007FF7E030106C  ret
```
The `main` procedure copies the result from XMM0 to RSP+32, which corresponds to `x`. After releasing the previously allocated 56 bytes, it also returns with `ret`.

###### 64-BIT VECTORCALL EXAMPLE 1
`vectorcall` extends `fastcall` to take advantage of additional registers. This convention is available only if Streaming SIMD Extensions 2 (SSE2) or above is supported. Parameter-passing rules are similar to `fastcall`: the same registers `RCX` `RDX` `R8` `R9` are used for integer parameters, but this time up to 6 SSE registers XMM0…XMM5 can be used for both floating-point and vector values. Vector values can therefore be copied into registers rather than passed by address. Due to the higher number of available registers, the position-based mapping is slightly different:
```
     RCX      RDX      R8       R9          STACK    STACK    STACK...
     XMM0     XMM1     XMM2     XMM3        XMM4     XMM5     STACK...
 
```
```
     ECX      EDX
Func(int,     int)

     RCX      EDX
Func(int*,    enum)

     ECX      XMM1     R8D
Func(int,     float,   int)

     XMM0     XMM1     R8D      R9D
Func(float,   float,   int,     int)

     ECX      XMM1     R8D      XMM3        STACK
Func(int,     float,   int,     float,      int)

     RCX      XMM1     R8       XMM3        STACK
Func(__m128&, float,   __m256*, float,      int)

     RCX      XMM1     R8D      XMM3        STACK
Func(char&,   float,   int,     double,     int)

     ECX      EDX      XMM2     XMM3        XMM4     STACK
Func(char,    int,     double,  float,      double,  char)

     XMM0     EDX      R8D      YMM3        XMM4     YMM5
Func(float,   int,     char,    __m256,     double,  __m256)

     RCX      XMM1     XMM2     R9D         STACK    STACK
Func(float*,  float,   __m128,  char,       int,     int)

     RCX      RDX      XMM2     YMM3        STACK    STACK
Func(Data*,   Data&,   __m128,  __m256,     int*,    unsigned char)

     ECX      EDX      XMM2     R9          XMM4
Func(char,    char,    float,   const int&, __m128)

      RCX     RDX      XMM2     XMM3
Func(__int64, int[][4],float,   float)

     XMM0     EDX      R8D      R9          STACK    XMM5
Func(__m128,  int,     char,    __int64,    __int32, float)

     ECX      EDX      YMM2     R9D         STACK 
Func(int,     int,     __m256,  char8_t,    int[][4])
```
The same rules apply for 8-bit and 16-bit literals. Literal parameters can be encoded directly into instructions and use a smaller register; otherwise, they are zero-extended to 32-bit integers. The volatile and non-volatile register lists are the same in both calling conventions.

I will focus only on parameter passing, not on what the function is computing:
```cpp
float __vectorcall Test(char c, int i, __m128 v1, __m256 v2, double d)
{
  return (float)c + (float)i + v1.m128_f32[0] + v1.m128_f32[0] + (float)d;
}

__m128 v1 = {1.0f, 2.0f, 3.0f ,4.0f};
__m256 v2 = {5.0f, 6.0f, 7.0f, 8.0f, 8.0f, 7.0f, 6.0f, 5.0f};

auto r = Test('a', 1, v1, v2, 0.5);
```
According to the table, parameters should be propagated as follows:
```
               CL  EDX  XMM2 YMM3 XMM4      
auto r = Test('a', 1,   v1,  v2,  0.5);
```
Assembly instructions for the call:
```
00007FF74BB82001  movsd   xmm4,mmword ptr [00007FF74BB8BD20h]  
00007FF74BB82009  vmovups ymm3,ymmword ptr [rbp+40h]  
00007FF74BB8200E  movaps  xmm2,xmmword ptr [rbp+10h]  
00007FF74BB82012  mov     edx,1  
00007FF74BB82017  mov     cl,61h  
00007FF74BB82019  call    00007FF74BB814C9 
```

##### STACK SHADOW POINTER
Since we are discussing calling conventions, let us also mention the SSP (stack shadow pointer) register and its relevance to the current topic. During the introduction, while discussing the decision to disable optimization and the stack security check, I mentioned the ability of malicious programs to overwrite the return address by exploiting stack overflows. Several mechanisms exist to prevent this. One of them is called CET (Control-flow Enforcement Technology). CET is available if the underlying CPU implements such mechanisms; otherwise, it is ignored. You can try to enable it with the option `CET Shadow Stack Compatible`, found here:
```
Properties > Linker > Advanced
```
If enabled and supported, whenever the CPU executes a `call` instruction, the same return address is also pushed onto a separate stack (called the shadow stack) residing in a dedicated region of memory. The shadow stack contains only return addresses stacked on top of each other — no variables or other data. During a `ret` instruction, the CPU retrieves the return address from the shadow stack as well, and if it does not match the one from the program stack, an exception is generated, and the system may decide to terminate the process for security reasons.
If your CPU supports this protection mechanism and you have enabled it, you can inspect the shadow stack by reading its address from the `SSP` register:
```
SSP 000000C281DFEFD8  00007ff7d0c810a1
    000000C281DFEFE0  00007fff5a5b2dad
    000000C281DFEFE8  00007ff7d0c811c4
    000000C281DFEFF0  00007fff5b8054e0
    000000C281DFEFF8  00007fff5cfe485b
    ....
    ....
```
The currently executing function should return to 00007ff7d0c810a1 once it terminates. The CPU will raise an exception if this value does not match the one retrieved from the program stack.
