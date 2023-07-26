> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### BUILT-IN TYPES
C++ built-in types (or fundamental types) are implementation-dependent. Their size can be different among compilers and hardware architectures. Most types are numeric of different sizes and can be either signed or unsigned. The standard defines which types belong to the language but doesn't dictate strict rules, except the minimum representable range of numbers that must be guaranteed (which indirectly corresponds to their size because a minimum number of bits is required to represent a range of numbers). There are no rules for floating-point values except that `double` must provide at least the same precision as `float`. `char` size must always be 8-bit. Compilers are also required to agree with the following size-relationship between types:
```cpp
sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)
```
In recent Windows systems with widely-used compilers, the chance of finding the same type of different sizes should be nearly non-existent, and most of the types, either in 32-bit or 64-bit, have the same size. The following table lists types and their corresponding size (first colum for 32-bit and second column for 64-bit). The first few compilers are Microsoft-compatible. The rightmost ones are running on other operating systems:
```
                     Microsoft    Intel  clang-cl  clang     GCC
                    19.36.32537 2023.2.0  15.0.1   16.0.0    13.1
void                   -  -       -  -     -  -     -  -     -  -
nullptr                4  8       4  8     4  8     4  8     4  8

bool                   1  1       1  1     1  1     1  1     1  1
char                   1  1       1  1     1  1     1  1     1  1
signed char            1  1       1  1     1  1     1  1     1  1
unsigned char          1  1       1  1     1  1     1  1     1  1
short                  2  2       2  2     2  2     2  2     2  2
short int              2  2       2  2     2  2     2  2     2  2
signed short           2  2       2  2     2  2     2  2     2  2
signed short int       2  2       2  2     2  2     2  2     2  2
unsigned short         2  2       2  2     2  2     2  2     2  2
unsigned short int     2  2       2  2     2  2     2  2     2  2

int                    4  4       4  4     4  4     4  4     4  4
signed                 4  4       4  4     4  4     4  4     4  4
signed int             4  4       4  4     4  4     4  4     4  4
unsigned               4  4       4  4     4  4     4  4     4  4
unsigned int           4  4       4  4     4  4     4  4     4  4
long                   4  4       4  4     4  4     4  8     4  8
long int               4  4       4  4     4  4     4  8     4  8
signed long            4  4       4  4     4  4     4  8     4  8
signed long int        4  4       4  4     4  4     4  8     4  8
unsigned long          4  4       4  4     4  4     4  8     4  8
unsigned long int      4  4       4  4     4  4     4  8     4  8

long long              8  8       8  8     8  8     8  8     8  8
long long int          8  8       8  8     8  8     8  8     8  8
signed long  long      8  8       8  8     8  8     8  8     8  8
signed long long int   8  8       8  8     8  8     8  8     8  8
unsigned long long     8  8       8  8     8  8     8  8     8  8
unsigned long long int 8  8       8  8     8  8     8  8     8  8

float                  4  4       4  4     4  4     4  4     4  4
double                 8  8       8  8     8  8     8  8     8  8
long double            8  8       8  8     8  8    12 16    12 16

__int8                 1  1       1  1     1  1     -  -     -  -
signed __int8          1  1       1  1     1  1     -  -     -  -
unsigned __int8        1  1       1  1     1  1     -  -     -  -

__int16                2  2       2  2     2  2     -  -     -  -
signed __int16         2  2       2  2     2  2     -  -     -  -
unsigned __int16       2  2       2  2     2  2     -  -     -  -

__int32                4  4       4  4     4  4     -  -     -  -
signed __int32         4  4       4  4     4  4     -  -     -  -
unsigned __int32       4  4       4  4     4  4     -  -     -  -

__int64                8  8       8  8     8  8     -  -     -  -
signed __int64         8  8       8  8     8  8     -  -     -  -
unsigned __int64       8  8       8  8     8  8     -  -     -  -

wchar_t                2  2       2  2     2  2     4  4     4  4
__wchar_t              2  2       2  2     2  2     -  -     -  -
char8_t                1  1       1  1     1  1     1  1     1  1
char16_t               2  2       2  2     2  2     2  2     2  2
char32_t               4  4       4  4     4  4     4  4     4  4
```
As you can see from the table above, Microsoft-compatible compilers implement the same type using the same size to provide binary compatibility.\
`long` is always 4-byte in Windows but can be 8-byte somewhere else. Therefore, assuming `long` size to be always 4-byte can have undefined behavior when writing cross-platform code. We don't have to worry about that anyway because we're only interested in Windows systems; since the systems and the ABI are different, there is no chance that different binaries can natively run in different operating systems.\
Microsoft-specific sized integers are not implemented in non-compatible compilers for obvious reasons. Even for a Microsoft-compatible compiler, these sized-integer types might not be real built-in types; for example, GCC-MinGW simulates them using typedef.

Some types in the list are redundant due to synonyms. Each of the following groups contains siblings treated the same way by the Microsoft compiler, and cannot be used for overload resolution. The only exception is the `double - long double` regardless having the same size, they're different built-in types:

```cpp
char
__int8
```
```cpp
signed char
signed __int8
```
```cpp
unsigned char
unsigned __int8
```
```cpp
short
short int
signed short
signed short int
__int16
signed __int16
```
```cpp
unsigned short
unsigned short int
unsigned __int16
```
```cpp
int
signed
signed int
__int32
signed __int32
```
```cpp
unsigned
unsigned int
unsigned __int32
```
```cpp
long
long int
signed long
signed long int
```
```cpp
unsigned long
unsigned long int
```
```cpp
long long
long long int
signed long long
signed long long int
__int64
signed __int64
```
```cpp
unsigned long long
unsigned long, long int
unsigned __int64
```
```cpp
double
long double
```
Signed integers use 2's complement notation. Floating-point values adhere to IEEE-754 standards. The compiler implements some of the siblings equally, either at low-level or high-level, without making distinctions. That's the case for `char` and `__int8`, or `short` and `__int16`:
```cpp
void Hello(char c)
{...}

//error: 'void Hello(char)' already has a body
void Hello(__int8 i8)
{...}
```
```cpp
void Hello(short s)
{...}

//error: 'void Hello(short)' already has a body
void Hello(signed short int ssi)
{...}

//error: 'void Hello(short)' already has a body
void Hello(__int16 i16)
{...}
```
There can be exceptions to the rule. The exception in this case belong to `double`-`long double` pair. Both are implemented according to IEEE-754 standard using 64-bit: 1-bit sign + 11-bit exponent + 52-bit fraction but at high-level they behave slightly different than previous pairs:
```cpp
void Hello(double d)
{...}

void Hello(long double ld)
{...}

double      d  = 0;
long double ld = 0;

Hello(d );   //Hello(double)
Hello(ld);   //Hello(long double)

Hello(0.0);  //Hello(double)
Hello(0.0l); //Hello(long double)
```
The language provides built-in literal that cannot be changed. Built-in literals are named boolean, numeric, and pointer literals. We'll explore the details in next paragraphs.

`void` defines a sort of incomplete type, and for such reason, it doesn't have a size. Hence objects of type void are not allowed. It's normally used to declare functions that don't return values. It can also be used to declare generic pointers to arbitrary data; in that case, the size of void pointer is the same as any other raw pointer for the current platform. In more complex scenarios (e.g., templates), an expression can evaluate to void, which translates to `nothing` for that context.

`nullptr` is the pointer literal that replaces the NULL macro. Its size is equal to any raw pointer for the current platform. Under the hood, the compiler still assigns zero to 32-bit or 64-bit variables. Well-known issues using NULL macro don't exist with nullptr because it's neither defined as an integer nor convertible to any other type except raw pointers:
```cpp
int a = nullptr; //error

int b = 0;

if(b == nullptr) //error

bool c = false;

if(c == nullptr) //error

void F1(int  x){...}
void F1(int* p){...}

F1(0);          //call F1(int )
F1(NULL);       //call F1(int )
F1((int*)0);    //call F1(int*)
F1((int*)NULL); //call F1(int*)
F1(nullptr);    //call F1(int*)
```
There are still workarounds to foolish the language rules, but they must be intentionally used:
```cpp
int a = reinterpret_cast<int>(nullptr); //OK but non-sense (use 0 instead)
```
`bool` uses 8-bit to wrap boolean values encoded using `1` or `0` for `true` and `false` (boolean literals). The compiler still accepts non-zero values for true and zero for false. The value -6 can be assigned to a boolean, making it true. The compiler will replace -6 with 1 and move 1 to the corresponding byte in memory:
```cpp
bool a = -6; //true
```
```
000000B905C7F6B0  88faddcc
000000B905C7F6B4  cccccc01 [a]
000000B905C7F6B8  50afafaf
```
Unfortunately, the boolean type has not been yet updated to solve the same language glitches of enumerators (now strongly typed and strongly scoped), and the use of true and false can be optional:
```cpp
bool b = 1; //set b true

if(b == 0)  //check for false
```
Minimum and maximum representable values for built-in numeric types can be retrieved using `numeric_limits` defined in `<limits>` header file. It's nothing more than a template class with specialization for each type that can expose minimum and maximum value, which obviously doesn't apply to certain types such as `void` or `nullptr`.\
Before C++, limit values were defined using C macros, such as INT_MIN and CHAR_MAX. These macros are still preferred because the text-replacing mechanism can be used as a constant expression at compile-time. Now numeric_limits static members are `constexpr` and can be used for the same purpose as well; no need to keep using old C macros. Plus, numeric_limits offers more information:
```
numeric_limits<..>::min()
numeric_limits<..>::max()
numeric_limits<..>::digits
numeric_limits<..>::is_integer
numeric_limits<..>::has_infinity
...
...
```
```cpp
bool                    false                true

char                    -128                 127
signed char             -128                 127
__int8                  -128                 127
signed __int8           -128                 127

unsigned char           0                    255
unsigned __int8         0                    255

short                   -32768               32767
short int               -32768               32767
signed short            -32768               32767
signed short int        -32768               32767
__int16                 -32768               32767
signed __int16          -32768               32767

unsigned short          0                    65535
unsigned short int      0                    65535
unsigned __int16        0                    65535

int                     -2147483648          2147483647
signed                  -2147483648          2147483647
signed int              -2147483648          2147483647
long                    -2147483648          2147483647
long int                -2147483648          2147483647
signed long             -2147483648          2147483647
signed long int         -2147483648          2147483647
__int32                 -2147483648          2147483647
signed __int32          -2147483648          2147483647

unsigned                0                    4294967295
unsigned int            0                    4294967295
unsigned long           0                    4294967295
unsigned long int       0                    4294967295
unsigned __int32        0                    4294967295

long                    -9223372036854775808 9223372036854775807
long long int           -9223372036854775808 9223372036854775807
signed long             -9223372036854775808 9223372036854775807
signed long int         -9223372036854775808 9223372036854775807
__int64                 -9223372036854775808 9223372036854775807
signed __int64          -9223372036854775808 9223372036854775807

unsigned long           0                    18446744073709551615
unsigned long long int  0                    18446744073709551615
unsigned __int64        0                    18446744073709551615

wchar_t                 0                    65535
__wchar_t               -                    -
char8_t                 0                    255
char16_t                0                    65535
char32_t                0                    4294967295

float                   1.17549e-38          3.40282e+38
double                  2.22507e-308         1.79769e+308
long double             2.22507e-308         1.79769e+308
```
`signed` and `unsigned` modifiers are ignored (yielding a warning) for:
```cpp
bool
char8_t
char16_t
char32_t
wchar_t
float
double
long double
```
Qualifiers such as `const` or `volatile` doesn't affect the size, and the minimum and maximum values are the same as the unqualified type:
```cpp
int          -2147483648  2147483647
const int    -2147483648  2147483647
volatile int -2147483648  2147483647
...
```
`int` type was designed to be equal to the platform `word size`. The term has little to do with the 16-bit type used in Assembly or any Win32 typedef. It's rather a name to identify a class of architectures. In hardware jargon, the size of `hardware word` is used to quantify the size of CPU registers or the largest possible memory address and other details. In a 16-bit era of Windows, the `int` type was implemented using 16-bit. In 32-bit Windows using 32-bit. 64-bit still use 32-bit because too much software runs on the assumption that `int` is 32-bit.

Numerical literals are integers by default. Whenever you see a number `1024`, `-55`, `076`, `0xAA`, `0B11001101`, all of them are signed integers (32-bit if they fit, 64-bit integers otherwise). This means there is no way to query them as 8-bit or 16-bit:
```cpp
decltype(0)           x; //int
decltype(12000)       y; //int
decltype(10000000000) z; //__int64 (long long)
```
Suffixes can be used to instruct the compiler to treat numerical literals differently:\

`l`   `L`    = long\
`ll`  `LL`   = long long\
`u`   `U`    = unsigned\
`f`   `F`    = float\
`l`   `L`    = long double\
`i8`  `I8`   = char\
`i16` `I16`  = short\
`i32` `I32`  = int\
`i64` `I64`  = long long\

Once again, there is no way to define a `short` literal in standard C++. Suffixes `iN` are Microsoft-specific and not much used. A suffix can be specified in either upper-case or lower-case, some of them in different order too:
```cpp
auto a = 1u;      //decltype(1u)    = unsigned int
auto b = 2l;      //decltype(2l)    = long
auto c = 3ll;     //decltype(3ll)   = long long
auto d = 4ul;     //decltype(4ul)   = unsigned long
auto e = 5ull;    //decltype(5ull)  = unsigned long long
auto f = 6lU;     //decltype(6LU)   = unsigned long
auto g = 7LLU;    //decltype(7LLU)  = unsigned long long
auto h = 1.0f;    //decltype(1.0f)  = float
auto i = 1.0;     //decltype(1.0)   = double
auto j = 1.0l;    //decltype(1.0l)  = long double
auto k = 0b01;    //decltype(0b01)  = int
auto l = 0b1000u; //decltype(0b11u) = unsigned int
auto m = 10i8;    //decltype(10i8)  = char
auto n = 1i16;    //decltype(1i16)  = short
auto o = 2I32;    //decltype(2I32)  = int
auto p = 10i64;   //decltype(10i64) = __int64
auto q = 0b1I64;  //decltype(0b1I64)= __int64
auto r = 0717;    //decltype(0776)  = int
```
The language defines conversions between built-in types. Integer promotions, integer conversions, floating-point conversions to name a few.

Expressions that use operands of different sizes will use built-in rules for conversion. In the case of promotion (smaller type assigned to larger one), the compiler will not issue any warning even with the highest warning level set. For narrow conversion (larger type assigned to smaller one), the compiler will yield a warning (information loss may not happens depending on the case). Narrow conversion clips out the higher part from the source operand because the destination doesn't have enough bits to hold the value. If the lower part is outside the representable range for the destination type, the difference exceeding the maximum value will be wrapped around. To understand what wrap-around means, you can try incrementing the number on the edge of a representable range.\
Suppose you have a `char` whose value is 127 in binary 01111111. Adding 1 to it yelds 10000000 which corresponds to -128. The same happens on the opposite side: if you have -128 = 10000000, subtract 1 to get 01111111 = 127. Let's see a few examples.
```cpp
short s = 980500;
```
980500 is equal to 000EF614. `short` is only 16-bit wide, so only F614 will be converted. F614 = 62996. According to the table, a signed short cannot hold such big number because outside its range. It exceeds the maximum value for an amount of 62996-32767 = 30229. Adding 1 to 32767 jumps to -32768. Adding the remaining 30228 gives -2540. That's the value assigned to s.
```cpp
short s = -850600;
```
-850600 is equal to FFF30558. The `short` have only 16-bit to store that value so FFF3 will be removed. 0558 = 1368; in this case, the number is already in range and assigned to s.
```cpp
unsigned short s = -5000;
```
-5000 = FFFFEC78h. Cutting the higher 16-bit part, we get EC78 = 60536 which is in the range.
```cpp
char c = -5000;
```
We have already calculated the value for -5000, which is EC78. The `char` can hold 1 byte, hence 78h = 120. That's in range.
```cpp
char c = -10000;
```
-10000 = FFFFD8F0h. The `char` is still 1 byte, so F0 = 240 is out of the range. The difference is 240-127 = 113. Adding 1 to 127 gets to -128; adding the remaining 112 yelds to -16.
```cpp
unsigned char c = -10000;
```
The value from the previous example is again F0 but this time the char is unsigned and 240 is an acceptable value.

Floating-point values can be converted from lower precision to higher precision without loss of information. Example from `float` to `double`:
```cpp
double d = 1.0f;
```
1.0f encoded as 3F800000h will be converted to 3FF0000000000000h (1.0 double-precision) and assigned to d. No warnings are reported because there is no information loss since the `double` must use at least the same precision of `float`. The other way around:
```cpp
float f = 3.1415926535;
```
The compiler will warn about the truncation that may cause a loss of information. If the value cannot be represented (due to the encoding constraints), the same value is rounded to next or previous representable value.
> For more information about IEEE-754 floating-point encoding please check proper documentation.
Conversion from floating-point to integral types:
```cpp
char c = 97.99f;
```
The floating-point value is first converted to integral type and then assigned (with wrap-around if necessary). 97 in this case is representable value so c = 'a'.
```cpp
char c = 1500.5;
```
`double` value 1500.5 become 1500 first. 1500 = 5DC but we can only take the lower byte DC = 220. The value is outside the range for an amount of 220-127 = 93. Wrap around to -128 yelds -36.

`bool` minimum and maximum values are somehow deceptive because they are defined as `false` and `true`. numeric_limits returns 0 and 1 (or false and true depending on the bool alpha operator). The most tricky thing is that the conversion from numerical literals doesn't seem to follow rules described above:
```cpp
bool b = 40960; //true
```
40960 is equal to A000. Since `bool` is an 8-bit value internally, after removing extra bytes, the most obvious answer would be 0 = false. The compiler will assign true instead. The only value accepted for false is zero. Any other sequence of bits becomes true:
```cpp
bool a = 0;    //false
bool b = 0.0f; //false
bool c = '\0'; //false

bool d = 1.0f; //true
bool e = 'c';  //true
bool f = 0xA0; //true
```
There is still an important rule to watch out for, used by implicit conversions in expressions evaluation. The conversion goes from smaller to larger and floating-point types, and never the opposite to avoid loss of precision:
```cpp
char -> int -> long long -> float -> double
```
An expression like `'a' + 1.0f` will convert `a` to floating-point value producing `97.0f + 1.0f = 98.0f`. Here are some examples:
```cpp
int    a = 1.0  + 'a'  - 22.5;     //1.0 + 97.0 - 22.5 = 75.5 -> 75
char   b = 1.0  + 'a'  - 22;       //1.0 + 97.0 - 22.0 = 76.0 -> 76
double c = 1.0f + 'a'  - 22;       //1.0f + 97.0f - 22.0f = 76.0f -> 76.0
auto   d = 1.0f + L'a' - 22;       //1.0f + 97.0f - 22.0f = 76.0f
char   e = 100  * L'a' - 22;       //100 * 97 - 22 = 9678 = 25CEh -> CEh = -50
char   f = 100  * 0.5  - 22 + 'a'; //100.0 * 0.5 - 22.0 + 97.0 = 125.0 -> 125
```
Conversion also happen during other arithmetics operations. For example:
```cpp
auto x = 'a' << 30;
```
We know that 'a' = 97. Integer literal are 32-bit integer by default. We're performing a 32-bit integer operation:
```
97 << 30 = 0001 1000 0100 0000 0000 0000 0000 0000 0000 0000
```
There is space for the lower 4 bytes only:
```
0100 0000 0000 0000 0000 0000 0000 0000 = 1073741824
```
```cpp
auto x = 'a' << 48ull;
```
Still in the 32-bit integer case even we specify we want to shift by an amount that requires a bigger type. This time x will be 0. An explicit conversion is the only way:
```cpp
auto x = (unsigned long long)'a' << 48ull; //x = 27303072740933632
```
Microsoft-specific sized-integer types are signed by default and synonyms of the corresponding ANSI type that have the same size as we've seen before. These types can use `signed` and `unsigned` modifiers:
```cpp
__int8  ~ char
__int16 ~ short
__int32 ~ int
__int64 ~ long long

unsigned __int8  ~ unsigned char
unsigned __int16 ~ unsigned short
unsigned __int32 ~ unsigned int
unsigned __int64 ~ unsigned long long
```
Sized-integer types are real types and not typedef. Some compilers simulate them using typedef instead (that's what GCC-MinGW does). The point of having these particular types is portability: their size will always be as specified in their name and not implementation-defined. For cross-platform code, however, you may prefer to use std::int16_t and siblings.

Character types represent alphanumeric characters. `char` is an 8-bit integer by default although its name sounding like `character`, and commonly used to store ASCI ISO-8859 codes. UTF-8 UNICODE can use `char` to store characters. The literal `a` corresponds to the ASCII code for the same character, which in turn equals 65. The compiler treats `char` and `__int8` equally, even though they're different built-in types:
```cpp
void Func(char x)
{}

void Func(__int8 x) //error: function 'void Func(char)' already has a body
{}
```
This means you can create an ASCII string using `__int8`:
```cpp
void Print(const char* str)
{
  std::cout << str <<std::endl;
}

void main()
{
  const __int8* str = "hello";

  std::cout << str << std::endl; //print "hello"
  Print(str);                    //print "hello" (no warning or implicit cast)
}
```
`char` is different than both `signed char` and `unsigned char` hence the following code will compile:
```cpp
void Func(char c)
{}

void Func(signed char c)
{}

void Func(unsigned char c)
{}
```
`unsigned char` is often used to define a `BYTE` type (not yet a built-in type). `char` strings, either UNICODE or multi-byte, are called `narrow strings`. We've seen the numeric limits for all of them but I report here for clarity:
```
char            -128   127
signed char     -128   127
unsigned char      0   255
```
Either `char` or `signed char` can be used to represent the same range of values. The default behvaior for `char` is the same as `signed char` but remember those are different built-in types and can be used for overload resolution. There is a way to change the default type for `char`. Use the compiler option `/J` to change the default behavior to `unsigned char`:
```
//with /J compiler option
char               0   255
signed char     -128   127
unsigned char      0   255
```
This allow the `char` to represent a different range of values, all of them are still different built-in types.

The standard doesn't specify the size of `wchar_t`. It's 16-bit in Microsoft compiler and used to implement UTF-16LE (16-bit little-endian) native Windows character type. By default `wchar_t` is treated as built-in type (according to the standard) that maps to the real native type `__wchar_t`. The option `Treat wchar_t As Built in Type` can be changed from project settings:
```
Properties > C/C++ > Language
```
Once again I report the command line option for completeness: `/Zc:wchar_t` or `/Zc:wchar_t-`

If disabled, `wchar_t` falls back into a typedef, eventually to resolve issues when working with old code:
```cpp
typedef unsigned short wchar_t;
```
The rest of the types are used to encode one of the 3 forms defined by the standard: UNICODE encoded as UTF-8 can be stored to `char8_t`. UTF-16 can use `char16_t`, and UTF-32 is stored using `char32_t`.\
Wide character types such as `char16_t` or `char32_t` are not synonyms of UNICODE just because they are bigger than 8-bit `char`. `wchar_t` implements UNICODE if enabled, and it's implementation-defined. Furthermore, UNICODE can use variable-length encoding. It means UTF-8 may use multiple 8-bit to encode characters. Consequently, the 10th character in a string might not be the 10th byte from the base address, and most string operations (such as calculating a string length) must always scan the string from the beginning. To avoid this burden, the Kernel implements UTF-16 with fixed length encoding, where each character is exactly 2 bytes. Character literal prefixes exist for:\
wchar_t `L`\
UTF-8   `u8`\
UTF-16  `u`\
UTF-32  `U`
```cpp
auto a = u8'a'; //decltype(u8'a') = char8_t
auto w =  L'w'; //decltype( L'w') = wchar_t
auto b =  u'b'; //decltype( u'b') = char16_t
auto c =  U'c'; //decltype( u'c') = char32_t
```
```
000000601387F9AC  00000000
000000601387F9B0  ******61  [a]
000000601387F9B4  ****0077  [w]
000000601387F9B8  ****0062  [b]
000000601387F9BC  00000063  [c]
000000601387F9C0  00000000
```
A character literal without a prefix is considered ordinary `char`. These prefixes cannot change position nor alter their case. `u8` behaves differently in between C++ versions. Prior to C++20 it defined ordinary `char`, since C++20 it creates `char8_t` literal:
```cpp
//C++ 11/14/17
auto a = u8'a'; //decltype(u8'a') = char
auto b = u8"a"; //decltype(u8"a") = const char*

//C++20
auto a = u8'a'; //decltype(u8'a') = char8_t
auto b = u8"a"; //decltype(u8"a") = const char8_t*
```
Single-quote literals exist but are rarely used. Their underlying type is a 32-bit integer. When assigned to `char`, they will truncate to the lowest byte:
```cpp
char c = 'abxy'; //decltype('abxy') = int -> assign 'y' to c
```
Assigning the same to an `int` is the same as converting each digit to an 8-bit value and concatenate them to create a 32-bit integer. Example:
```cpp
int i = 'abxy';
```
First translate the ASCII code to hexadecimal:
```cpp
'a' = 97  = 61h
'b' = 98  = 62h
'x' = 120 = 78h
'y' = 121 = 79h
```
Then concatenate all values to create a 32-bit integer:
```cpp
i = 0x61627879; //1633843321
```
The number of characters in the single-quote literal cannot exceed 4. Only 32-bit integers can be created from single-quote literals, either in 32-bit and 64-bit build or when assigned to larger types. Example:
```cpp
char a = '12';  //assign '2' to a
char b = 'ABC'; //assign 'c' to b
int  c = '123'; //0x313233 = 3224115

int       d = 'abcdef'; //error: too many (no truncation happen)
long long e = 'abcdef'; //error: too many (no implicit conversion or promotion)
```
Since C++ 14, the compiler accepts single-quote characters to be used as a separator to ease readability:
```cpp
int a   = 1'200;    //1200
int b   = 2'2'5;    //225
int b   = 0xAA'BB;  //0xAABB
float c = 3.14'15f; //3.1415f
```
Data size is crucial because it can affect parameters passing and data layout. Let's analyze a problem in this category: the extended-precision `long double` case.

For the Microsoft compiler, both `double` and `long double` types are identical (nevertheless, they're different built-in types) and implemented using 64-bit. Back in time, when the FPU was the central floating-point unit, the `long double` was implemented using 80-bit and called extended-precision type. Some compilers still support it, but Microsoft dropped FPU support and translated most floating-point calculations to SSE scalar instructions. SSE instructions don't handle 80-bit floating-point type.
>Microsoft compiler may eventually use the FPU for the non-optimized 32-bit build.

Intel compiler implements the `long double` using 64-bit for binary compatibility, but at the same time it comes with `/Qlong-double` option to override the default size to 80-bit extended precision, thus using the FPU. Let's suppose the following code is part of a static library built with Intel compiler using `/Qlong-double`:
```cpp
//Util.h
struct Data
{
  int         a = 1;
  long double b = 1.0; //80-bit extended precision here
};

int Compute(const Data& data);
```
```cpp
//Util.cpp -> Util.lib
int Compute(const Data& data)
{
  return d.a + static_cast<int>(data.b);
}
```
We need to understand how Microsoft and Intel compilers implement Data allocation due to the different `long double` sizes:
```
Intel /Qlong-double:
000000718E07F7AC  ac54cccc
000000718E07F7B0 |00000001| [a] &data
000000718E07F7B4 |********|
000000718E07F7B8 |********|
000000718E07F7BC |********|
000000718E07F7C0 |00000000| [b] &data+16
000000718E07F7C4 |80000000|
000000718E07F7C8 |****3FFF|
000000718E07F7CC |********|
000000718E07F7D0  b3ef916c

Microsoft:
000000718E07F744  b3ef9094
000000718E07F748 |00000001| [a] &data
000000718E07F74C |********|
000000718E07F750 |00000000| [b] &data+8
000000718E07F754 |3ff00000|
000000718E07F758  0000000a
```
We'll explore alignment details in a later chapter. For now, remember that Intel compiler uses 16-bytes aligned data for the `long double`, writing the meaningful data to lower 10 bytes and padding the remaining 6. Being 16 bytes long, the `long double` will be stored at an address multiple of 16 (an address that ends with 0). As you can see from the first layout, the `long double` will be placed 16 bytes ahead of the base address to match the alignment. Typically the base address is the address of the Data itself, in this case, equal to the address of its first member `a` (but that's not always the case).

You're free to use the library (Util.h and Util.lib) in a project you're developing using Microsoft compiler. You include the header and link the library. Then create an object of type Data and pass it to function Compute:
```cpp
#include <iostream>
#include "Util.h."

void main()
{
  Data data;

  std::cout << Compute(data) << std::endl;
}
```
The code will compile and link successfully. Problems arise at run-time. This time Microsoft compiler allocates Data, and because the `long double` is still 64-bit (8 bytes), the compiler can place that value at an address multiple of 8. On the second layout above `Data.b` is stored 8 bytes away from the base address. What will happen at run-time is quite obvious: data allocated by Microsoft compiler will be processed by instructions generated by Intel compiler. These instructions tell the CPU to read the `long double` at &data+16. At that address, there is anything but the `long double`. We don't take a crash for granted because there might be other readable data at higher addresses, but the CPU will read garbage and return a wrong value anyway. That's the first problem. The second problem is how the encoding of 1.0 differs between 64-bit and 80-bit:
```
Microsoft long double 64-bit = 3FF0000000000000
Intel /Qlong-double   80-bit = 3FFF8000000000000000
```
Even if we move the Microsoft `long double` to the proper location at &data+16, the FPU won't be happy anyway. We can anyway attempt to fix it. The trick used here is valid for this specific case and for educational purposes only:
```cpp
#include <iostream>
#include "Util.h."

void main()
{
  __declspec(align(16)) BYTE data[32];

  int i = 1;

  int a = 0;
  int b = 0x80000000;
  int c = 0x00003FFF;

  memcpy(data,    &i, sizeof(int));
  memcpy(data+16, &a, sizeof(int));
  memcpy(data+20, &b, sizeof(int));
  memcpy(data+24, &c, sizeof(int));

  std::cout << Compute(reinterpret_cast<Data&>(data)) << std::endl;
}
```
The code allocates data to a 16-byte boundary, and after setup the first integer, it copies the 80-bit value to proper locations recreating the same Intel layout:
```
000000BEFDEFFB8C  00007ff6
000000BEFDEFFB90 |00000001| [i] &Data
000000BEFDEFFB94 |00000000|
000000BEFDEFFB98 |00000001|
000000BEFDEFFB9C |00000000|
000000BEFDEFFBA0 |00000000| [a] &Data+16
000000BEFDEFFBA4 |80000000| [b] &Data+20
000000BEFDEFFBA8 |00003fff| [c] &Data+24
000000BEFDEFFBAC |00000000|
000000BEFDEFFBB0  00000000
```
`a` `b` `c` hold the encoding of 80-bit 1.0. The FPU will read it correctly and give us the proper results. This is a fortunate case anyway because the function returns a 32-bit integer which always returns using EAX register. This lead to the third problem: the return type. When the FPU is involved, and the function returns a floating-point value, the callee function leaves the value for the caller in ST0 and not in the XMM0 register as expected by the Microsoft compiler.

The `long double` example shows how different binaries built with different compilers can introduce problems. We're intentionally creating troubles by manually changing the `long double` size; we could have foreseen dangers coming. In normal conditions, these compilers are binary-compatible and entanglements-free. Needless to say, there are dozens of other things that makes binaries built with different compilers (or even the same compiler) easy to break. We'll see other examples along the article.

It's possible to query built-in and user-defined object size using the `sizeof` operator at compile-time. The compiler replaces the expression with a 64-bit unsigned integer:
```cpp
std::cout << sizeof(int) << " " << sizeof(long double) << " " << sizeof(Data) << std::endl;
```
For empty objects, the size is implementation-defined and meaningless. For the Microsoft compiler, the size of the following objects is equal to 1:
```cpp
struct Empty    //sizeof(Empty) = 1
{};

class Nothing   //sizeof(Nothing) = 1
{};
```
Brackets for the `sizeof` operator are mandatory for built-in types but can be omitted for user-defined types:
```cpp
struct ABC
{
  int  x;
  char c;
};

std::cout << sizeof ABC << std::endl; //print "8"
```
