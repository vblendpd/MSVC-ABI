> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### ONE DEFINITION RULE
The One Definition Rule (ODR) is one of the most well-known principles in C++. Only one definition of an entity (function, class, variable, etc.) may exist in a translation unit for a given scope. A translation unit is a source file (after the preprocessor stage) that will produce an object file during compilation. Multiple declarations are allowed, but only one definition is permitted. The language standard does not require the compiler to diagnose the violation; in that case, the program behavior is undefined. Most tools, however, report compilation or linking errors. The first example we will examine is a direct ODR violation caused by defining the same function twice:
```C++
//Util.h
int GetValue(){ return 1; }
```
```C++
//main.cpp -> main.obj
#include "Util.h"

int GetValue(){ return 2; }

void main()
{
  int value = GetValue();
}
```
Compiling the previous code will produce an error:
```
error C2084: function 'int GetValue(void)' already has a body
```
Both functions attempt to coexist in the same translation unit within the global scope. Since this is not a function overloading case, the compiler does not know which one to use for the call. Functions do not hide previous definitions the way variables do at different scopes:
```C++
int x = 10;

void Func(int x = 3)
{
  int a = x * 2; //uses x = 3
}
```
The problem extends beyond multiple definitions of the same symbol in a single translation unit. The same symbol cannot exist in different translation units at the same scope either. The toolset will defer the error to the linking stage:
```C++
//Util.h
int GetValue(){ return 1; }
```
```C++
//main.cpp -> main.obj
#include "Util.h"

void main()
{
  int x = GetValue();
}
```
```C++
//source.cpp -> source.obj
int GetValue()
{
  return 2;
}
```
```
error LNK2005: "int __cdecl Function(void)" (?Function@@YAHXZ) already defined in main.obj
```
From a compilation standpoint, everything is correct: main.cpp compiles to main.obj containing `Util.h:GetValue`. Similarly, source.cpp compiles to source.obj with its own version `source.cpp:GetValue`. Each translation unit generated an object file containing different binary code for different `GetValue` functions, but both use the same mangled name (name mangling is covered in another chapter). The linker makes no assumptions about which one to place in the executable, despite it being evident that `main` uses `Util:GetValue` and `source:GetValue` is not used at all.

So far, these `GetValue` functions are causing problems because they have been assigned the same mangled name, despite generating different machine code (they are physically distinct procedures). Let us see what happens if we use the same function in multiple translation units; in theory, this function should generate a unique object code shared across many translation units. Commonly, source files share functionality through header files:
```C++
//Util.h
int GetValue(){ return 1; }
```
```C++
//source.cpp
#include "Util.h"

int Func(){ return GetValue()+1; }
```
```C++
//main.cpp
#include "Util.h"

int Func();

void main()
{
    int x = GetValue();
    int y = Func();
}
```
`GetValue` this time appears to be unique for the whole program; no other declaration or definition of `GetValue` exists apart from the one in Util.h. Being a definition as well as a declaration, `GetValue` will produce object code in every unit that includes it. Each unit must generate a symbol name, bringing back the ODR issue at link time.

It is possible to solve all the problems encountered so far. For duplicated symbol definitions in the same unit, it is sufficient to enclose one of them in a namespace. Two preconditions apply: the namespace cannot be unnamed, and its contents must not be exposed with `using namespace...` at the same scope that collides with the other symbol definition:
```C++
//Util.h
int GetValue(){ return 1; }
```
```C++
//main.cpp
#include "Util.h"

namespace nn {
  //no collision with ::GetValue
  int GetValue(){ return 2; }
}

void main()
{
  int value1 = GetValue();     //call Util.h:GetValue
  int value2 = nn::GetValue(); //call the nn namespace one
}
```
There are several ways to make the same function available across many translation units. A simple solution is to move the definition to a separate source file, leaving only its declaration in Util.h:
```C++
//Util.h
int GetValue();
```
```C++
//Util.cpp -> Util.obj
int GetValue(){ return 1; }
```
```C++
//source.cpp -> source.obj
#include "Util.h"

int Func(){ return GetValue()+1; }
```
```C++
//main.cpp -> main.obj
#include "Util.h"

int Func();

void main()
{
  int x = GetValue();
  int y = Func();
}
```
In this case, even though both source.cpp and main.cpp include Util.h, there will be no ODR violation, because a declaration does not generate object code or a symbol name. The function declaration simply informs the unit that the function exists somewhere. It is up to the linker to locate the function later (in this case, in Util.obj at link time) and establish the call between `Func` and `main`. This works because functions have external linkage by default, which will be covered in more detail shortly.

A better approach is to mark the function `inline`, which informs the linker that when it encounters multiple definitions of the same symbol, it can pick any one of them because they are all identical. For example:
```C++
//Util.h -> Util.obj
inline int GetValue(){ return 1; }
```
```C++
//source.cpp -> source.obj
#include "Util.h"

int Func(){ return GetValue()+1; }
``` 
```C++
//main.cpp -> main.obj
#include "Util.h"

int Func();

void main()
{
  std::cout << GetValue() << " " << Func() << std::endl; //print "1 2" 
}
```
Stepping into the disassembly, you can locate both calls and observe that they resolve to the same function, since they translate to the same address:
```
source.cpp:Func:GetValue -> call 07FF7A1211780h
main.cpp:GetValue        -> call 07FF7A1211780h
```
There is a distinction between functions inlined by compiler optimization and `inline` variables. An inline variable is a feature that can resolve ODR issues. Inline variables and functions have nothing to do with the compiler optimization that replaces a function call with its machine code. Remember that we have optimization disabled, so a function will not be substituted with its instructions (we would not be able to see any `call` instructions otherwise).

Marking a function `static` resolves the same issue. It tells the toolset that the function should be made local to a translation unit and not visible to others.
```C++
//Util.h
static int GetValue(){ return 1; }
```
```C++
//source.cpp
#include "Util.h"

int Func(){ return GetValue()+1; }
``` 
```c
//main.cpp
#include "Util.h"

int Func();

void main()
{
  std::cout << GetValue() << " " << Func() << std::endl; //print "1 2"
}
```
What is the difference from the `inline` case? Since the function is made local to each translation unit, each unit must produce its own machine code. We will have N copies of the same function (2 in our example) embedded in the executable. Every call to `GetValue` within a given translation unit will use its local copy. Browsing the Assembly code again, you will find a different call address this time:
```
source.cpp:Func:GetValue -> call 07FF688AC1460h
main.cpp:GetValue        -> call 07FF688AC1470h
```
The `static` keyword causes this kind of duplication-isolation for functions, and it can also help with the problem of duplicate function names at the same scope across different translation units.

The fundamental issue is that global symbols (functions and variables) have external linkage by default. External linkage means they are accessible from anywhere in the program. Consider what happens when a function gets compiled into machine code: the code resides somewhere in memory, and any part of the program can invoke that procedure if it knows the signature and address. Similarly, global variables reside in the `.data` section and are accessible to any function that knows they exist. You can control a symbol's linkage manually by marking it `extern` or `static`. Once a function is declared `static`, its linkage switches to internal, making it visible only to the current translation unit. The linker will disregard other symbols from different object files that share the same name:
```C++
//A.cpp -> A.obj
static int GetValue(){ return 1; }

int FA()
{
  return GetValue(); //use A.cpp::GetValue and ignore others
}
```
```C++
//B.cpp -> B.obj
static int GetValue(){ return 2; }

int FB()
{
  return GetValue(); //use B.cpp::GetValue and ignore others
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl;  //print "1 2"
}
```
`A.cpp:GetValue` and `B.cpp:GetValue` still share the same mangled names, but neither violates ODR. At the Assembly level, the calls again resolve to different addresses because they are separate functions:
```
A.cpp:GetValue -> call 07FF6E3E11780h
B.cpp:GetValue -> call 07FF6E3E117A0h
```
In general, it is sufficient to mark N-1 functions `static` rather than all of them. Marking only the first one in A.cpp would be enough, since there are only two units attempting to generate the same symbol.

A symbol table is generated for each compilation unit and each object file. To display the symbol table, open `Visual Studio Command Prompt` from `Tools`, navigate to the directory where the .obj files are stored, and type:
```
dumpbin /symbols A.obj
```
> Release builds use `Whole Program Optimization (/GL)` by default. Object files produced with the /GL option are not compatible with the DUMPBIN utility. Either use a Debug build to inspect the symbol table, or disable `Whole Program Optimization` in:
```
Properties > C/C++ > Optimization
```
The table lists the function linkage, among many other things. The default linkage is external; for a non-static `GetValue`, the corresponding entry looks like this:
```
00D 00000000 SECT4  notype () External | ?GetValue@@YAHXZ (int __cdecl GetValue(void))
```
For a `static` `GetValue`, the linkage changes to `Static`:
```
00D 00000000 SECT4  notype () Static   | ?GetValue@@YAHXZ (int __cdecl GetValue(void))
```
`Static` means that the current translation unit will use its local copy of the function, ignoring the same symbol from other translation units, and symbol collision is resolved.

There is an alternative to the `static` keyword. Functions enclosed in an anonymous namespace also have internal linkage. Since the namespace provides no name, other translation units have no way to access it:
```C++
//A.cpp -> A.obj
namespace {
  int GetValue(){ return 1; }
}

int FA()
{
  return GetValue(); //use A.cpp::GetValue
}
```
```C++
//B.cpp
static int GetValue(){ return 2; }

int FB()
{
  return GetValue(); //use B.cpp::GetValue since it's the only one it can see
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl;  //print "1 2"
}
```
This produces the same result for both A.obj and B.obj:
```
notype () Static | ?GetValue@?A0x13152905@@YAHXZ (int __cdecl `anonymous namespace'::GetValue(void))
```
Anonymous namespaces that expose symbols at the same level the namespace is defined can lead to symbol ambiguity:
```C++
int x = 10;

namespace
{
  int x = 22; //ambiguous but perhaps OK
}

void main()
{
  std::cout << ::x << std::endl; //explicitly access the global namespace 'x' and print "10"
}
```
```C++
int x = 10;

namespace
{
  int x = 22; //ambiguous but perhaps OK
}

void main()
{
  std::cout << x << std::endl; //error: ambiguous symbol, which 'x' — first or second?
}
```
Variables are subject to the same rules. A global variable has external linkage by default:
```C++
//Value.h
int value = 10;
```
```C++
//A.cpp -> A.obj
#include "Value.h"

int FA()
{
  return value;
}
```
```C++
//B.cpp -> B.obj
#include "Value.h"

int FB()
{
  return value;
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl;
}
```
The same symbol exists in both A.obj and B.obj:
```
error LNK2005: "int value" (?value@@3HA) already defined in A.obj
```
`inline` variables have external linkage. The `inline` specifier tells the linker to select one shared definition because all occurrences refer to the same variable:
```C++
//Value.h
inline int value = 10;
```
```C++
//A.cpp -> A.obj
#include "Value.h"

int FA()
{
  return value; //use Value.h::value
}
```
```C++
//B.cpp -> B.obj
#include "Value.h"

int FB()
{
  return value; //use Value.h::value
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl; //print "10 10"
}
```
`FA` and `FB` retrieve the 32-bit integer from the same address in the data section:
```
00007FF60A521450  mov eax,dword ptr [07FF60A529000h]  
00007FF60A521456  ret  
```
```
00007FF60A521460  mov eax,dword ptr [07FF60A529000h]  
00007FF60A521466  ret  
```
The symbol table:
```
A.obj:
00C 00000000 SECT4  notype External | ?value@@3HA (int value)

B.obj:
00C 00000000 SECT4  notype External | ?value@@3HA (int value)
```
`const` variables, by contrast, have internal linkage. `static` has the same effect on a variable as on a function — it makes the variable local to the current translation unit, creating a copy. This can be a source of subtle errors:
```C++
//Value.h
static int value = 10;
```
```C++
//A.cpp -> A.obj
#include "Value.h"

int FA()
{
  return value; //return A.obj::value
}
```
```C++
//B.cpp -> B.obj
#include "Value.h"

int FB()
{
  value++;      //increment B.obj::value
  return value; //return B.obj::value
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl; //print "10 11"
}
```
The executable contains two integers stored at different addresses in the data section. Nothing is inherently wrong with this scenario, and no error or warning will be produced since we explicitly requested this behavior.
You may only notice you are incrementing a copy if you are paying close attention. If the variable is intended to be non-modifiable, you can mark it `const static`. It will still be a copy, but cannot be modified, giving the illusion of sharing the same variable. Dumping the symbol table reveals the same `Static` property as for static functions:
```
00C 00000000 SECT4  notype Static | ?value@@3HA (int value)
```
`const` variables also have internal linkage, and `const` can additionally trigger optimization. In our case, the value 10 can be encoded directly in a `MOV` instruction as an immediate operand, in which case the symbol will be absent from the symbol table:
```
FA:
00007FF674331450  mov eax, 0Ah
00007FF674331455  ret

FB:
00007FF674331460  mov eax, 0Ah
00007FF674331465  ret
```
>If you cannot find the symbol because the `const` variable was reduced to an immediate operand, try adding `volatile` to prevent the compiler from doing so.

For small types, the `const` value may therefore become an immediate operand. For larger types, `const` behaves the same as `static`, and the variable will be duplicated and non-modifiable in each translation unit that uses it:
```C++
//Util.h
struct Vector
{
  int x = 1;
  int y = 2;
};

const Vector vec;
```
```C++
//A.cpp -> A.obj
#include "Util.h"

int FA()
{               //cannot alter x or y
  return vec.x; //return Vector::x A.obj copy
}
```
```C++
//B.cpp -> B.obj
#include "Util.h"

int FB()
{               //cannot alter x or y
  return vec.x; //return Vector::x B.obj copy
}
```
```C++
//main.cpp -> main.obj
int FA();  //defined in another compilation unit
int FB();  //defined in another compilation unit

void main()
{
  std::cout << FA() << " " << FB() << std::endl; //print "1 1"
}
```
`FA` and `FB` read the 32-bit integer from different locations, confirming the duplicated `Vector`:
```
FA:
00007FF60C8E1450  mov eax, dword ptr [07FF60C8E7D58h]
00007FF60C8E1456  ret

FB:
00007FF60C8E1460  mov eax, dword ptr [07FF60C8E7BC8h]
00007FF60C8E1466  ret
```
For both translation units the symbol is `Static` as expected:
```
009 00000000 SECT3  notype  Static | ?vec@@3UVector@@B (struct Vector const vec)
```
To summarize: `static` and `const` both produce internal linkage, except that `const` additionally makes the variable non-modifiable. Prefer `inline` when sharing the same variable definition across translation units.



###### constexpr
`constexpr` stands for constant expression — a value or computation that should be evaluated at compile time rather than deferred to run time. This specifier can be applied to variables and functions (including constructors) to indicate that the value can be computed at compile time wherever possible. For variables, `constexpr` implies `const` and has internal linkage by default.

Prior to C++17, `constexpr` implied only `const`; since C++17, `inline` semantics were also introduced for `constexpr` variables, allowing a single definition to be shared across translation units without ODR violations.

A `constexpr` function must accept and return only literal types, can be recursive, cannot use `try`/`catch` blocks or `goto`, but can use `if`/`else` and other standard C++ constructs.




```C++
//A.cpp -> A.obj
constexpr int global_x = 1050;

int Get()
{
    return global_x;
}
```
```asm
...
...
mov  eax, 41Ah
...
```
The compiler translated `global_x` to the immediate value 0x41a, and no allocation took place in the `.data` section or anywhere else. The variable does not exist as a memory object; there is no symbol by which it can be accessed from the current or other translation units. As a result, you will not find the symbol `?global_x@@3AH` if you run dumpbin on A.obj's symbols. For the same reason, attempting to access the variable from another translation unit will yield a compilation error:
```C++
//B.cpp -> B.obj
extern int global_x;

int AnotherGet()
{
  return global_x;
}
```
```
error LNK2001: unresolved external symbol "int global_x" (?global_x@@3HA)
```
Using `extern constexpr int global_x;` will not work either, because an initializer must be present for `const` variables. This is the same reason why `extern const int x;` will not work if `const int x = 10;` was defined in another translation unit. In addition, `constexpr` is not interchangeable with `const` and cannot be used as a drop-in replacement. There is, however, a compiler option that allows `constexpr` functions defined in other translation units to be accessed externally:
```
/Zc:externConstexpr
```
Marking an entity `constexpr` does not automatically make other entities connected to it `constexpr` — at least not always. In the following case, the compiler will likely optimize the `Value()` call to an `inc` instruction, but this is left to the optimizer since the variable `v` is marked neither `const` nor `constexpr`:
```C++
constexpr int Value(int x)
{
  return x + 1;
}

void Func()
{
  int x = 0;
  std::cin >> x;

  int v = Value(x);
}
```
Sometimes the `constexpr` specifier may be treated as a hint that the compiler can choose to ignore:
```C++
constexpr int Value(int x)
{
  return x + 1;
}

void main()
{
  using FPTR = int(*)(int);
  FPTR fptr = Value;

  constexpr int x = fptr(10);

  cout << x << endl;
}
```
The dumpbin utility shows:
```
0B2 00000000 SECT6  notype () External | ?Value@@YAHH@Z (int __cdecl Value(int))
```
Something curious happens here. You can access this function from another translation unit as if it were not `constexpr`:

```C++
//B.cpp -> B.obj
extern int Value(int x);

int GetXb(int x)
{
  return Value(x);
}
```
There is another surprise: the call to `Value` is absent when accessing the function as though it were `constexpr` again:
```C++
//A.cpp ->A.obj
constexpr int Value(int x)
{
  return x + 1;
}

int GetXb();

void main()
{
  using FPTR = int(*)(int);
  FPTR fptr = Value;

  int x = fptr(1);

  cout << x << " " << GetXb(2) << endl;
}
```

```asm
mov  edx, 1  ;fptr(1)
call ...

mov  edx, 3  ;GetXb(2)
call ...
```
Because `constexpr` carries the `const` qualifier, and `const` variables have internal linkage, the same name can exist in many translation units:
```C++
//A.cpp -> A.obj
constexpr int global_x = 1050;

int GetXa(){ return global_x; }
```
```C++
//B.cpp -> B.obj
constexpr int global_x = 100;

int GetXb(){ return global_x; }

```
```C++
//main.cpp
int GetXa();
int GetXb();

void main()
{
    std::cout << GetXa() << " " << GetXb() << std::endl; //print "1050 100"
}
```
In this case, the compiler will likely have generated the following:
```asm
mov  edx, 41ah
call ...
...
...
mov  edx, 64h
call ...
...
...
```
Instead of duplicating the definition across different translation units, you could achieve the same behavior by placing a single definition in a shared header file.

You can use `static_assert` to verify that an expression can be evaluated at compile time:

```C++
constexpr int Value(int x)
{
  return x+1;
}

void main()
{
  int x = Value(1);
  static_assert(Value(1) == 2); //OK
  static_assert(x ==2);         //Error
}
```
