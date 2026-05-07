> Part of MSVC-ABI [Article](https://github.com/vblendpd/MSVC-ABI)

### BUILT-IN TYPES
C++ built-in types (or fundamental types) are implementation-dependent. Their size can vary across compilers and hardware architectures. Most types are numeric, of varying sizes, and can be either signed or unsigned. The standard defines which types belong to the language but does not dictate strict rules, except for the minimum representable range of values that must be guaranteed (which indirectly determines their size, since a minimum number of bits is required to represent that range). There are no rules for floating-point values except that `double` must provide at least the same precision as `float`. The size of `char` must always be 8 bits. Compilers are also required to adhere to the following size ordering between types:
```cpp
sizeof(char) <= sizeof(short) <= sizeof(int) <= sizeof(long) <= sizeof(long long)
```
In recent Windows systems with widely-used compilers, the chance of finding the same type implemented with different sizes should be nearly non-existent, and most types, whether in 32-bit or 64-bit, have the same size. The following table lists types and their corresponding sizes (first column for 32-bit and second column for 64-bit). The first few compilers are Microsoft-compatible. The rightmost ones run on other operating systems:
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
`long` is always 4 bytes in Windows but can be 8 bytes on other platforms. Therefore, assuming `long` to always be 4 bytes may lead to undefined behavior in cross-platform code. We need not worry about that here, however, since we are only interested in Windows systems; as the systems and their ABIs differ, binaries built for different operating systems cannot run natively on one another.\
Microsoft-specific sized integers are not implemented in non-compatible compilers for obvious reasons. Even for a Microsoft-compatible compiler, these sized-integer types might not be real built-in types; for example, GCC-MinGW simulates them using typedefs.

Some entries in the list are redundant, as they are synonyms of the same underlying type. Each of the following groups contains types that are treated identically by the Microsoft compiler and therefore cannot be used to distinguish function overloads. The only exception is the `double`-`long double` pair: despite having the same size, they are treated as different built-in types:

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
Signed integers use 2's complement notation. Floating-point values adhere to IEEE-754 standards. The compiler treats some of these synonyms identically, both at the low level and high level, making no distinction between them. This is the case for `char` and `__int8`, or `short` and `__int16`:
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
There is an exception to this rule. The exception in this case belongs to the `double`-`long double` pair. Both are implemented according to the IEEE-754 standard using 64 bits: 1-bit sign + 11-bit exponent + 52-bit fraction, but at the high level they behave slightly differently from the previous pairs:
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
The language provides built-in literals that cannot be changed. Built-in literals are named boolean, numeric, and pointer literals. We will explore the details in the following paragraphs.

`void` defines an incomplete type, and for this reason, it does not have a size. Hence, objects of type `void` are not allowed. It is normally used to declare functions that do not return values. It can also be used to declare generic pointers to arbitrary data; in that case, the size of a `void` pointer is the same as any other raw pointer for the current platform. In more complex scenarios (e.g., templates), an expression can evaluate to `void`, which translates to `nothing` for that context.

`nullptr` is the pointer literal that replaces the `NULL` macro. Its size is equal to that of any raw pointer for the current platform. Under the hood, the compiler still assigns zero to 32-bit or 64-bit variables. The well-known issues associated with the `NULL` macro do not exist with `nullptr`, because it is neither defined as an integer nor convertible to any type other than raw pointers:
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
There are still workarounds to circumvent the language rules, but they must be used intentionally:
```cpp
int a = reinterpret_cast<int>(nullptr); //OK but non-sense (use 0 instead)
```
`bool` uses 8 bits to wrap boolean values encoded using `1` or `0` for `true` and `false` (boolean literals). The compiler still accepts non-zero values for `true` and zero for `false`. The value -6 can be assigned to a boolean, making it `true`. The compiler will replace -6 with 1 and move 1 to the corresponding byte in memory:
```cpp
bool a = -6; //true
```
```
000000B905C7F6B0  88faddcc
000000B905C7F6B4  cccccc01 [a]
000000B905C7F6B8  50afafaf
```
Unfortunately, the boolean type has not yet been updated to address the same language pitfalls as enumerators (now strongly typed and strongly scoped), and the use of `true` and `false` is still optional:
```cpp
bool b = 1; //set b true

if(b == 0)  //check for false
```
Minimum and maximum representable values for built-in numeric types can be retrieved using `numeric_limits`, defined in the `<limits>` header file. It is nothing more than a template class with a specialization for each type, exposing its minimum and maximum values. This obviously does not apply to certain types such as `void` or `nullptr`.\
Before C++, limit values were defined using C macros such as `INT_MIN` and `CHAR_MAX`. These macros are still used because their text-substitution mechanism allows them to serve as constant expressions at compile time. Now, `numeric_limits` static members are `constexpr` and can serve the same purpose; there is no longer a need to rely on the old C macros. Furthermore, `numeric_limits` exposes additional information:
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
Qualifiers such as `const` or `volatile` do not affect the size, and the minimum and maximum values are the same as those of the unqualified type:
```cpp
int          -2147483648  2147483647
const int    -2147483648  2147483647
volatile int -2147483648  2147483647
...
```
The `int` type was designed to be equal to the platform word size. The term has little to do with the 16-bit type used in Assembly or any Win32 typedef. It is rather a name used to identify a class of architectures. In hardware jargon, the size of a `hardware word` is used to describe the size of CPU registers, the largest addressable memory unit, and other details. In the 16-bit era of Windows, the `int` type was implemented as 16 bits. In 32-bit Windows, it was implemented using 32 bits. 64-bit systems still use 32-bit integers for `int` because too much software relies on the assumption that `int` is 32 bits.

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
`i64` `I64`  = long long

Once again, there is no way to define a `short` literal in standard C++. Suffixes `iN` are Microsoft-specific and rarely used. A suffix can be specified in either upper or lower case, and some can be specified in a different order as well:
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
The language defines conversions between built-in types, including integer promotions, integer conversions, and floating-point conversions, among others.

Expressions that use operands of different sizes will apply built-in conversion rules. In the case of promotion (a smaller type assigned to a larger one), the compiler will not issue any warning even with the highest warning level set. For narrow conversion (a larger type assigned to a smaller one), the compiler will issue a warning, as information loss may occur depending on the case. Narrow conversion truncates the higher-order bits of the source operand because the destination does not have enough bits to hold the value. If the lower part is outside the representable range for the destination type, the difference exceeding the maximum value will be wrapped around. To understand what wrap-around means, you can try incrementing a number at the edge of a representable range.\
Suppose you have a `char` whose value is 127, or 01111111 in binary. Adding 1 to it yields 10000000, which corresponds to -128. The same happens on the opposite side: if you have -128 = 10000000, subtracting 1 gives 01111111 = 127. Let us see a few examples.
```cpp
short s = 980500;
```
980500 is equal to 000EF614. `short` is only 16 bits wide, so only F614 will be converted. F614 = 62996. According to the table, a signed `short` cannot hold such a large number, as it is outside its representable range. It exceeds the maximum value by an amount of 62996-32767 = 30229. Adding 1 to 32767 wraps to -32768. Adding the remaining 30228 gives -2540. That is the value assigned to `s`.
```cpp
short s = -850600;
```
-850600 is equal to FFF30558. The `short` has only 16 bits to store that value, so the upper FFF3 will be discarded. 0558 = 1368; in this case, the number is already in range and is assigned to `s`.
```cpp
unsigned short s = -5000;
```
-5000 = FFFFEC78h. Discarding the upper 16 bits, we get EC78 = 60536, which is within the range.
```cpp
char c = -5000;
```
We have already calculated the value for -5000, which is EC78. The `char` can hold 1 byte, so 78h = 120. That is within range.
```cpp
char c = -10000;
```
-10000 = FFFFD8F0h. The `char` is still 1 byte, so F0 = 240, which is out of range. The difference is 240-127 = 113. Adding 1 to 127 wraps to -128; adding the remaining 112 yields -16.
```cpp
unsigned char c = -10000;
```
The value from the previous example is again F0, but this time the `char` is unsigned and 240 is an acceptable value.

Floating-point values can be converted from lower precision to higher precision without loss of information. For example, converting from `float` to `double`:
```cpp
double d = 1.0f;
```
1.0f encoded as 3F800000h will be converted to 3FF0000000000000h (1.0 in double precision) and assigned to `d`. No warnings are reported because there is no information loss, since `double` must use at least the same precision as `float`. The other way around:
```cpp
float f = 3.1415926535;
```
The compiler will warn about the truncation that may cause a loss of information. If the value cannot be represented (due to encoding constraints), it is rounded to the next or previous representable value.
> For more information about IEEE-754 floating-point encoding please check proper documentation.

Conversion from floating-point to integral types:
```cpp
char c = 97.99f;
```
The floating-point value is first converted to an integral type and then assigned (with wrap-around if necessary). 97 in this case is a representable value, so `c = 'a'`.
```cpp
char c = 1500.5;
```
The `double` value 1500.5 becomes 1500 first. 1500 = 5DC, but we can only take the lower byte DC = 220. The value is outside the range by an amount of 220-127 = 93. Wrapping around from -128 yields -36.

`bool` minimum and maximum values are somewhat misleading because they are defined as `false` and `true`. `numeric_limits` returns 0 and 1 (or `false` and `true` depending on the `boolalpha` manipulator). The trickiest aspect is that the conversion from numerical literals does not seem to follow the rules described above:
```cpp
bool b = 40960; //true
```
40960 is equal to A000. Since `bool` is an 8-bit value internally, after removing the extra bytes, the most obvious answer would be 0 = `false`. The compiler will assign `true` instead. The only value accepted for `false` is zero. Any other sequence of bits becomes `true`:
```cpp
bool a = 0;    //false
bool b = 0.0f; //false
bool c = '\0'; //false

bool d = 1.0f; //true
bool e = 'c';  //true
bool f = 0xA0; //true
```
There is still an important rule to be aware of, used by implicit conversions in expression evaluation. Conversions proceed from smaller to larger types and toward floating-point types, never in the opposite direction, to avoid loss of precision:
```cpp
char -> int -> long long -> float -> double
```
An expression like `'a' + 1.0f` will convert `'a'` to a floating-point value, producing `97.0f + 1.0f = 98.0f`. Here are some examples:
```cpp
int    a = 1.0  + 'a'  - 22.5;     //1.0 + 97.0 - 22.5 = 75.5 -> 75
char   b = 1.0  + 'a'  - 22;       //1.0 + 97.0 - 22.0 = 76.0 -> 76
double c = 1.0f + 'a'  - 22;       //1.0f + 97.0f - 22.0f = 76.0f -> 76.0
auto   d = 1.0f + L'a' - 22;       //1.0f + 97.0f - 22.0f = 76.0f
char   e = 100  * L'a' - 22;       //100 * 97 - 22 = 9678 = 25CEh -> CEh = -50
char   f = 100  * 0.5  - 22 + 'a'; //100.0 * 0.5 - 22.0 + 97.0 = 125.0 -> 125
```
Conversions also occur during other arithmetic operations. For example:
```cpp
auto x = 'a' << 30;
```
We know that `'a'` = 97. Integer literals are 32-bit integers by default. We are performing a 32-bit integer operation:
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
This is still a 32-bit integer operation, even though we specify a shift amount that would require a larger type. This time `x` will be 0. An explicit conversion is the only way:
```cpp
auto x = (unsigned long long)'a' << 48ull; //x = 27303072740933632
```
Microsoft-specific sized-integer types are signed by default and are synonyms of the corresponding ANSI type of the same size, as shown earlier. These types can use `signed` and `unsigned` modifiers:
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
Sized-integer types are real types and not typedefs. Some compilers simulate them using typedefs instead (that is what GCC-MinGW does). The point of having these particular types is portability: their size will always be as specified in their name and not implementation-defined. For cross-platform code, however, you may prefer to use `std::int16_t` and its siblings.

Character types represent alphanumeric characters. `char` is an 8-bit integer by default, despite its name suggesting otherwise, and is commonly used to store ASCII/ISO-8859 character codes. UTF-8 UNICODE can use `char` to store characters. The literal `'a'` corresponds to the ASCII code for the same character, which in turn equals 97. The compiler treats `char` and `__int8` equally, even though they are different built-in types:
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
`char` is different from both `signed char` and `unsigned char`, hence the following code will compile:
```cpp
void Func(char c)
{}

void Func(signed char c)
{}

void Func(unsigned char c)
{}
```
`unsigned char` is often used to define a `BYTE` type (not yet a built-in type). `char` strings, either UNICODE or multi-byte, are called `narrow strings`. We have seen the numeric limits for all of them, but I repeat them here for clarity:
```
char            -128   127
signed char     -128   127
unsigned char      0   255
```
Either `char` or `signed char` can be used to represent the same range of values. The default behavior for `char` is the same as `signed char`, but remember that these are different built-in types and can be used for overload resolution. There is a way to change the default type for `char`. Use the compiler option `/J` to change the default behavior to `unsigned char`:
```
//with /J compiler option
char               0   255
signed char     -128   127
unsigned char      0   255
```
This allows `char` to represent a different range of values; all three remain distinct built-in types.

The standard does not specify the size of `wchar_t`. It is 16 bits in the Microsoft compiler and is used to implement UTF-16LE (16-bit little-endian), the native Windows character type. By default, `wchar_t` is treated as a built-in type (according to the standard) that maps to the real native type `__wchar_t`. The option `Treat wchar_t As Built in Type` can be changed from the project settings:
```
Properties > C/C++ > Language
```
Once again, I include the command-line option for completeness: `/Zc:wchar_t` or `/Zc:wchar_t-`

If disabled, `wchar_t` falls back to a typedef, primarily to resolve compatibility issues when working with legacy code:
```cpp
typedef unsigned short wchar_t;
```
The remaining types are used to encode one of the three forms defined by the standard: UNICODE encoded as UTF-8 can be stored in `char8_t`. UTF-16 can use `char16_t`, and UTF-32 is stored using `char32_t`.\
Wide character types such as `char16_t` or `char32_t` do not inherently represent UNICODE just because they are wider than an 8-bit `char`. `wchar_t` implements UNICODE if enabled, and it is implementation-defined. Furthermore, UNICODE can use variable-length encoding. This means UTF-8 may use multiple bytes to encode characters. Consequently, the 10th character in a string might not be the 10th byte from the base address, and most string operations (such as calculating a string length) must always scan the string from the beginning. To avoid this burden, the Kernel implements UTF-16 with fixed-length encoding, where each character is exactly 2 bytes. Character literal prefixes exist for:\
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
A character literal without a prefix is considered an ordinary `char`. These prefixes cannot be reordered or modified in case. `u8` behaves differently across C++ versions. Prior to C++20, it defined an ordinary `char`; since C++20, it creates a `char8_t` literal:
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
Assigning the same to an `int` is equivalent to converting each character to an 8-bit value and concatenating them to form a 32-bit integer. Example:
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
The number of characters in a single-quote literal cannot exceed 4. Only 32-bit integers can be created from single-quote literals, whether in a 32-bit or 64-bit build, or when assigned to larger types. Example:
```cpp
char a = '12';  //assign '2' to a
char b = 'ABC'; //assign 'c' to b
int  c = '123'; //0x313233 = 3224115

int       d = 'abcdef'; //error: too many (no truncation happen)
long long e = 'abcdef'; //error: too many (no implicit conversion or promotion)
```
Since C++14, the compiler accepts single-quote characters as separators to aid readability:
```cpp
int a   = 1'200;    //1200
int b   = 2'2'5;    //225
int b   = 0xAA'BB;  //0xAABB
float c = 3.14'15f; //3.1415f
```
Data size is crucial because it can affect parameter passing and data layout. Let us examine a problem in this category: the extended-precision `long double` case.

For the Microsoft compiler, both `double` and `long double` types are identical (nevertheless, they are different built-in types) and implemented using 64 bits. Historically, when the FPU was the primary floating-point unit, the `long double` was implemented using 80 bits and was known as the extended-precision type. Some compilers still support it, but Microsoft dropped FPU support and translated most floating-point calculations to SSE scalar instructions. SSE instructions do not support the 80-bit floating-point type.
>Microsoft compiler may eventually use the FPU for the non-optimized 32-bit build.

The Intel compiler implements the `long double` using 64 bits for binary compatibility, but also provides the `/Qlong-double` option to override the default size to 80-bit extended precision, thereby using the FPU. Let us suppose the following code is part of a static library built with the Intel compiler using `/Qlong-double`:
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
We need to understand how the Microsoft and Intel compilers implement `Data` allocation due to the different `long double` sizes:
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
We will explore alignment details in a later chapter. For now, remember that the Intel compiler uses 16-byte-aligned data for the `long double`, writing the meaningful data to the lower 10 bytes and padding the remaining 6. Being 16 bytes long, the `long double` will be stored at an address that is a multiple of 16 (an address ending in 0). As you can see from the first layout, the `long double` will be placed 16 bytes ahead of the base address to match the alignment. Typically, the base address is the address of the `Data` object itself, in this case equal to the address of its first member `a` (though this is not always the case).

You are free to use the library (Util.h and Util.lib) in a project you are developing using the Microsoft compiler. You include the header and link the library. Then create an object of type `Data` and pass it to the function `Compute`:
```cpp
#include <iostream>
#include "Util.h."

void main()
{
  Data data;

  std::cout << Compute(data) << std::endl;
}
```
The code will compile and link successfully. Problems arise at run time. This time, the Microsoft compiler allocates `Data`, and because the `long double` is still 64 bits (8 bytes), the compiler places that value at an address that is a multiple of 8. In the second layout above, `Data.b` is stored 8 bytes away from the base address. What will happen at run time is quite clear: data allocated by the Microsoft compiler will be processed by instructions generated by the Intel compiler. These instructions tell the CPU to read the `long double` at `&data+16`. At that address, the data is anything but the expected `long double`. A crash is not guaranteed, as there may be other readable data at higher addresses, but the CPU will inevitably read garbage and return an incorrect value. That is the first problem. The second problem is how the encoding of 1.0 differs between 64-bit and 80-bit:
```
Microsoft long double 64-bit = 3FF0000000000000
Intel /Qlong-double   80-bit = 3FFF8000000000000000
```
Even if we move the Microsoft `long double` to the proper location at `&data+16`, the FPU will not interpret it correctly. We can nevertheless attempt to fix it. The trick used here is valid for this specific case and for educational purposes only:
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
The code allocates data to a 16-byte boundary, and after setting up the first integer, it copies the 80-bit value to the proper locations, recreating the same Intel layout:
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
`a` `b` `c` hold the encoding of the 80-bit value 1.0. The FPU will read it correctly and return the proper result. This is a fortunate circumstance, however, because the function returns a 32-bit integer, which is always returned via the EAX register. This leads to the third problem: the return type. When the FPU is involved and the function returns a floating-point value, the callee leaves the value for the caller in ST0 rather than in the XMM0 register, as expected by the Microsoft compiler.

The `long double` example illustrates how different binaries built with different compilers can introduce problems. We are intentionally creating issues by manually changing the `long double` size; these dangers could have been foreseen. In normal conditions, these compilers are binary-compatible and free of such entanglements. Needless to say, there are dozens of other factors that make binaries built with different compilers (or even the same compiler) prone to breakage. We will see other examples throughout the article.

It is possible to query the size of built-in and user-defined objects using the `sizeof` operator at compile time. The compiler replaces the expression with a 64-bit unsigned integer:
```cpp
std::cout << sizeof(int) << " " << sizeof(long double) << " " << sizeof(Data) << std::endl;
```
For empty objects, the size is implementation-defined and has no particular semantic meaning. For the Microsoft compiler, the size of the following objects is equal to 1:
```cpp
struct Empty    //sizeof(Empty) = 1
{};

class Nothing   //sizeof(Nothing) = 1
{};
```
Parentheses for the `sizeof` operator are mandatory for built-in types but can be omitted for user-defined types:
```cpp
struct ABC
{
  int  x;
  char c;
};

std::cout << sizeof ABC << std::endl; //print "8"
```
