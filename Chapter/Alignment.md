> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### DATA ALIGNMENT AND PACKING
Throughout the preceding chapter we've seen what a type size is and why it's important. This chapter aims to explain how size and alignment are related.\
Data is aligned to N-byte boundary whenever it starts in memory from an address evenly divisible by N. Each type in C++ has an alignment requirement called natural alignment. This requirement restricts the list of addresses where the same type can be stored. For types that require 2-byte alignment, the address must be evenly divisible by 2 (the address terminates with a multiple of 2); for types that require 8-byte alignment, the address must be evenly divisible by 8 (the address terminates with a multiple of 8) and so on. To give an example, the natural alignment of a 32-bit integer is 4-byte, so any address terminating with `0`, `4`, `8` `C` is a valid location. You can check the last digits of any memory address to figure out what type can be stored at that location. Given the address `00000050AAA0011A` you can realize `A` or `1A` is multiple of 1 and 2 (but not multiple of 4 or 8). This condition only allows a 8-bit or 16-bit type to be stored at that location. You can manually calculate it this way:
```
A = 10

10 / sizeof(char)      = 10   //OK for 8-bit types
10 / sizeof(short)     = 5    //OK for 16-bit types
10 / sizeof(int)       = 2.5  //not for 32-bit types
10 / sizeof(long long) = 1.25 //not for 64-bit types
```
Alternatively you can translate the previous to a simple function:
```cpp
bool IsAligned(const void* address, unsigned long long n)
{
  return ((unsigned long long)address) % n == 0;
}
```
You can derive other alignment for any type. For example, data that requires a base address multiple of 128-byte should be stored at any address ending with a multiple of 128. Since 128 = 80h, any address terminating with a multiple of 80h (ending with `80` or `00`) will be suitable for your data:
```
80h*1 = 080h = 128
80h*2 = 100h = 256
80h*3 = 180h = 384
80h*4 = 200h = 512
80h*5 = 280h = 640
80h*6 = 300h = 768
80h*7 = 380h = 896
80h*8 = 400h = 1024
```
The alignment is not an intrinsic language requirement; it rather concerns the underlying hardware architecture developed in such a way to work more efficiently if data get aligned to its natural alignment. Unless otherwise specified with special directives, the default alignment for a built-in type is equal to its size. For aggregates, the alignment is equal to the largest built-in type used. Since built-in types size are powers of 2, and the same types are used as building blocks for user-defined types, we can conclude that for structures and classes, the alignment is always a positive power of 2 multiple than size of the largest member inside. The compiler will insert padding bytes between types or at the end of structures to make alignment consistent whenever required. Padding bytes are just unused bytes.

Let's try to calculate the following structure size. Note that from now on, I will only report the last byte whenever I mention an address to avoid writing the entire 64-bit address.
```cpp
struct Data
{
  int  a = 1;
  char b = 0xBB;
  int  c = 2;
};
```
A first guess could be sizeof(int)+sizeof(char)+sizeof(int) = 9. It's logically correct but it doesn't keep into consideration the alignment. Member variables lie in memory in the same order they're declared (in another chapter we'll cover object memory layout). Let's take a random 4-byte aligned start address to place the previous structure in memory:
```
000000209AA9FBB4  ffcc0000
000000209AA9FBB8 |00000001| [a]
000000209AA9FBBC  ......bb  [b]
000000209AA9FBC0  ........
```
The first integer takes 4 bytes starting at B8, which is fine because multiple of 4. The next location to write the char is BC. Since it takes 1 byte, any address is suitable. The next available address for the second integer is BD but not a multiple of 4. Bytes for the second integer cannot be stored like this:
```
000000209AA9FBB8  ........
000000209AA9FBBC  000002..
000000209AA9FBC0  ......00
```
The compiler needs to climb over some bytes to find the next 4-byte aligned address, which turns out to be C0. Thus 3 bytes in `BD` `BE` `BF` are left unused:
```
000000209AA9FBB4  ffcc0000
000000209AA9FBB8 |00000001| [a]
000000209AA9FBBC |******BB| [b]
000000209AA9FBC0 |00000002| [c]
000000209AA9FBC4  1a1acccc
```
I highlight padding bytes with `*`. Although the total size of Data seems to be 9, it's 12 instead, and the following structure is equivalent to the original one:
```cpp
struct Data
{
  int  a = 1;
  char b = 0xBB;
  char padding[3];
  int  c = 3;
};
```
Accessing padding locations doesn't affecting the code that uses Data. Padding bytes are there even if not explicitly declared:
```cpp
struct Data
{
  int  a = 1;
  char b = 0xBB;
  int  c = 3;
};

Data data;

//get the address of padding[0]
unsigned char* ptr = reinterpret_cast<unsigned char*>(&data) + sizeof(int) + sizeof(char);

//write the value 0xEE at that location
*ptr = 0xEE;
```
```
000000209AA9FBB4  ffcc0000
000000209AA9FBB8 |00000001| [a]
000000209AA9FBBC |****EEBB| [e][b]
000000209AA9FBC0 |00000002| [c]
000000209AA9FBC4  1a1acccc
```
Sometime padding bytes are placed at the end of structures. The next example belongs to that case:
```cpp
struct T
{
  double a = 1.0;
  char   b = 0xBB;
};
```
The alignment of T is 8 because the larger type inside is `double`. `a` goes first:
```
00000095A37BF9FC  00007ff7
00000095A37BFA00 |00000000| [a]
00000095A37BFA04 |3ff00000| [a]
00000095A37BFA08  ........
```
`b` can be placed at 08 and the remaining bytes from 09 to 0F are left unused:
```
00000095A37BF9FC  00007ff7
00000095A37BFA00 |00000000| [a]
00000095A37BFA04 |3ff00000| [a]
00000095A37BFA08 |******BB| [b]
00000095A37BFA0C  ccff0000
```
Everything is aligned to its natural boundary, makes us believe the size is 12 bytes. It's not, and creating an array of the same structure helps to highlight the issue:
```cpp
T arr[2];
```
```
000000F904DCF83C  00007ff6
000000F904DCF840 |00000000| arr[0].a
000000F904DCF844 |3ff00000| arr[0].a
000000F904DCF848 |******BB| arr[0].b
000000F904DCF84C |00000000| arr[1].a
000000F904DCF850 |3ff00000| arr[1].a
000000F904DCF854 |******BB| arr[1].b
000000F904DCF858  00000000
```
Everything looks fine except `arr[1].a`. It must be located at an address multiple of 8 since it appears to be a double floating-point value. The address terminating with 4C is not. The proper address where to write that value should be the one ending with 50. Moving the item to that address is equivalent to inserting padding bytes at the end of the first array element. Thus the actual structure is:
```C++
struct T
{
  double a = 1.0;
  char   b = 0xBB;
  char   padding[7];
};
```
```
000000F904DCF83C  00007ff6
000000F904DCF840 |00000000| arr[0].a
000000F904DCF844 |3ff00000| arr[0].a
000000F904DCF848 |******BB| arr[0].b
000000F904DCF84C |********|
000000F904DCF850 |00000000| arr[1].a
000000F904DCF854 |3ff00000| arr[1].a
000000F904DCF858 |******BB| arr[1].b
000000F904DCF85C |********|
000000F904DCF860  00000000
```
Every element in the array must have the same size and alignment so the padding happens at the end of the second element too. The alignment of the array is the same as the alignment of one of its elements which in turn is the alignment of the largest type used. The array size is also an integral multiple of the size of an element and a multiple of the alignment. In our example, the size and alignment are 16 and 8 respectively. Considering that a 32-bit integer must align to a 4-byte boundary, we can declare another equivalent structure:
```cpp
struct T
{
  double a = 1.0;
  char   b = 0xBB;
  int    padding;
};
```
According to the first example, 3 bytes are left unused between the char and the int. The integers take 4 bytes; thus 7 bytes of padding are also placed here.\
Another example:
```cpp
struct Data
{
  int    a = 1;
  char   b = 0xBB;
  int    c = 2;
  double d = 1.0;
  char   e = 0xEE;
  int    f = 3;
};
```
The largest type is `double`, hence the structure is aligned to 8 bytes boundary. We randomly establish the base address to be F0:
```
000000C3C047FAEC  9c002400
000000C3C047FAF0 |00000001| [a]
000000C3C047FAF4  ......bb| [b]
000000C3C047FAF8  ........
```
The integer goes first followed by the char. 3 bytes are then left unused to align the second integer to F8:
```
000000C3C047FAEC  9c002400
000000C3C047FAF0 |00000001| [a]
000000C3C047FAF4 |******bb| [b]
000000C3C047FAF8 |00000002| [c]
000000C3C047FAFC  ........
```
Since `d` cannot be stored at FC because not a multiple of 8, the next 4 bytes are padded, reaching 00 (which is a multiple of 8):
```
000000C3C047FAEC  9c002400
000000C3C047FAF0 |00000001| [a]
000000C3C047FAF4 |******bb| [b]
000000C3C047FAF8 |00000002| [c]
000000C3C047FAFC |********|
000000C3C047FB00 |00000000| [d]
000000C3C047FB04 |3ff00000| [d]
000000C3C047FB08  ........
```
The last char is fine to go at 08 followed by 3 bytes padding to place the last integer at 0C:
```
000000C3C047FAEC  9c002400
000000C3C047FAF0 |00000001| [a]
000000C3C047FAF4 |******bb| [b]
000000C3C047FAF8 |00000002| [c]
000000C3C047FAFC |********|
000000C3C047FB00 |00000000| [d]
000000C3C047FB04 |3ff00000| [d]
000000C3C047FB08 |******ee| [e]
000000C3C047FB0C |00000003| [f]
000000C3C047FB10  cc9dfa64
```
To be sure it ends here, you can again visualize a 2 elements array:
```
000000C3C047FAEC  9c002400
000000C3C047FAF0 |00000001| arr[0].a
000000C3C047FAF4 |******bb| arr[0].b
000000C3C047FAF8 |00000002| arr[0].c
000000C3C047FAFC |********|
000000C3C047FB00 |00000000| arr[0].d
000000C3C047FB04 |3ff00000| arr[0].d
000000C3C047FB08 |******ee| arr[0].e
000000C3C047FB0C |00000003| arr[0].f
000000C3C047FB10 |00000001| arr[1].a
000000C3C047FB14 |******bb| arr[1].b
000000C3C047FB18 |00000002| arr[1].c
000000C3C047FB1C |********|
000000C3C047FB20 |00000000| arr[1].d
000000C3C047FB24 |3ff00000| arr[1].d
000000C3C047FB28 |******ee| arr[1].e
000000C3C047FB2C |00000003| arr[1].f
000000C3C047FB30  00006ecd
```
Everything looks aligned to its natural size. The conclusion is that Data alignment and size are 8 and 32:
```cpp
struct Data
{
  int    a = 1;
  char   b = 0xBB;
  char   padding[3];
  int    c = 2;
  char   padding[4];
  double d = 1.0;
  char   e = 0xEE;
  char   padding[3];
  int    f = 3;
};
```
You can now realize that the size of user-define objects is affected by how the data is declared. Arranging data in different ways can save space. The previous structure can shrink its size to 24 bytes if rewritten as follow:
```cpp
struct Data
{
  double d = 1.0;
  int    a = 1;
  int    c = 2;
  int    f = 3;
  char   b = 0xBB;
  char   e = 0xEE;
};
```
```
000000C3C047FAEC  43c400fe
000000C3C047FAF0 |00000000| [d]
000000C3C047FAF4 |3FF00000| [d]
000000C3C047FAF8 |00000001| [a]
000000C3C047FAFC |00000002| [c]
000000C3C047FB00 |00000003| [f]
000000C3C047FB04 |****EEBB| [e][b]
000000C3C047FB08  34b83e6e
```
Unions are like structures but their fields overlap in memory. The union size is the size of the biggest type used, and since all the items share the same base address, the address must be suitable to align any of them. Example:
```cpp
union U1
{
  int    i = 97;
  char   c;
  double d;
};
```
```
0000009F592FFA04  cccccccc
0000009F592FFA08 |00000061| [d] [c] [i]
0000009F592FFA0C |cccccccc| [d]
0000009F592FFA10  cccccccc
```
The biggest type used is `double`. Its size and alignment are both 8, so it is for U1. Any currently used type, `i` or `c` or `d` will start at the same 8-byte aligned address.

Another example:
```cpp
struct Vector
{
  int x;
  int y;
};

union U2
{
  int    i;
  char   c[4];
  Vector v;
};

U2 u2;
u2.v.x = 1;
u2.v.y = 2;
```
```
000000BCBB3AF8B4  cccccccc
000000BCBB3AF8B8 |00000001| [v.x] c[3]c[2]c[1]c[0] [i]
000000BCBB3AF8BC |00000002| [v.y]
000000BCBB3AF8C0  cccccccc
```
The biggest size is Vector; hence U2 size and alignment are 8 and 4. All the items inside are lied-out in memory starting from the same base address, and they share the same space: &i equals &c[0] which equals &v.x. Changing the value of c[0] will overwrite the lower byte of v.x:
```
u.c[0] = 0xA;

000000BCBB3AF8B4  cccccccc
000000BCBB3AF8B8 |0000000A| [v.x] c[3]c[2]c[1]c[0] [i]
000000BCBB3AF8BC  00000002  [v.y]
000000BCBB3AF8C0  cccccccc
```
Yet another example:
```cpp
struct T
{
  char   c;
  double d;
};

union U3
{
  int i;
  T   t[2];
};

U3 u3;

u3.t[0].c = 'a';
u3.t[0].d = 1.0;
u3.t[1].c = 'b';
u3.t[1].d = 2.0;
```
```
00000085C4B7FE44  00007ff7
00000085C4B7FE48 |6d921d61| t[0].c        [i]
00000085C4B7FE4C |********| t[0].padding
00000085C4B7FE50 |00000000| t[0].d
00000085C4B7FE54 |3ff00000| t[0].d
00000085C4B7FE58 |00000062| t[1].c
00000085C4B7FE5C |********| t[1].padding
00000085C4B7FE60 |00000000| t[1].d
00000085C4B7FE64 |40000000| t[1].d
00000085C4B7FE68  1c85a207
```
The size and alignment for T are 16 and 8. The size and alignment for U3 are 32 and 8. Make sure to distinguish between the union member from the structure member used in the union. Union members overlap in memory and not the member of T. Let's look at this:
```cpp
struct T
{
  double d;
};

union U4
{
  double dd;
  T      t[2];
};

U4 u4;
u4.t[0].d = 1.0;
u4.t[1].d = 2.0;
```
The size and alignment of T are 8 and 8. The size and alignment of U4 are 16 and 8. What happens in memory:
```
000000F69E6FFE04  00000000
000000F69E6FFE08 |00000000| t[0].d [dd]
000000F69E6FFE0C |3ff00000| t[0].d [dd]
000000F69E6FFE10 |00000000| t[1].d
000000F69E6FFE14 |40000000| t[1].d
000000F69E6FFE18  95da2cc4
```
The union member `dd` and the first element of the array `t[0].d` share the same space. If we set the value for `dd` we overwrite `t[0].dd`. Let's see what happens if we add a `char` to T:
```cpp
struct T
{
  char   c;
  double d;
};

union U4
{
  double dd;
  T      t[2];
};

U4 u4;
u4.t[0].c = 'a';
u4.t[0].d = 1.0;
u4.t[1].c = 'b';
u4.t[1].d = 2.0;
```
You must be able to calculate the size and alignment for T and U4 that correspond to 16 and 8 for T, and 32 and 8 for U4. The memory layout will change to:
```
00000063DE79FB84  00007ff7
00000063DE79FB88 |******61| t[0].c       [dd]
00000063DE79FB8C |********| t[0].padding [dd]
00000063DE79FB90 |00000000| t[0].d
00000063DE79FB94 |3ff00000| t[0].d
00000063DE79FB98 |******62| t[1].c
00000063DE79FB9C |********| t[1].padding
00000063DE79FBA0 |00000000| t[1].d
00000063DE79FBA4 |40000000| t[1].d
00000063DE79FBA8  ac4bda2c
```
Changing the value for `dd` will overwrite `t[0].c` only, leaving `t[0].d` still valid:
```cpp
u3.dd = 1.0;
```
```
00000063DE79FB84  00007ff7
00000063DE79FB88 |00000000| t[0].c       [dd]
00000063DE79FB8C |3ff00000| t[0].padding [dd]
00000063DE79FB90 |00000000| t[0].d
00000063DE79FB94 |3ff00000| t[0].d
00000063DE79FB98 |******62| t[1].c
00000063DE79FB9C |********| t[1].padding
00000063DE79FBA0 |00000000| t[1].d
00000063DE79FBA4 |40000000| t[1].d
00000063DE79FBA8  ac4bda2c
```
The structure T still obeys to padding rules to align the `double` to an address multiple of 8. So remember that union members share the same space starting from the base address, but each built-in type or structure inside gets treated as before; no special rules unless we pack the data using directives as we will see shortly.

Data can be packed differently using the `pragma` directive. The rule works for structures, classes, and unions. Accepted values are `1` `2` `4` `8` `16`. The default value is 8 for 32-bit and 16 for 64-bit. The default value `Struct Member Alignment` can be changed in compiler settings:
```
Properties > C/C++ > Code Generation
```
Alternatively `/ZpN` (where N is one of the numbers above) will do the same work from the command line. The compiler will ignore any values greater than the maximum alignment for the current item or greater than the default platform alignment.

You can set packing criteria per-item or use a stack-based mechanism. The first method requires specifying the packing value at the beginning of the structure and restoring the default at the end unless you want the next structures to be affected too. The second method uses `push` and `pop` to set and restore packing values. Just pass `show` to the directive to display the current packing value in the output window. Suppose `Struct Member Alignment` is set to default, and we're building 64-bit:
```cpp
#pragma pack(2) //set packing to 2
struct { ... }; //use 2
#pragma pack()  //restore default packing

//print 16
#pragma pack(show)

struct {...} //use 16

//add 1
#pragma pack(push, 1)

struct {...}; //use 1

struct {...}; //use 1

//print 1
#pragma pack(show)

//add 4
#pragma pack(push, 4)

struct {...}; //use 4

struct {...}; //use 4

//restore the previous value from the stack
#pragma pack(pop)

struct {...}; //use 1

//restore the previous value from the stack
#pragma pack(pop)

struct {...}; //use 16
```
When used without the parameter, `pragma pack()` always reset to the default value clearing the stack. All the values added to the stack before that point are not available anymore unless you push some again.

The example below highlights the differences between default packing (on the left) with 2-byte and 1-byte packing:
```cpp
                                      #pragma pack(2)               #pragma pack(1)
       struct Data                    struct Data2                  struct Data1
       {                              {                             {
         char    a = 0xAA;              char   a = 0xAA;              char   a = 0xAA;
         short   b = 1;                 short  b = 1;                 short  b = 1;
         char    c = 0xBB;              char   c = 0xBB;              char   c = 0xBB;
         int     d = 2;                 int    d = 2;                 int    d = 2;
         char    e = 0xCC;              char   e = 0xCC;              char   e = 0xCC;
         double  f = 1.0;               double f = 1.0;               double f = 1.0;
       };                             };                            };
```
```
000000C3DB1EFB7C  00007ff6    000000C3DB1EFB7C  00007ff6    000000C3DB1EFB7C  00007ff6
000000C3DB1EFB80 |0001**AA|   000000C3DB1EFB80 |0001**AA|   000000C3DB1EFB80 |BB0001AA|
000000C3DB1EFB84 |******BB|   000000C3DB1EFB84 |0002**BB|   000000C3DB1EFB84 |00000002|
000000C3DB1EFB88 |00000002|   000000C3DB1EFB88 |**CC0000|   000000C3DB1EFB88 |000000CC|
000000C3DB1EFB8C |******CC|   000000C3DB1EFB8C |00000000|   000000C3DB1EFB8C |f0000000|
000000C3DB1EFB90 |00000000|   000000C3DB1EFB90 |3ff00000|   000000C3DB1EFB90  .....|3F|
000000C3DB1EFB94 |3ff00000|   000000C3DB1EFB94  00000011    000000C3DB1EFB94  00000011
000000C3DB1EFB98  2dfe2e0e    000000C3DB1EFB98  2dfe2e0e    000000C3DB1EFB98  2dfe2e0e
```
Data that uses default packing values, alignes items to their natural boundaries: 1-byte for char, 4-byte for int, etc. The alignment and size  are 8 and 24.

Data2 tells the compiler that members can be aligned to 2-byte boundaries. All built-in types inside can be stored at any memory address multiple of 2. Hence Data2 alignment and size are 2 and 20.

Similarly, Data1 uses a 1-byte alignment to write all values starting from any memory address. Tightly packed bytes waste no space for padding. Data1 alignment and size are 1 and 17.

The compiler will ignore any value greater than object alignment or greater than the default platform alignment. In short:
```cpp
#pragma pack(32)
struct Data
{
  char    a = 0xAA;
  short   b = 1;
  char    c = 0xBB;
  int     d = 2;
  char    e = 0xCC;
  double  f = 1.0;
};
```
32 is not valid. It's bigger than Data alignment (equal to 8 because of the `double`) and higher than 64-bit platform alignment (which is 16). The value is thus ignored, and the alignment and size of Data are still 8 and 24.

It's possible to align to a specific boundaries even bigger than natural alignment using `alignas` or Microsoft-specific `__declspec(align)` directive. C++ used to have weaknesses in providing specific features. Compilers have implemented their non-portable language extensions. The alignment directive was one of that. Since the language didn't provide any mechanisms to query/specify alignment, Microsoft and GCC introduced `__declspec` and `__attribute__` respectively.

These specifiers don't produce object code, so there is no run-time overhead, but data will eventually waste a lot of space for padding if not used carefully. Alignment is only possible to the power of 2: `1` `2` `4` `8` `16` `32` etc., and can be applied to structure, classes, unions, enumerators, arrays, and single built-in types, per-object and for individual members. The compiler will ignore the alignment if smaller than the default alignment for the current item:
```cpp
//ignored because default alignment is 4 (int)
struct alignas(2) Test
{
  int  x = 1;
  char c = 0xCC;
  int  y = 2;
};

//ignored because default alignment is 8 (double)
struct alignas(4) Test
{
  int    x = 1;
  char   c = 0xCC;
  double d = 1.0;
  int    y = 2;
};

//ignored because less than default 4
struct alignas(2) Test
{
  int x = 0;
  int y = 0;
};

//OK, but useless in this case
alignas(4) int x = 1;

//ignored because less than default 4
alignas(2) int y = 1;

//align to 8 bytes boundary
alignas(long long) int z = 0;

//ignored because the default is 8
alignas(4) double = 1.0;

//OK will store the char at an address multiple of 16
__declspec(align(16)) char c = 0xCC;

//the underlying type will be stored at an address multiple of 8
enum class alignas(8) Color{...};

//align to 2 bytes boundary
struct alignas(__int16) AAA
{
  char c = 'c';
};

//ignored because less than default 8
struct alignas(__int8) BBB
{
  char   c = 'c';
  double d = 1.0;
};
```
Microsoft compiler reports a warning whenever alignment gets ignored. Some compilers don't diagnose the problem at all.\
Alignment can be specified for individual members, and rules remain the same. The XY structure below has alignment and sizes 8 and 16:
```cpp
struct XY
{
  alignas(8) int  i = 1;
  alignas(8) char c = 0xAA;
};
```
```
0000002CADB3F75C  00007ff7
0000002CADB3F760 |00000001| [i]
0000002CADB3F764 |********|
0000002CADB3F768 |******aa| [c]
0000002CADB3F76C |********|
0000002CADB3F770  e0f7a5e3
```
Same rules for compositions:
```cpp
//OK align i to 8-byte boundary
struct alignas(8) i32
{
  int i = 1;
};

//ignored because 4 is less than default (which is equal to 8 from i32)
struct alignas(4) XY
{
  i32  a;
  int  i = 1;
  char c = 0xCC;
};
```
The following structures are all equivalent:
```cpp
struct alignas(8) A
{
  int   arr[4];
  float f;
};

struct B
{
  alignas(8) int arr[4];
  float f;
};

__declspec(align(8)) struct C
{
  int   arr[4];
  float f;
};

struct D
{
  __declspec(align(8)) int arr[4];
  float f;
};
```
Using a bigger alignment than the default one may waste a lot of space. For example:
```cpp
struct alignas(16) Test
{
  int x = 4;
};
```
Test will always be constructed in memory starting from an address multiple of 16. The structure itself seems harmless because `alignas` is telling the compiler to align its only member to an address multiple of 16. To understand the issue you need to visualize an array of Test. Each `x` for every array element will align to a 16-byte address wasting 12 bytes:
```
000000123289FA3C  00007ff6
000000123289FA40 |00000004| arr[0].x
000000123289FA44 |********|
000000123289FA48 |********|
000000123289FA4C |********|
000000123289FA50 |00000004| arr[1].x
000000123289FA54 |********|
000000123289FA58 |********|
000000123289FA5C |********|
000000123289FA60  ccccffcc
```
This occurs whenever the array is a local function variable, a member of structures, or even when stored in the binary data section.

A combination of `pragma pack` and `aligns` is possible. In the next example, each member can start at any address evenly divisible by 2, but the structure will be stored starting from a 16-byte boundary address:
```cpp
#pragma pack(2)
struct alignas(16) XY
{
  int   i = 1;
  char  c = 0xBB;
  short s = 10;
};
```
```
000000123289FA9C  00000000
000000123289FAA0 |00000001| [i]
000000123289FAA4 |000A**bb| [s][c]
000000123289FAA8 |********|
000000123289FAAC |********|
000000123289FAB0  00000000
```
The same happens for this structure:
```cpp
#pragma pack(2)
struct alignas(16) Data16
{
  char    a = 0xAA;
  short   b = 1;
  char    c = 0xBB;
  int     d = 2;
  char    e = 0xCC;
  double  f = 1.0;
};
```
```
00000099430BFE3C  00007ff6
00000099430BFE40 |0001**aa| [b][a]
00000099430BFE44 |0002**bb| [d][c]
00000099430BFE48 |**cc0000| [e][d]
00000099430BFE4C |00000000| [f]
00000099430BFE50 |3ff00000| [f]
00000099430BFE54 |********|
00000099430BFE58 |********|
00000099430BFE5C |********|
00000099430BFE60  00000000
```
Particular attention must be taken when mixing structures with different packing values. Consider the following case:
```cpp
#pragma pack(1)
struct XY
{
  int x = 1;
  int y = 2;
};

#pragma pack() //restore default

struct Test
{
  char a = 0xAA;
  XY xy;
};
```
XY alignment and size are 1 and 8. Creating single objects of type XY is harmless because it's challenging to identify a case where the integers will be unaligned. The reason is the same as we have seen when discussing stack alignment: the compiler keeps the stack-top always aligned to the current platform default alignment. When the compiler reserve stack space for XY (or allocate a block for local variables that includes XY), it will resize the stack keeping the stack-top aligned. Remember that also 8-bit values are promoted to a larger type when pushed onto the stack, so there won't be a case that misaligns XY members. The only exception is when the programmer misaligns the stack manually at the Assembly language level. Using XY structure alone is harmless:
```
000000964F0BFBDC  00000000
000000964F0BFBE0 |00000001| [x]
000000964F0BFBE4 |00000002| [y]
000000964F0BFBE8  1162028b
```
Using XY within other aggregates is a different story. For the Test structure, XY will be constructed immediately after the `char` because XY can get aligned to a 1-byte boundary. The result is that alignment and size for Test are 1 and 9, and might go unnoticed:
```
000000010007FD9C  00007ff7
000000010007FDA0 |000001aa| [xy.x] [a]
000000010007FDA4 |00000200| [xy.y] [xy.x]
000000010007FDA8  .....|00| [xy.y]
000000010007FDAC  ........
```
In any case the alignment is not relative to the parent structure. If you have a simple structure defined as:
```cpp
struct alignas(8) Test
{
  int x = 1;
}
```
You can try to create a structure that encapsulates Test in a way that `x` will always be stored at an address multiple of 8:
```cpp
struct alignas(16) AAA
{
  char c = 'c';
  int  a = 5;
  Test t;
  int  b = 6;
};
```
It's wrong to expect the structure to be allocated like this:
```
00000099430BFE40 |******63| [c]
00000099430BFE44 |00000005| [a]
00000099430BFE48 |00000001| [t]
00000099430BFE4C |00000006| [b]
00000099430BFE50  3ff00000
00000099430BFE54  2dfe2e0e
00000099430BFE58  ffcc0000
```
It seems legit because even an array of AAA will always align `t` to a multiple of 8. Unfortunately, the structure `Test` doesn't care where/how `x` will lie in memory because the actual `Test` structure is this:
```
0000003446BCF764  aa000099
0000003446BCF768 |00000001| [x]
0000003446BCF76C |********| padding
0000003446BCF770  00111111
```
So even if `x` will fall into 8-byte aligned address, and no arrays of Test are used, the padding bytes will aways be there, expanding AAA to 32 bytes in size. Since AAA must start at 16-byte aligned address, the memory layout will be:
```
00000099430BFE3C  00000000
00000099430BFE40 |******63| [c]
00000099430BFE44 |00000005| [a]
00000099430BFE48 |00000001| [t]
00000099430BFE4C |********|  T.padding
00000099430BFE50 |00000006| [b]
00000099430BFE54 |********| AAA.padding
00000099430BFE58 |********| AAA.padding
0000003446BCF75C |********| AAA.padding
0000003446BCF760  cccccc63
```
Let's bring back some union examples to see how the packing can affect the alignment.
```cpp
#pragma pack(2)
union U1
{
  char   c
  int    i;
  double d;
};
```
The memory layout we expect:
```
0000006E9C93F6EC  00000000
0000006E9C93F6F0 |00000000| [d] [i] [c]
0000006E9C93F6F4 |00000000| [d]
0000006E9C93F6F8  00000000
0000006E9C93F6FC  00000000
```
Its size and alignment are 8 and 2. If we use the union inside this structure:
```cpp
struct H
{
  char a;
  U1   u1;
};

H h;
h1.a    = 'a';
h1.u.d  = 1.0;
```
The structure alignment and size are 2 and 10. Members are now misaligned:
```
0000006BF77FF744  ........
0000006BF77FF748 |0000**61| [u1.d][u1.i][u1.c] [a]
0000006BF77FF74C |00000000| [u1.d][u1.i]
0000006BF77FF750  ...|3ff0| [u1.d]
0000006BF77FF754  ........
```
The first byte is taken for `c`, and the second byte is a padding byte; the union can lie its members starting from an address multiple of 2. So union fields can start in memory from the address 4A. It's similar to what happened with structure XY using a non-natural alignment. Either `i` or `d` for U1 will be misaligned in this case.

A combination of packing and alignment is possible for unions too, and it changes the allocation behavior as we can expect:
```cpp
#pragma pack(2)
union U2
{
              int    i;
  alignas(16) double d;
};

U2 u2;
u2.d = 1.0;
```
For U2 itself, whose size and alignment are both 16, a typical allocation would be:
```
0x0000006C4C5EF9FC  00000000
0x0000006C4C5EFA00 |00000000| [u2.d] [u2.i]
0x0000006C4C5EFA04 |3ff00000| [u2.d]
0x0000006C4C5EFA08 |********|
0x0000006C4C5EFA0C |********|
0x0000006C4C5EFA10  9f313250
```
Using U2 inside H again:
```cpp
#pragma pack(2)
union U2
{
              int    i;
  alignas(16) double d;
};

struct H
{
  char a;
  U2   u2;
};

H h;
h.a    = 'a';
h.u2.d = 1.0;
```
```
00000011596FFACC  00000010
00000011596FFAD0 |******61| [a]
00000011596FFAD4 |********|
00000011596FFAD8 |********|
00000011596FFADC |********|
00000011596FFAE0 |00000000| [u2.d] [u2.i]
00000011596FFAE4 |3ff00000| [u2.d]
00000011596FFAE8 |********|
00000011596FFAEC |********|
00000011596FFAF0  9af5c52c
```
Why the union is not allocated immediately after `a` is quite simple: we specified that we want the `double` to be allocated at an address multiple of 16 and we cannot break this rule specifying 2-byte packing. Once `a` is places at D0, the compiler will find next suitable address for `i` (or `d`) to be E0. The packing directive is getting ignored and the size and alignment are 32 and 16.

What happens if the inheritance comes into play? Let's start with a simple example:
```cpp
struct alignas(2) A
{
  char c = 'c';
};

struct B : public A
{
  int i = 1;
};
```
Since A will occupy 2 bytes and the other 2 bytes are padded to store `i`, the alignment only affects A. Size and alignment for A are both 2. Size and alignment for B are 8 and 4, regardless of having or not alignas(2) for A. The same is true if A is required to align to 4-byte. Let's see what happens if we align A to 8 bytes instead:
```cpp
struct alignas(8) A
{
  char c = 'c';
};

struct B : public A
{
  int i = 1;
};
```
In this case the size and alignment of A are both 8. Object A will be allocated this way:
```
00000010A3BBFBCC  00007ff7
00000010A3BBFBD0 |******63| [c]
00000010A3BBFBD4 |********|
00000010A3BBFBD8  41cd7afc
```
When the compiler constructs B in memory, it merges the data, overwriting padding bytes with the following member whenever the latter can align and use the space of padding bytes. In this case, writing the integer after 7 bytes is unnecessary since it can align after 3. The first construction is wrong, while the second is correct:
```
00000010A3BBFBCC  00007ff7
00000010A3BBFBD0 |******63| [c]
00000010A3BBFBD4 |********|
00000010A3BBFBD8 |00000001| [i] //wrong
00000010A3BBFBDC  41cd7afc
```
```
00000010A3BBFBCC  00007ff7
00000010A3BBFBD0 |******63| [c]
00000010A3BBFBD4 |00000001| [i] //correct
00000010A3BBFBD8  cccccccc
00000010A3BBFBDC  41cd7afc
```
If we merge the results, we can consider B defined as follow:
```cpp
struct alignas(8) B
{
  char c = 'c';
  int  i = 1;
}
```
No padding bytes are added at the end because a 2-element array of B will align the second `c` to an address multiple of 8, which fits perfectly the requirement.\
Let's increase the alignment of A to 16:
```cpp
struct alignas(16) A
{
  char c = 'c';
};

struct B : public A
{
  int i = 1;
};
```
Object A must align to a 16-byte address; its size and alignment are 16. B inherit the alignment for `c` but can insert `i` immediately after 3 bytes:
```
000000A06CAFF6FC  00007fff
000000A06CAFF700 |******63| [c]
000000A06CAFF704 |00000001| [i]
000000A06CAFF708 |********|
000000A06CAFF70C |********|
000000A06CAFF710  cf1532cc
```
It's time to analyze a more complicated example:
```cpp
#pragma pack(2)
struct A
{
  char c = 'c';
  int  i = 1;
};
#pragma pack()

struct B : public A
{
  char c2 = 'd';
  int  i2 = 2;
};
```
You must be able to depict the memory layout for A, whose size and alignment are 6 and 2:
```
000000127816FADC  00007fff
000000127816FAE0 |0001**63| [i][c]
000000127816FAE4  ccc|0000| [i]
000000127816FAE8  cccccccc
```
To construct B, remember to inherit the packing as well:
```
000000127816FAEC  00007ff6
000000127816FAF0 |0001**63| [ i][c]
000000127816FAF4 |**640000| [c2][i]
000000127816FAF8 |00000002| [i2]
000000127816FAFC  00010063
```
The size of B is 12, but the alignment is 4. Remember the alignment cannot be smaller than the default alignment; in this case the biggest type used is `int`.

Next example:
```cpp
#pragma pack(2)
struct alignas(8) A
{
  char c = 'c';
  int  i = 1;
};
#pragma pack()

struct alignas(4) B : public A
{
  char  c2 = 'd';
  double d = 1.0;
};
```
The size and alignment of A are both 8. For object B, alignas(4) will cause a compilation error because requested alignment is less than minimum for B (which is 8). Removing the alignas (or replacing 4 with 8) make object B size and alignment 16 and 8:
```
0000007E0CAFFC7C  00000000
0000007E0CAFFC80 |0001**63| [ i][c]
0000007E0CAFFC84 |**640000| [c2][i]
0000007E0CAFFC88 |00000000| [d]
0000007E0CAFFC8C |3ff00000| [d]
0000007E0CAFFC90  00010010
```
For enumerators, the alignment depends on the underlying value:
```cpp
//alignof(Color) = 4 (default)
enum class Color {...};

//alignof(Color) = 1
enum class Color : char {...};

//alignof(color) = 4
enum class Color : int  {...};

//alignof(Color) = 8
enum class Color : long long {...};
```
The compiler emits a warning when types misalign or when a structure gets padded due to alignment directives. If not, you can raise the warning level to catch those (maybe unwanted) situations. The highest warning level is `EnableAllWarnings`:
```
Properties > C/C++ > General > Warning Level
```
The command line option to set the highest available warning level is `/Wall`.

Specifying alignment is particularly useful in many cases. Working with 64-bit integers in 32-bit code is one of those. As the stack is always aligned to 4-byte boundaries, 64-bit integers might need to be properly aligned to their natural alignment. Specifying the alignment when declaring a 64-bit integer will help the compiler dynamically adjust the stack to match the integer alignment. Another case is whenever vector types are involved. These SIMD types must be aligned to specific boundaries for faster access unless we want slower unaligned access. Another possibility is that some system functions cannot be used with unaligned locations (e.g., interlocked operations).

It's possible to assume some data to be not aligned using the Microsoft-specific `__unaligned` modifier. Doing so, the compiler responsible for the current platform will take appropriate actions to generate code that performs the unaligned access. Marking data unaligned isn't valid on x86 architectures and raises a compilation error.

I want to add a final note about the packing. Packing data to avoid padding bytes always seems useful because it saves space. In general it's not, because accessing unaligned data raises hardware exceptions. Most systems re-issue the read to recompose the data into a register with a consequent small performance loss. The CPU and operating system handle the exception behind the scene. When the system expects the program to handle such exception but the program doesn't, the process crashes. Packing can be beneficial for those systems whose space is more important than speed. In general, stuffing data should be avoided unless necessary. Some considerations to keep into account are:
* Size (code and memory)
* CPU caching consequences
* Possible inhibition of compiler optimizations
* Breaking existing client binaries

The compiler offers a way to query alignment for built-in and user-defined objects. Any `pragma pack` and alignment directives affect the calculation:
```cpp
std::cout << alignof(short) << " " << alignof(char&) << " " << alignof(Test) << std::endl;
```
`alignas` accept only compile-time constants, including constant values or values returned by `alignof` and `sizeof`:
```cpp
//align to 2 bytes boundary
struct alignas(2) AAA
{
  char c = 'c';
  constexpr static int Value() { return 32; }
};

//align to 8 bytes boundary
struct alignas(sizeof(int)*2) BBB
{...};

//same alignment as BBB
struct alignas(alignof(BBB)) CCC
{...};

//align to 32 bytes boundary
struct alignas(AAA::Value()) DDD
{...};

//size = 6, alignment = 2
#pragma pack(2)
struct EEE
{
  char c = 'c';
  int  i = 1;
};

//compilation error: alignment must be a power of 2
struct alignas(sizeof(EEE)) FFF
{
  int i = 1;
};

//no need for alignof -> alignof(EEE) implied
//alignment = 2 -> not ignored even if smaller than the default (int)
struct alignas(EEE) GGG
{
  int i = 1;
};
```
