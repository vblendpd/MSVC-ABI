### NAME MANGLING
Name mangling (often referred to as symbol decoration) is a technique that encodes extra information into symbol names to make them unique for the compiler and linker. The evolution of the C++ language has led to an inevitable increase in the complexity of the toolset. C++ functions, unlike those in C, belong to various entities such as classes or namespaces. Things get even more complicated with overloading, inheritance, templates, and so on. A typical case that requires function names to be decorated is function overloading. Function overloading is the feature that permits multiple functions with the same name to coexist, as long as they have different parameter lists. Since the name alone does not uniquely identify a function, additional information (e.g., function parameters) is encoded together to create the ASCII string for the corresponding symbol name. That symbol name will be used by the linker to resolve references, among other things. Example:
```C++
extern "C" int GetValue();

void SetParam(int   i);
void SetParam(float f);

void Func(const std::string& str);
```
The first function `GetValue` is exported using C-linkage. The C ABI is well-defined and more stable; most tools are able to understand it, making it possible to bind C code with other languages as well. Since overloading, exceptions, inheritance, and templates are not available in C, C-linkage does not provide name decoration. In fact, a C compiler is not able to differentiate between the following two functions:
```C++
extern "C"
{
  void SetParam(int   i);
  void SetParam(float f);
}
```
```
'SetParam': you cannot overload a function with 'extern "C"' linkage
```
C++ compilers are able to decorate names instead. `SetParam` needs additional information to distinguish between the one that takes an integer parameter and the one that takes a floating-point one. The Microsoft compiler decorates the previous functions this way:
```
GetValue
?SetParam@@YAXH@Z
?SetParam@@YAXM@Z
?Func@@YAXAEBV?$basic_string@DU?$char_traits@D@std@@V?$allocator@D@2@@std@@@Z
```
The Microsoft mangling scheme is probably the most cryptic one. Different compilers implement different mangling schemes, and none of them is currently part of any official documentation. This does not seem like a huge problem at first, since we are only interested in the output (the executable, mostly) and not in how the compiler creates intermediate object files, etc. That is not entirely true — as we will see, knowing these details can help foresee potential issues that may arise when deploying libraries or SDKs, etc.\
Microsoft-compatible compilers, however, including Intel and LLVM Clang-cl, use the same technique to mangle names for binary compatibility. Linking a static library compiled with the Intel compiler to an executable built with the Microsoft compiler will work fine. The opposite is true if GCC was used for the same static library, because GCC uses a different mangling scheme. To illustrate this, consider the following GNU variant:
```
GetValue
_Z8SetParami
_Z8SetParamf
_Z4FuncRKNSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE
```
Linking a static library compiled with GCC to an executable built with the Microsoft toolset will not work:
```
unresolved external symbol "void __cdecl SetParam(int)" (?SetParam@@YAXH@Z)
unresolved external symbol "void __cdecl SetParam(float)" (?SetParam@@YAXM@Z)
unresolved external symbol "void __cdecl Func(class std::basic_string<char,struct...
```
The Microsoft linker is unable to find those symbols because they have been exported with different names it does not recognize. The machine code for each function is present in the library but cannot be located; thus, references are not resolved and function calls cannot be established. No errors are reported for `GetValue` since it uses C-linkage, which most tools (including Microsoft and GCC compilers) are able to understand. To locate `GetValue`, the linker performs the search using the plain name `GetValue` and not its corresponding decorated form `?GetValue@@YAHXZ`.\
This helps explain why it is recommended to compile external libraries (e.g., Boost, OpenCV) using the same compiler before using them in a project.

To view the list of symbols for an object file, use the `dumpbin` utility. Run `Visual Studio Command Prompt` and navigate to the object file directory to execute the command:
```
dumpbin /symbols file.obj
```
Name decoration can be quite complex, and studying all the details is time-consuming or even impractical. To understand what I mean, take a look at this decorated name:
```
??$Add@M@?$Vector@H@Math@@QEAAAEAV01@AEBV?$Vector@M@1@@Z 
```
To de-mangle this name you can use the utility `undname` from the Visual Studio toolset:
```
undname ??$Add@M@?$Vector@H@Math@@QEAAAEAV01@AEBV?$Vector@M@1@@Z
```
The output:
```
public: class Math::Vector<int> & __ptr64 __cdecl Math::Vector<int>::Add<float>(class Math::Vector<float> const & __ptr64) __ptr64
```
More human-friendly, but still not easy to reconstruct. `Vector` appears to be a template class, part of the `Math` namespace, and an instance of `Vector<int>` is calling the `Add` function taking a `Vector<float>` as a parameter:
```C++
namespace Math
{
  template<typename T>
  class Vector
  {
  public:
    template<typename U>
    Vector<T>& Add(const Vector<U>& v){...}
...
...
void main()
{
  Math::Vector<int> vi;
  Math::Vector<float> vf;

  vi.Add(vf);
}
```
Not straightforward. We will take a look at the mangling scheme anyway. We start with variables and then move on to functions, special functions (e.g., constructors and operators), and templates. We cannot cover all possible details and the following table may not be exhaustive. You may be able to find additional information in other online resources, but at the moment I am not aware of any article that provides in-depth coverage with examples for this topic.\
Let us start with some built-in and other codes:
```
bool                    _N
char                    D
signed char             D
unsigned char           E

short                   F
short int               F
signed short            F
signed short int        F
unsigned short          G
unsigned short int      G

int                     H
signed                  H
signed int              H
unsigned                I
unsigned int            I

long                    J
long int                J
signed long             J
signed long int         J
unsigned long           K
unsigned long int       K

long long               _J
long long int           _J
signed long long        _J
signed long long int    _J
unsigned long long      _K
unsigned long long int  _K

float                   M
double                  N
long double             O

__int8                  D
__int16                 F
__int32                 H
__int64                 _J
wchar_t                 _W
__wchar_t               _W
char8_t                 _Q
char16_t                _S
char32_t                _U

__m64                   T_m64@@
__m128                  T_m128@@
__m256                  T_m256@@
__m512                  T_m512@@
void                    X

union [NAME]             T[NAME]@[NAMESPACE]@
struct[NAME]             U[NAME]@[NAMESPACE]@
class [NAME]             V[NAME]@[NAMESPACE]@
enum  [NAME]            W4[NAME]@[NAMESPACE]@
                        
TYPE[]                  PA[TYPE]
const TYPE[]            QB[TYPE]
volatile TYPE[]         RC[TYPE]
const volatile TYPE[]   SD[TYPE]

TYPE[][N]               PAY
const TYPE[][N]         QAY
volatile TYPE[][N]      RAY
const vol TYPE[][N]     SAY

                        32-bit:   64-bit:
TYPE*                   PA[TYPE]  PEA[TYPE]

const TYPE*             PB[TYPE]  PEB[TYPE]
TYPE const*             PB[TYPE]  PEB[TYPE]

volatile TYPE*          PC[TYPE]  PEC[TYPE]
TYPE volatile*          PC[TYPE]  PEC[TYPE]

const volatile TYPE*    PD[TYPE]  PED[TYPE]
volatile const TYPE*    PD[TYPE]  PED[TYPE]
TYPE const volatile*    PD[TYPE]  PED[TYPE]
TYPE volatile const*    PD[TYPE]  PED[TYPE]

TYPE* const             QA[TYPE]  QEA[TYPE]

const TYPE* const       QB[TYPE]  QEB[TYPE]
TYPE const* const       QB[TYPE]  QEB[TYPE]

volatile TYPE* const    QC[TYPE]  QEC[TYPE]
TYPE volatile* const    QC[TYPE]  QEC[TYPE]

const vol. TYPE* const  QD[TYPE]  QED[TYPE]
vol. const TYPE* const  QD[TYPE]  QED[TYPE]
TYPE const vol.* const  QD[TYPE]  QED[TYPE]
TYPE vol. const* const  QD[TYPE]  QED[TYPE]

TYPE* volatile          RA[TYPE]  REA[TYPE]

const TYPE* volatile    RB[TYPE]  REB[TYPE]
TYPE const* volatile    RB[TYPE]  REB[TYPE]

volatile TYPE* volatile RC[TYPE]  REC[TYPE]
TYPE volatile* volatile RC[TYPE]  REC[TYPE]

const TYPE vol.* vol.   RD[TYPE]  RED[TYPE]
vol. const TYPE* vol.   RD[TYPE]  RED[TYPE]
TYPE const vol.* vol.   RD[TYPE]  RED[TYPE]
TYPE vol. const* vol.   RD[TYPE]  RED[TYPE]

TYPE* const volatile    SA[TYPE]  SEA[TYPE]

const TYPE* const vol.  SB[TYPE]  SEB[TYPE]
const TYPE* const vol.  SB[TYPE]  SEB[TYPE]

vol. TYPE* const vol.   SC[TYPE]  SEC[TYPE]
TYPE vol.* const vol.   SC[TYPE]  SEC[TYPE]

const TYPE vol* const vol  SD[TYPE]  SED[TYPE]
const vol TYPE* const vol  SD[TYPE]  SED[TYPE]
TYPE const vol* const vol  SD[TYPE]  SED[TYPE]
TYPE vol const* const vol  SD[TYPE]  SED[TYPE]

TYPE&                   AA[TYPE]  AEA[TYPE]
const TYPE&             AB[TYPE]  AEB[TYPE]
volatile TYPE&          AC[TYPE]  AEC[TYPE]
const volatile TYPE&    AD[TYPE]  AED[TYPE]

TYPE&&                $$QEA[TYPE] 
const TYPE&&          $$QEB[TYPE]
volatile TYPE&&       $$QEC[TYPE]
const vol TYPE&&      $$QED[TYPE]

TYPE&& volatile       $$REA[TYPE]
const TYPE&& vol      $$REB[TYPE]
vol TYPE&& vol        $$REC[TYPE]
const vol TYPE&& vol  $$RED[TYPE]

TYPE(*)(..)                P6
const TYPE(*)(..)          Q6
volatile TYPE(*)(..)       R6
const volatile TYPE(*)(..) Q6
```
We start with global variables as mentioned earlier. Two more tables are needed. The first is for the object qualifier and the second for the access control:
```
[OBJ_Q]

default        A
const          B
volatile       C
const volatile D
```
```
[ACCESS]

private    0
protected  1
public     2
default    3
```
The Microsoft compiler starts C++ names with a question mark `?` to distinguish them from C-linkage names.

###### GLOBAL OBJECTS
A symbol for a global object is formatted according to this rule:
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [TYPE] [PTR_REF 64-bit: E] [QUALIFIER]
```
For 64-bit pointer and reference types, the additional `E` is part of the name. It is also part of the [TYPE] to distinguish between 32-bit and 64-bit pointers and references (e.g., `PA` and `PEA` for 32-bit and 64-bit pointers respectively, according to the first table). For 32-bit symbols, this additional `E` is absent. For a global variable, the access code is always `3` since it is not a member of any object. Examples:
```
int a; ?a@@3HA

NAME      = a
NAMESPACE = -
ACCESS    = 3 (default)
TYPE      = H (int)
QUALIFIER = A (default)
```
```
const float PI; ?PI@@3MB

NAME      = PI
NAMESPACE = -
ACCESS    = 3 (default)
TYPE      = M (float)
QUALIFIER = B (const)
```
```
double value; ?value@@3NA

NAME      = value
NAMESPACE = -
ACCESS    = 3 (default)
TYPE      = N (double)
QUALIFIER = A (default)
```
```
volatile long number; ?number@@3JC
NAME      = number
NAMESPACE = -
ACCESS    = 3 (default)
TYPE      = J (long)
QUALIFIER = C (volatile)
```
```
namespace ns01
{
  unsigned long ul; ?ul@ns01@@3KA
};

NAME      = ul
NAMESPACE = ns01@
ACCESS    = 3 (default)
TYPE      = K (unsigned long)
QUALIFIER = A (default)
```
```
wchar_t wc; ?wc@@3_WA
NAME      = wc
NAMESPACE = -
ACCESS    = 3  (default)
TYPE      = _W (wchar_t)
QUALIFIER = A  (default)
```
```
volatile const char c; ?c@@3DD

NAME      = c
NAMESPACE = -
ACCESS    = 3 (default)
TYPE      = D (char)
QUALIFIER = D (const volatile)
```
```
namespace n1
{
  namespace sub1
  {
    char c; ?c@sub1@n1@@3DA

NAME      = c
NAMESPACE = sub1@n1
ACCESS    = 3 (default)
TYPE      = D (char)
QUALIFIER = A (default)

    const __int16 test; ?test@sub1@n1@@3FB

NAME      = test
NAMESPACE = sub1@n1
ACCESS    = 3 (default)
TYPE      = F (__int16)
QUALIFIER = B (const)
```
```
int* ptr_a;

           32-bit:        64-bit:
           ?ptr_a@@3PAHA  ?ptr_a@@3PEAHEA

NAME      = ptr_a          ptr_a
NAMESPACE = -              -
ACCESS    = 3              3
TYPE      = PAH            PEAH
PTR_REF   = -              E
QUALIFIER = A              A
```
```
namespace strings {
    const char* str1;

            32-bit:               64-bit:
            ?str1@strings@@3PBDB  ?str1@strings@@3PEBDEB

NAME      = str1                  str1
NAMESPACE = strings@              strings@
ACCESS    = 3                     3
TYPE      = PBD                   PEBD
PTR_REF   = -                     E
QUALIFIER = B                     B
```
```
const float* float_ptr;
           
            32-bit:            64-bit:
            ?float_ptr@@3PBMB  ?float_ptr@@3PEBMEB

NAME      = float_ptr          float_ptr
NAMESPACE = -                  -
ACCESS    = 3                  3
TYPE      = PBM                PEBM
PTR_REF   = -                  E
QUALIFIER = B                  B
```
```
const volatile int* test01;

            32-bit:         64-bit:
            ?test01@@3PDHD  ?test01@@3PEDHED

NAME      = test01          test01
NAMESPACE = -               -
ACCESS    = 3               3
TYPE      = PDH             PEDH
PTR_REF   = -               E
QUALIFIER = D               D
```
```
const int* const abc;

            32-bit:      64-bit:
            ?abc@@3QBHB  ?abc@@3QEBHEB

NAME      = abc          abc
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = QBH          QEBH
PTR_REF   = -            E
QUALIFIER = B            B
```
```
char& r1;
            32-bit:     64-bit:
            ?r1@@3AADA  ?r1@@3AEADEA

NAME      = r1          r1
NAMESPACE = -           -
ACCESS    = 3           3
TYPE      = AAD         AEAD
PTR_REF   = -           E
QUALIFIER = A           A
```
```
volatile int& r2;

            32-bit:    64-bit:
            ?r2@@ACHC  ?r2@@AECHEC

NAME      = r2         r2
NAMESPACE = -          -
ACCESS    = 3          3
TYPE      = ACH        AECH
PTR_REF   = -          E
QUALIFIER = C          C
```
```
namespace hello {
    const short& r3;
    
            32-bit:           64-bit:
            ?r3@hello@@3ABFB  ?r3@hello@@3AEBFEB

NAME      = r3                r3
NAMESPACE = hello@            hello@
ACCESS    = 3                 3
TYPE      = ABF               AEBF
PTR_REF   = -                 E
QUALIFIER = B                 B
```
```
struct Vector v; ?v@@3UVector@@A

NAME      = v
NAMESPACE = -
ACCESS    = 3
TYPE      = UVector@@
QUALIFIER = A
```
```
struct Vector* pv;
          
            32-bit:            64-bit:
            ?pv@@3PAUVector@@A ?pv@@3PEAUVector@@EA

NAME      = pv                 pv
NAMESPACE = -                  -
ACCESS    = 3                  3
TYPE      = PAUVector@@        PEAUVector@@
PTR_REF   = -                  E
QUALIFIER = A                  A
```
```
struct Vector& rv;
          
            32-bit:            64-bit:
            ?rv@@3AAUVector@@A ?rv@@3AEAUVector@@EA

NAME      = rv                 rv
NAMESPACE = -                  -
ACCESS    = 3                  3
TYPE      = AAUVector@@        AEAUVector@@
PTR_REF   = -                  E
QUALIFIER = A                  A
```
```
union um32; ?um32@@3Tum32@@A

NAME      = um32
NAMESPACE = -
ACCESS    = 3
TYPE      = Tum32@@
PTR_REF   = -
QUALIFIER = A
```
```
__m128 gv4f; ?gv4f@@3T__m128@@A

NAME      = gv4f
NAMESPACE = -
ACCESS    = 3
TYPE      = T__m128@@
QUALIFIER = A
```
```
const __m256 gv8f; ?gv8f@@3T__m256@@B

NAME      = gv8f
NAMESPACE = -
ACCESS    = 3
TYPE      = T__m256@@
QUALIFIER = B
```
```
namespace util{
    void* uz_ptr;
        
            32-bit:             64-bit:
            ?uz_ptr@util@@3PAXA ?uz_ptr@util@@3PEAXEA

NAME      = uz_ptr              uz_ptr
NAMESPACE = util@               util@
ACCESS    = 3                   3
TYPE      = PAX                 PEAX
PTR_REF   = -                   E
QUALIFIER = A                   A
```
The `Vector` structure encountered so far was declared at global scope, which is why we did not specify any namespace in `UVector@[NAMESPACE]@`. Leaving it empty corresponds to the global namespace. We know that a structure and its instances can be located in different namespaces. This is where the additional [NAMESPACE] comes into play. To understand this, let us look at the following example:
```C++
namespace N1 {
  struct Vector{...};
}

namespace N2 {
  struct Vector{...};
}

namespace util {
  N1::Vector v1;
  N2::Vector v2;
}
```
The following symbols are not valid:
```
?v1@util@@3UVector@@A
?v2@util@@3UVector@@A
```
From the mangled names, it is not clear which `Vector` `v1` and `v2` refer to. Is `v1` `N1::Vector` or `N2::Vector`? This information is encoded in the type itself in the [NAMESPACE] part of `UVector@[NAMESPACE]@`:
```
?v1@util@@3UVector@N1@@A
?v2@util@@3UVector@N2@@A
```
This way it is clear that `v1` is inside the `util` namespace but the `Vector` is the one from `N1`. Let us look at another example:
```C++
namespace math {
  struct Point{...};
    
  const Point pt;
}
```
This time both the structure and the object are declared within the same namespace. It is not the global namespace, so we expect `math` to appear twice:
```
?pt@math@@3UPoint@math@@B
```
This is logically correct — `pt` is inside `math` (`pt@math@@`) and the `Point` structure is also declared inside `math` (`3UPoint@math@@`). Unfortunately, this is not the proper symbol name: it is logically correct but ill-formed according to mangling rules. Dumping the symbols from the object file, the string is translated to this instead:
```
?pt@math@@3UPoint@1@B
```
There is a rule for repeating string tokens. We will take a closer look at this rule in the functions mangling section, as it is widely used for repeating function arguments. For now, just remember that in order to keep the symbol name as short as possible, the scheme performs a kind of token-labeling, using incrementing numbers to replace repeated text strings. Let us go back to our example. The first part of the symbol looks like this:
```
  0    1          2 
? pt @ math  @@3U Point...
```
A label starting from 0 is associated with every name already present in the string. The maximum number is 9, for reasons we will see in the functions section. Both symbols are logically correct, but the toolset uses the second one, replacing the second `math` token with `1`:
```
1) ?pt@math@@3UPoint@math@@B
2) ?pt@math@@3UPoint@1@B
```
Let us look at a couple of slightly harder examples:
```C++
namespace math {
  namespace geom {
    struct Point{...};
  }
  const volatile geom::Point point;
}
```
`point` is part of the `math` namespace, so the first piece of the string is clearly:
```
?point@math@@3UPoint
```
Now we need to figure out how to encode the namespace where `Point` is defined. The structure is `math::geom::Point`. Can we encode something with a number to save space? Yes, because we already have some tokens we can reuse:
```
  0       1         2
? point @ math @@3U Point
```
We can replace `math` with `1`. For `geom` we cannot do much other than writing it as a full string. Thus the namespace where `Point` is defined can be translated to `geom@1` (namespaces are encoded from innermost to outermost), and the ASCII string is:
```
?point@math@@3UPoint@geom@1@D
```
Let us try to mangle this:
```C++
namespace lib {
  namespace util  {
    enum Color{...};
    const Color bkg_color;
  }
}
```
Clearly, the first piece is:
```
?bkg_color@util@lib@@W4Color
```
Labeling each token, we get:
```
  0          1     2       3
? bkg_color@ util@ lib@@W4 Color
```
`Color` is defined in `lib::util`, so replacing each token with its corresponding index, we shrink `lib@util@` to `12`:
```
?bkg_color@util@lib@@W4Color@12@B
```
###### MEMBER OBJECTS
For member variables, the rules are the same. Since they are no longer global, the ACCESS code `3` is replaced with `0` `1` `2` for private, protected, and public respectively. The template is repeated here:
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [TYPE] [PTR_REF 64-bit: E] [QUALIFIER]
```
Let us start with simple built-in variables:
```
Class SomeClass
{
private:
static int a; ?a@SomeClass@@0HA

NAME      = a
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = H (int)
QUALIFIER = A (default)


static const float PI; ?PI@SomeClass@@0MB

NAME      = PI
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = M (float)
QUALIFIER = B (const)


static double value; ?value@SomeClass@@0NA

NAME      = value
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = N (double)
QUALIFIER = A (default)


static volatile long number; ?number@SomeClass@@0JC

NAME      = number
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = J (long)
QUALIFIER = C (volatile)


static unsigned long ul; ?ul@SomeClass@@0KA

NAME      = ul
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = K (unsigned long)
QUALIFIER = A (default)


static wchar_t wc; ?wc@SomeClass@@0_WA

NAME      = wc
NAMESPACE = SomeClass@
ACCESS    = 0  (private)
TYPE      = _W (wchar_t)
QUALIFIER = A  (default)


volatile const char c; ?c@SomeClass@@0DD

NAME      = c
NAMESPACE = SomeClass@
ACCESS    = 0 (private)
TYPE      = D (char)
QUALIFIER = D (const volatile)

protected:
static int* pInt;

            32-bit:                64-bit:
            ?pInt@SomeClass@@1PAHA ?pInt@SomeClass@@1PEAHEA

NAME      = pInt                   pInt
NAMESPACE = SomeClass@             SomeClass@
ACCESS    = 1                      1
TYPE      = PAH                    PEAH
PTR_REF   = -                      E
QUALIFIER = A                      A
          

static float m_ff;

            32-bit:              64-bit:
            ?m_ff@SomeClass@@1MA ?m_ff@SomeClass@@1MA

NAME      = m_ff                 m_ff
NAMESPACE = SomeClass@           SomeClass@
ACCESS    = 1                    1
TYPE      = M                    M
QUALIFIER = A                    A

public:
static int value;

            32-bit:               64-bit:
            ?value@SomeClass@@2HA ?value@SomeClass@@2HA
  
NAME      = value                 value
NAMESPACE = SomeClass@            SomeClass@
ACCESS    = 2                     2
TYPE      = H                     H
QUALIFIER = A                     A


static const char& refABC;

            32-bit:                  64-bit:
            ?refABC@SomeClass@@2ABDB ?refABC@SomeClass@@2AEBDEB

NAME      = refABC                   refABC
NAMESPACE = SomeClass@               SomeClass@
ACCESS    = 2                        2
TYPE      = ABD                      AEBD
PTR_REF   = -                        E
QUALIFIER = B                        B

private:
static const volatile int* test01;

            32-bit:                   64-bit:
            ?test01@SomeClass@@0PDHD  ?test01@SomeClass@@0PEDHED

NAME      = test01                   test01
NAMESPACE = SomeClass@               SomeClass@
ACCESS    = 0                        0
TYPE      = PDH                      PEDH
PTR_REF   = -                        E
QUALIFIER = D                        D


static volatile int& r2;

            32-bit:              64-bit:
            ?r2@SomeClass@@0ACHC ?r2@SomeClass@@0AECHEC

NAME      = r2                   r2
NAMESPACE = SomeClass@           SomeClass@
ACCESS    = 0                    0
TYPE      = ACH                  AECH
PTR_REF   = -                    E
QUALIFIER = C                    C

protected:
static const int* const abc;

            32-bit:               64-bit:
            ?abc@SomeClass@@1QBHB ?abc@SomeClass@@1QEBHEB

NAME      = abc                   abc
NAMESPACE = SomeClass@            SomeClass@
ACCESS    = 1                     1
TYPE      = QBH                   QEBH
PTR_REF   = -                     E
QUALIFIER = B                     B

public:
static const float* float_ptr;
           
            32-bit:                     64-bit:
            ?float_ptr@SomeClass@@2PBMB ?float_ptr@SomeClass@@2PEBMEB

NAME      = float_ptr                   float_ptr
NAMESPACE = SomeClass@                  SomeClass@
ACCESS    = 2                           2
TYPE      = PBM                         PEBM
PTR_REF   = -                           E
QUALIFIER = B                           B


static char& r1;
            32-bit:              64-bit:
            ?r1@SomeClass@@2AADA ?r1@SomeClass@@2AEADEA

NAME      = r1                   r1
NAMESPACE = SomeClass@           SomeClass@
ACCESS    = 2                    2
TYPE      = AAD                  AEAD
PTR_REF   = -                    E
QUALIFIER = A                    A
```
Let us now include some namespaces and structures.
```
class Something
{
  struct Vector{...};

public:
static Vector x; ?x@Something@@2UVector@1@A

NAME      = x  
NAMESPACE = Something@
ACCESS    = 2
TYPE      = UVector@1@ (@1 = Something@)
QUALIFIER = A

private:
static Vector* pv;
          
            32-bit:                       64-bit:
            ?pv@Something@@0PAUVector@1@A ?pv@Something@@0PEAUVector@1@EA

NAME      = pv                            pv
NAMESPACE = Something@                    Something@
ACCESS    = 0                             0
TYPE      = PAUVector@1@                  PEAUVector@1@
PTR_REF   = -                             E
QUALIFIER = A                             A
```
```
class Hello
{
  struct Point{...};

protected:
static Point& ref_pt;
          
            32-bit:                        64-bit:
            ?ref_pt@Hello@@1AAUPoint@1@A   ?ref_pt@Hello@@1AEAUPoint@1@EA

NAME      = ref_pt                         ref_pt
NAMESPACE = Hello@                         Hello@
ACCESS    = 1                              1
TYPE      = AAUPoint@1@                    AEAUPoint@1@
PTR_REF   = -                              E
QUALIFIER = A                              A
```
```
union um32 {...};

class Utility
{
private:
static um32 m_u32; ?m_u32@Utility@@0Tum32@@A

NAME      = m_u32
NAMESPACE = Utility@
ACCESS    = 0
TYPE      = Tum32@@ (no namespace @@ = defined at global scope)
QUALIFIER = A

protected:
static __m128 v4f; ?v4f@Utility@@1T__m128@@A

NAME      = v4f
NAMESPACE = Utility@
ACCESS    = 1
TYPE      = T__m128@@
QUALIFIER = A

public:
static const __m256 v8f; ?v8f@Utility@@2T__m256@@B

NAME      = v8f
NAMESPACE = Utility@
ACCESS    = 2
TYPE      = T__m256@@
QUALIFIER = B
```
```
namespace N1 {
  struct Vector{...};
}

namespace N2 {
  struct Vector{...};
}

namespace math {
  struct Point{...};

  namespace geom {
    struct Line{...};
  }
}

namespace collection {
  struct temp_coll{...};

class Tree
{
public:
static temp_coll tcoll; ?tcoll@Tree@collection@@2Utemp_coll@2@A

NAME      = tcoll
NAMESPACE = Tree@collection@
ACCESS    = 2 
TYPE      = Utemp_coll@2
QUALIFIER = A

protected:
static N1::Vector m_v1; ?m_v1@Tree@collection@@1UVector@N1@@A

NAME      = m_v1
NAMESPACE = Tree@collection@
ACCESS    = 1
TYPE      = UVector@N1@
QUALIFIER = A


static N2::Vector m_v2; ?m_v2@Tree@collection@@1UVector@N2@@A

NAME      = m_v2
NAMESPACE = Tree@collection@
ACCESS    = 1
TYPE      = UVector@N2@
QUALIFIER = A

private:
static const math::Point pt; ?pt@Tree@collection@@0UPoint@math@@B

NAME      = pt
NAMESPACE = Tree@collection@
ACCESS    = 0
TYPE      = UPoint@math@
QUALIFIER = B


static volatile math::geom::Line m_line; ?m_line@Tree@collection@@0ULine@geom@math@@C

NAME      = m_line
NAMESPACE = Tree@collection@
ACCESS    = 0
TYPE      = ULine@geom@math@@
QUALIFIER = C
```
###### ARRAYS
Moving forward, we can take a look at arrays. Arrays follow the same mechanisms, but they deal with dimensions, and numbers have special rules. If N is in the range [1, 10], it will be translated to N-1. Numbers greater than 10 are converted to hexadecimal first, and after removing any leading `0`, each digit is translated according to the following mapping and terminated with `@`:
```
 0 -> A
 1 -> B
 2 -> C
 3 -> D
 4 -> E
 5 -> F
 6 -> G
 7 -> H
 8 -> I
 9 -> J
10 -> K
11 -> L
12 -> M
13 -> N
14 -> O
15 -> P
```
For example, the number 20 is equal to 14h. From the table above, 1->B and 4->E, so it translates to `BE@`. The number 88 is 58h in hexadecimal; 5->F and 8->I, hence 88 = `FI@`.

The ASCII string for arrays is constructed using this template:
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [TYPE] [QUALIFIER]
```
We start again with global variables, where the ACCESS is still `3`:
```
int a0[]; ?a0@@3PAHA

NAME      = a0
NAMESPACE = -
ACCESS    = 3
TYPE      = PAH
QUALIFIER = A


const short cso[]; ?cso@@3QBFB

NAME      = cso
NAMESPACE = -
ACCESS    = 3
TYPE      = QBF
QUALIFIER = B


const int a2[]; ?a2@@3QBHB

NAME      = a2
NAMESPACE = -
ACCESS    = 3
TYPE      = QBH
QUALIFIER = B


volatile bool vba[]; ?vba@@3RC_NC

NAME      = vba
NAMESPACE = -
ACCESS    = 3
TYPE      = RC_N
QUALIFIER = C


const int a3[]; ?a3@@3QBHB

NAME      = a3
NAMESPACE = -
ACCESS    = 3
TYPE      = QBH
QUALIFIER = B


const char a6[]; ?a6@@3QBDB

NAME      = a6
NAMESPACE = -
ACCESS    = 3
TYPE      = QBD
QUALIFIER = B


const volatile long test[]; ?test@@3SDJD

NAME      = test
NAMESPACE = -
ACCESS    = 3
TYPE      = SDJ
QUALIFIER = D


int arr1[4]; ?arr1@@3PAHA

NAME      = arr1
NAMESPACE = -
ACCESS    = 3
TYPE      = PAH
QUALIFIER = A


const char arr2[100]; ?arr2@@3QBDB

NAME      = arr2
NAMESPACE = -
ACCESS    = 3
TYPE      = QBD
QUALIFIER = B


volatile char c2[2]; ?c2@@3RCDC

NAME      = c2
NAMESPACE = -
ACCESS    = 3
TYPE      = RCD
QUALIFIER = C


char8_t c8ta[]; ?c8ta@@3PA_QA

NAME      = c8ta
NAMESPACE = -
ACCESS    = 3
TYPE      = PA_Q
QUALIFIER = A
```
###### ARRAYS OF POINTERS
```
int* ptr_a[2];
            32-bit:         64-bit:
            ?ptr_a@@3PAPAHA ?ptr_a@@3PAPEAHA
NAME      = ptr_a           ptr_a
NAMESPACE = -               -
ACCESS    = 3               3
TYPE      = PAPAH           PAPEAH
QUALIFIER = A               A


const int* a4[]

            32-bit:      64-bit:
           ?a4@@3PAPBHA  ?a4@@3PAPEBHA
NAME      = a4           a4
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = PAPBH        PAPEBH
QUALIFIER = A            A


const int* const a5[]

            32-bit:      64-bit:
            ?a5@@3QBQBHB ?a5@@3QBQEBHB
NAME      = a5           a5
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = QBQBH        QBQEBH
QUALIFIER = B            B


int* a1[]
            32-bit:      64-bit:
            ?a1@@3PAPAHA ?a1@@3PAPEAHA
        
NAME      = a1           a1
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = PAPAH        PAPEAH
QUALIFIER = A            A


int* const a6[]
            32-bit:      64-bit:
            ?a6@@3QBQAHB ?a6@@3QBQEAHB
NAME      = a6           a6
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = QBQAH        QBQEAH
QUALIFIER = B            B

char* volatile a7[]
            32-bit:        64-bit:
            ?a7@@3RCRADC   ?a7@@3RCREADC
NAME      = a7             a7
NAMESPACE = -              -
ACCESS    = 3              3
TYPE      = RCRАД          RCREAD
QUALIFIER = C              C

volatile float* a8[]
            32-bit:      64-bit:
            ?a8@@3PAPCMA ?a8@@3PAPECMA
NAME      = a8           a8
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = PAPCM        PAPECM
QUALIFIER = A            A

volatile char* volatile a9[]
            32-bit:      64-bit:
            ?a9@@3RCRCDC ?a9@@3RCRECDC
NAME      = a9           a9
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = RCRCD        RCRECD
QUALIFIER = C            C

int* const volatile kk[]
            32-bit:      64-bit:
            ?kk@@3SDSAHD ?kk@@3SDSEAHD
NAME      = kk           kk
NAMESPACE = -            -
ACCESS    = 3            3
TYPE      = SDSAD        SDSEAD
QUALIFIER = D            D

const volatile short* p[]
            32-bit:     64-bit:
            ?p@@3PAPDFA ?p@@3PAPEDFA
NAME      = p           p
NAMESPACE = -           -
ACCESS    = 3           3
TYPE      = PAPDF       PAPEDF
QUALIFIER = A           A
```
###### ARRAYS OF STRUCTURES
```
namespace t1 {
  Vector vec[4];
}

?vec@t1@@3PAUVector@@A

NAME      = vec
NAMESPACE = t1@
ACCESS    = 3
TYPE      = PAUVector@@
QUALIFIER = A


const Color c[2]; ?c@@3QBW4Color@@B

NAME      = c
NAMESPACE = -
ACCESS    = 3
TYPE      = QBW4Color
QUALIFIER = B


namespace collection {
    Vector* pv[2];
}
            32-bit:                          64-bit:
            ?pv@collection@@3PAPAUVector@@A  ?pv@collection@@3PAPEAUVector@@A

NAME      = pv                               pv
NAMESPACE = collection@                      collection@
ACCESS    = 3                                3
TYPE      = PAPAUVector@@                    PAPEAUVector@@
QUALIFIER = A                                A


const Vector* d[2];

            32-bit:               64-bit:
            ?d@@3PAPBUVector@@A   ?d@@3PAPEBUVector@@A

NAME      = d                     d
NAMESPACE = -                     -
ACCESS    = 3                     3
TYPE      = PAPBUVector@@         PAPEBUVector@@
QUALIFIER = A                     A



const Vector* const a1[2];
            32-bit:              64-bit:
            ?a1@@3QBQBUVector@@B ?a1@@3QBQEBUVector@@B

NAME      = a1                   a1
NAMESPACE = -                    -
ACCESS    = 3                    3
TYPE      = QBQBUVector@@        QBQEBUVector@@
QUALIFIER = B                    B



volatile const Vector* a2[2]
            32-bit:              64-bit:
            ?a2@@3PAPDUVector@@A ?a2@@3PAPEDUVector@@A

NAME      = a2                   a2
NAMESPACE = -                    -
ACCESS    = 3                    3
TYPE      = PAPDUVector@@        PAPEDUVector@@
QUALIFIER = A                    A


Vector* volatile const a3[2];
            32-bit:               64-bit:
            ?a3@@3SDSAUVector@@D  ?a3@@3SDSEAUVector@@D

NAME      = a3                    a3
NAMESPACE = -                     -
ACCESS    = 3                     3
TYPE      = SDSAUVector@@         SDSEAUVector@@
QUALIFIER = D                     D


Color a4[2]; ?a4@@3PAVColor@@A

NAME      = a4
NAMESPACE = -
ACCESS    = 3
TYPE      = PAVColor@@
QUALIFIER = A


const Color a5[2]; ?a5@@3QBVColor@@B

NAME      = a5
NAMESPACE = -
ACCESS    = 3
TYPE      = QBVColor@@
QUALIFIER = B


volatile Color a6[2]; ?a6@@3RCVColor@@C

NAME      = a6
NAMESPACE = -
ACCESS    = 3
TYPE      = RCVColor@@
QUALIFIER = C


volatile const Color* const a7[2];
            32-bit:             64-bit:
            ?a7@@3QBQDVColor@@B ?a7@@3QBQEDVColor@@B
NAME      = a7                  a7
NAMESPACE = -                   -
ACCESS    = 3                   3
TYPE      = QBQDVColor@@        QBQEDVColor@@
QUALIFIER = B                   B
```
###### MULTIDIMENSIONAL ARRAYS
For multidimensional arrays, the string is composed based on the following rule:
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [MULTI_DIM_T] [NUM_DIM-2] [DIM_2, DIM_3...] [AEQ] [TYPE] [QUALIFIER]
```
We introduce another table for the multidimensional type (MULTI_DIM_T):
```
PAY (default)
QAY (const)
RAY (volatile)
SAY (const volatile)
```
Following is the total number of dimensions minus 2. You can think of this number as "how many dimensions more than a 1-dimensional array", since an array has at least 1 dimension.

After this, a list of numbers identifies the size of each dimension. The first dimension size is not included because it is not needed — and if you are reading this, you should already know why (we will revisit this when studying the object memory layout for arrays).

Next to the list of sizes, we find the multidimensional array element qualifier [AEQ]:
```
$$CA (for default and it's omitted)
$$CB (const)
$$CC (volatile)
$$CD (const volatile)
```
Type and qualifier at the end are still the same as seen so far.\
Example:
```
float matrix[4][4];
```
The matrix has 2 dimensions, so 2-1 = 1, and the number is translated to N-1 = 1-1 = 0. Skipping the first dimension, which is not taken into account, the second dimension is 4 and in the [1, 10] range, so the number translates to N-1 = 4-1 = 3. The matrix name will be:
```
float matrix[4][4]; ?matrix@@3PAY03MA

NAME      = matrix
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = PAY
NUM_DIM-2 = 2-2 = 0 (or 2-1 -> 1-1)
DIM_2     = 4-1 = 3
TYPE      = M
QUALIFIER = A
```
More examples with numbers greater than 10:
```
float matrix2[][12]; ?matrix2@@3PAY0M@MA

NAME      =  matrix2
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = PAY
NUM_DIM-2 = 2-2 = 0
DIM_2     = 12 = M@
TYPE      = M
QUALIFIER = A
```
```
int f[][9][10]; ?f@@3PAY189HA

NAME      = f
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = PAY
NUM_DIM-2 = 3-2 = 1
DIM_2     = 9-1 = 8
DIM_3     = 10-1 = 9
TYPE      = H
QUALIFIER = A
```
```
char strange[][20][30][40]; ?strange@@3PAY2BE@BO@CI@DA

NAME      = strange
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = PAY
NUM_DIM-2 = 4-2 = 2
DIM_2     = 20 = 14h = BE@
DIM_3     = 30 = 1Eh = BO@
DIM_4     = 40 = 28h = CI@
TYPE      = D
QUALIFIER = A
```
```
const int a1[4][4]; ?a1@@3QAY03$$CBHA (int const (* a1)[4])

NAME      = a1
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = QAY (const)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 4-1 = 3
TYPE      = $$CB H (const)
QUALIFIER = A
```
```
volatile int a2[4][4]; ?a2@@3RAY03$$CCHA (int volatile (* a2)[4])

NAME      = a2
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = RAY (volatile)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 4-1 = 3
TYPE      = $$CC H (volatile)
QUALIFIER = A
```
```
const volatile int a3[4][10]; ?a3@@3SAY09$$CDHA (int const volatile (* a3)[4])

NAME      = a3
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = SAY (const volatile)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 10-1 = 9
TYPE      = $$CD H (const volatile)
QUALIFIER = A
```
```
Vector v1[4][64][4]; ?v1@@3PAY1EA@3UVector@@A

NAME      = v1
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = PAY
NUM_DIM-2 = 3-2 = 1
DIM_2     = 64 = 40h = EA@
DIM_3     = 4-1 = 3
TYPE      = UVector@@
QUALIFIER = A
```
```
const Vector v2[2][2]; ?v2@@3QAY01$$CBUVector@@A(struct Vector const (*a2)[2])

NAME      = v2
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = QAY (const)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 2-1 = 1
TYPE      = $$CB UVector@@ (const)
QUALIFIER = A
```
```
volatile Vector v3[2][2]; ?v3@@3RAY01$$CCUVector@@A(struct Vector volatile (*a3)[2])

NAME      = v3
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = RAY (volatile)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 2-1 = 1
TYPE      = $$CC UVector@@ (volatile)
QUALIFIER = A
```
```
const volatile Vector v4[2][2]; ?v4@@3SAY01$$CDUVector@@A(struct Vector const volatile (*a4)[2])
NAME      = v4
NAMESPACE = -
ACCESS    = 3
MULTI_DIM = SAY (const volatile)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 2-1 = 1
TYPE      = $$CD UVector@@ (const volatile)
QUALIFIER = A
```
###### MEMBER ARRAYS
For member arrays, replace the ACCESS code `3` (used for global/non-member variables) with `0` `1` `2` for `private` `protected` `public` respectively.
```
namespace Util
{
  class SomeClass
  {
public:
long long arr[][20][30][1024]; ?arr@SomeClass@Util@@2PAY2BE@BO@EAA@_JA

NAME      = arr
NAMESPACE = SomeClass@Util@
ACCESS    = 2
MULTI_DIM = PAY
NUM_DIM-2 = 4-2 = 2
DIM_2     = 20 = 14h = BE@
DIM_3     = 30 = 1Eh = BO@
DIM_4     = 1024 = 400h = EAA@
TYPE      = _J
QUALIFIER = A


int* p[10][256][32];
            32-bit:                            64-bit:
            ?p@SomeClass@Util@@2PAY1BAA@CA@PAHA ?p@SomeClass@Util@@2PAY1BAA@CA@PEAHA

NAME      = p                                    p
NAMESPACE = SomeClass@Util@                      SomeClass@Util@
ACCESS    = 2                                    2
MULTI_DIM = PAY                                  PAY
NUM_DIM-2 = 3-2 = 1                              1
DIM_2     = 256 = 100h =  BAA@                   BAA@
DIM_3     = 32  =  20h =  CA@                    CA@
TYPE      = PAH                                  PEAH
QUALIFIER = A                                    A


const float* cr[32][32][32];
            32-bit:                             64-bit:
            ?cr@SomeClass@Util@@2PAY1CA@CA@PBMA ?cr@SomeClass@Util@@2PAY1CA@CA@PEBMA

NAME      = cr                   cr
NAMESPACE = SomeClass@Util@      SomeClass@Util@
ACCESS    = 2                    2
MULTI_DIM = PAY                  PAY
NUM_DIM-2 = 3-2 = 1              1
DIM_2     = 32 = 20h = CA@       CA@
DIM_3     = 32 = 20h = CA@       CA@
TYPE      = PBM                  PEBM
QUALIFIER = A                    A
```
```
class Test
{
public:
static int* ptr_arr[10][256][32];
            32-bit:                         64-bit:
            ?ptr_arr@Test@@2PAY1BAA@CA@PAHA ?ptr_arr@Test@@2PAY1BAA@CA@PEAHA

NAME      = ptr_arr                         ptr_arr
NAMESPACE = Test@                           Test@
ACCESS    = 2                               2
MULTI_DIM = PAY                             PAY
NUM_DIM-2 = 3-2 = 1                         1
DIM_2     = 256 = 100h = BAA@               BAA@
DIM_3     = 32  =  20h = CA@                CA@
TYPE      = PAH                             PEAH
QUALIFIER = A                               A

protected:
static const float* cr[32][32][32];
           32-bit:                   64-bit:
           ?cr@Test@@1PAY1CA@CA@PBMA ?cr@Test@@1PAY1CA@CA@PEBMA

NAME      = cr                        cr
NAMESPACE = Test@                     Test@
ACCESS    = 1                         1
MULTI_DIM = PAY                       PAY
NUM_DIM-2 = 3-2 = 1                   1
DIM_2     = 32 = 20h = CA@            CA@
DIM_3     = 32 = 20h = CA@            CA@
TYPE      = PBM                       PEBM
QUALIFIER = A                         A
```
```
namespace N1 {
    class Color {...};
}

class MyClass
{
public:
Vector v1[2]; ?v1@MyClass@@2PAUVector@@A

NAME      = v1
NAMESPACE = MyClass@
ACCESS    = 2
TYPE      = PAUVector@@
QUALIFIER = A


volatile Vector const v2[4][4]; ?v2@MyClass@@2SAY03$$CDUVector@@A

NAME      = v2
NAMESPACE = MyClass@
ACCESS    = 2
MULTI_DIM = SAY
NUM_DIM-2 = 2-2 = 0
DIM_2     = 4-1 = 3
TYPE      = $$CD UVector@@ (const volatile)
QUALIFIER = A


private:
const N1::Color m_cr[10][32]; ?m_cr@MyClass@@0QAY0CA@$$CBVColor@N1@@A

NAME      = m_cr
NAMESPACE = MyClass@
ACCESS    = 0
MULTI_DIM = QAY (const)
NUM_DIM-2 = 2-2 = 0
DIM_2     = 32 = 20h = CA@
TYPE      = $$CB VColor@N1@ (const)
QUALIFIER = A


int* const c1[2][3][4];
            32-bit:               64-bit:
            ?c1@MyClass@@2QAY123QAHA ?c1@MyClass@@2QAY123QEAHA

NAME      = c1                    c1
NAMESPACE = MyClass@              MyClass@
ACCESS    = 2                     2
MULTI_DIM = QAY                   QAY
NUM_DIM-2 = 3-2 = 1               1
DIM_2     = 3-1 = 2               2
DIM_3     = 4-1 = 3               3
TYPE      = QAH                   QEAH
QUALIFIER = A                     A


int* const volatile c2[][10][10];

            32-bit:               64-bit:
            ?c2@MyClass@@2SAY199SAHA ?c2@MyClass@@2SAY199SEAHA

NAME      = c2                    c2
NAMESPACE = MyClass@              MyClass@
ACCESS    = 2                     2
MULTI_DIM = SAY                   SAY
NUM_DIM-2 = 3-2 = 1               1
DIM_2     = 10-1 = 9              9
DIM_3     = 10-1 = 9              9
TYPE      = SAH                   SEAH
QUALIFIER = A                     A


const int* const c3[2][3][4];

            32-bit:               64-bit:
            ?c3@MyClass@@2QAY123QBHA ?c3@MyClass@@2QAY123QEBHA

NAME      = c3                    c3
NAMESPACE = MyClass@              MyClass@
ACCESS    = 2                     2
MULTI_DIM = QAY                   QAY
NUM_DIM-2 = 3-2 = 1               1
DIM_2     = 3-1 = 2               2
DIM_3     = 4-1 = 3               3
TYPE      = QBH                   QEBH
QUALIFIER = A                     A


volatile N1::Color* const c4[2][4][8][12];

            32-bit:                            64-bit:
            ?c4@MyClass@@2QAY237M@QCVColor@N1@@A ?c4@MyClass@@2QAY237M@QECVColor@N1@@A

NAME      = c4                                c4
NAMESPACE = MyClass@                          MyClass@
ACCESS    = 2                                 2
MULTI_DIM = QAY                               QAY
NUM_DIM-2 = 4-2 = 2                           2
DIM_2     = 4-1 = 3                           3
DIM_3     = 8-1 = 7                           7
DIM_4     = 12 = M@                           M@
TYPE      = QCVColor@N1@                      QECVColor@N1@
QUALIFIER = A                                 A


const volatile Vector* const volatile c5[2][256];

            32-bit:                         64-bit:
            ?c5@MyClass@@2SAY0BAA@SDUVector@@A ?c5@MyClass@@2SAY0BAA@SEDUVector@@A

NAME      = c5                               c5
NAMESPACE = MyClass@                         MyClass@
ACCESS    = 2                                2
MULTI_DIM = SAY                              SAY
NUM_DIM-2 = 2-2 = 0                          0
DIM_2     = 256 = 100h = BAA@                BAA@
TYPE      = SDUVector@@                      SEDUVector@@
QUALIFIER = A                                A

}
```
###### FUNCTIONS
Time to deal with functions. Two more tables are needed for translation: the return object qualifier and calling convention codes:
```
[RET_Q]

default         ?A
const           ?B
volatile        ?C
const volatile  ?D
```
```
[CALL_CONV]

__cdecl       A
__stdcall     G
__thiscall    E (A for 64-bit build)
__fastcall    I (A for 64-bit build)
__vectorcall  Q
```
The format for a global function is:
```
? [NAME] @ [NAMESPACE] @ Y [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
Function call distance is `Y` for near and `Z` for far. Since we are no longer in 16-bit Windows, calls are `Y` only. The return object qualifier is omitted when returning simple built-in types (only if non-const and non-volatile) and is always present for classes, structures, and unions. For an empty parameter list (void), the termination is simply `XZ`; otherwise the list of parameters is terminated with `@` to mark the end of the parameter list. Examples:
```
bool CheckValue();

32-bit __cdecl: ?CheckValue@@YA_NXZ

NAME      = CheckValue
NAMESPACE = -
CALL_CONV = A  (__cdecl)
RET_Q     = -
RET_TYPE  = _N (bool)
PARAMS    = X  (void)
TERMIN    = Z

32-bit __stdcall: ?CheckValue@@YG_NXZ

NAME      = CheckValue
NAMESPACE = -
CALL_CONV = G  (__stdcall)
RET_Q     = -
RET_TYPE  = _N (bool)
PARAMS    = X  (void)
TERMIN    = Z


64-bit __fastcall: ?CheckValue@@YA_NXZ

NAME      = CheckValue
NAMESPACE = -
CALL_CONV = A  (default __fastcall)
RET_Q     = -
RET_TYPE  = _N (bool)
PARAMS    = X  (void)
TERMIN    = Z

```
```
int DoSomething(int, int);
32-bit __cdecl: ?DoSomething@@YAHHH@Z
NAME      = DoSomething
NAMESPACE = -
CALL_CONV = A  (__cdecl)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H H (int, int)
TERMIN    = @Z


32-bit __stdcall: ?DoSomething@@YGHHH@Z
NAME      = DoSomething
NAMESPACE = -
CALL_CONV = G  (__stdcall)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H H (int, int)
TERMIN    = @Z


64-bit __fastcall: ?DoSomething@@YAHHH@Z
NAME      = DoSomething
NAMESPACE = -
CALL_CONV = A  (default __fastcall)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H H (int, int)
TERMIN    = @Z

```
```
const float Compute(int, float);
32-bit __cdecl: ?Compute@@YA?BMHM@Z

NAME      = Compute
NAMESPACE = -
CALL_CONV = A   (__cdecl)
RET_Q     = ?B  (const)
RET_TYPE  = M   (float)
PARAMS    = H M (int float)
TERMIN    = @Z


32-bit __stdcall: ?Compute@@YG?BMHM@Z

NAME      = Compute
NAMESPACE = -
CALL_CONV = G   (__stdcall)
RET_Q     = ?B  (const)
RET_TYPE  = M   (float)
PARAMS    = H M (int float)
TERMIN    = @Z


64-bit __fastcall: ?Compute@@YA?BMHM@Z

NAME      = Compute
NAMESPACE = -
CALL_CONV = A   (default __fastcall)
RET_Q     = ?B  (const)
RET_TYPE  = M   (float)
PARAMS    = H M (int float)
TERMIN    = @Z

```
```
Vector Get(int);

32-bit __cdecl: ?Get@@YA?AUVector@@H@Z

NAME      = Get
NAMESPACE = -
CALL_CONV = A  (__cdecl)
RET_Q     = ?A (default)
RET_TYPE  = UVector@@
PARAMS    = H  (int)
TERMIN    = @Z


32-bit __stdcall: ?Get@@YG?AUVector@@H@Z

NAME      = Get
NAMESPACE = -
CALL_CONV = G  (__stdcall)
RET_Q     = ?A (default)
RET_TYPE  = UVector@@
PARAMS    = H  (int)
TERMIN    = @Z


64-bit __fastcall: ?Get@@YA?AUVector@@H@Z

NAME      = Get
NAMESPACE = -
CALL_CONV = A  (default __fastcall)
RET_Q     = ?A (default)
RET_TYPE  = UVector@@
PARAMS    = H  (int)
TERMIN    = @Z

```
```

int F3(int, int*, const char&)
32-bit __cdecl: ?F3@@YAHHPAHABD@Z

NAME      = F3
NAMESPACE = -
CALL_CONV = A  (__cdecl)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H PAH ABD (int int* const char&)
TERMIN    = @Z


32-bit __stdcall: ?F3@@YGHHPAHABD@Z

NAME      = F3
NAMESPACE = -
CALL_CONV = G  (__stdcall)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H PAH ABD (int int* const char&)
TERMIN    = @Z


64-bit __fastcall: ?F3@@YAHHPEAHAEBD@Z

NAME      = F3
NAMESPACE = -
CALL_CONV = A  (default __fastcall)
RET_Q     = -
RET_TYPE  = H (int)
PARAMS    = H PEAH AEBD (int int* const char&)
TERMIN    = @Z

```
From this point on, only 64-bit functions will be analyzed.
```
namespace Util {
  struct Vector{...};
}

bool CheckVector(const Vector&, int, int) ?CheckVector@@YA_NAEBUVector@Util@@HH@Z

NAME      = CheckVector
NAMESPACE = -
CALL_CONV = A
RET_Q     = -
RET_TYPE  = _N (bool)
PARAMS    = AEBUVector@Util@ H H
TERMIN    = @Z
```
```
namespace Util {
  struct Vector{...};
}

union um64{...};

using namespace Util;

const Vector NonSense(const float*, um64&); ?NonSense@@YA?BUVector@Util@@PEBMAEATum64@@@Z

NAME      = NonSense
NAMESPACE = -
CALL_CONV = A
RET_Q     = ?B (const)
RET_TYPE  = UVector@Util@
PARAMS    = PEBM AEATum64@@
TERMIN    = @Z
```
The mangled string is intended to be as short as possible. For repeating parameters there are special rules that shrink the name even further. We covered the basics of this mechanism when composing the name for structures that belong to different namespaces, replacing tokens with identifiers wherever possible. Here we take a closer look at these details because repeating parameters are subject to similar rules.\
First, we need to understand the difference between named and unnamed parameters. Named parameters are parameters that have a name — for example, structures or classes. So `struct Vector` → `Vector` is a name; `enum Color` → `Color` is a name. The rest are classified as unnamed parameters.\
The symbol string can be constructed in successive passes. The first pass processes unnamed parameters and the second processes named ones. Parameters that are encoded with a single character (e.g., `int` = H) are obviously translated to their corresponding encoding character, since 1 character is already the minimum. On the other hand, a parameter like `int*` encoded as `PEAH` can be replaced by a number for subsequent occurrences. Let us begin with an example:
```C++
struct Vec;
enum class Color;

int Func(bool, int*, int*, float&, Vec&, Vec, Color, const Color&, const Color& c3, float&, const Vec&);
```
There are several repeating parameters such as `int*` or `float&`, as well as many repeating names (`Color` and `Vec`). The first step is to associate a number starting from `0` with each named and unnamed parameter that requires more than 1 character for encoding and has not yet been numbered:
```
     0     1           2       3     4    5      6                                   7
Func(bool, int*, int*, float&, Vec&, Vec, Color, const Color&, const Color&, float&, const Vec&)
```
You see the second `int*` is not associated with any number because it was already covered by the first `int*` = 1. The same applies to the second `const Color&`, already covered by number 5, and the last `float&`, already identified by 2. There are no parameters encoded with a single character, so we can proceed to translate the first occurrences of repeating parameters. To avoid confusion, I write the translation below the function:
```
     0     1           2       3       4    5       6                                   7
Func(bool, int*, int*, float&, Vec&,   Vec, Color,  const Color&, const Color&, float&, const Vec&)
    _N     PEAH        AEAM    AEAUVec      W4Color
```
Even though `Vec`, `Vec&`, and `const Vec&` are all different named parameters because they are encoded as `UVec`, `AEAUVec`, and `AEBUVec` respectively, they share the same name `Vec`, so we translate only the first occurrence of that name. The next step is to map indices (always starting from `0`) to names. This uses the mangled tokens we have so far:
```
?Func@@...Vec...Color....
 [0]      [1]    [2]
```
Yes, the function name belongs to the list. No references or pointers are used for named parameters — just the names. We can write these again below the function to avoid confusion:
```
     0     1           2       3       4    5       6                                   7
Func(bool, int*, int*, float&, Vec&,   Vec, Color,  const Color&, const Color&, float&, const Vec&)
    _N     PEAH        AEAM    AEAUVec      W4Color
[0]                                [1]        [2]
```
Every parameter that includes any token from the previous list can replace that token with its corresponding number. The first candidate is the fourth parameter, `Vec` passed by value. Since we defined `Vec` as equal to `1`, the string `UVec` can be replaced with `U1`. Similarly, `const Color&` equal to `AEBW4Color` becomes `AEBW42` because `Color` maps to `2`:
```
     0     1           2       3       4    5       6                                   7
Func(bool, int*, int*, float&, Vec&,   Vec, Color,  const Color&, const Color&, float&, const Vec&)
    _N     PEAH        AEAM    AEAUVec U1   W4Color AEBW42
[0]                                [1]        [2]
```
The remaining parameter before the last pass is `const Vec&`, with `AEBUVec` replaced by `AEBU1`:
```
     0     1           2       3       4    5       6                                   7
Func(bool, int*, int*, float&, Vec&,   Vec, Color,  const Color&, const Color&, float&, const Vec&)
    _N     PEAH        AEAM    AEAUVec U1   W4Color AEBW42                              AEBU1
[0]                                [1]        [2]
```
We are almost done; only the repeating parameters are missing. The second `int*` is equal to the first `int*`, which (from the first row of numbers) is equal to `1`. The second `const Color&` is equal to parameter `6`, and the second `float&` at the end is equal to `2`:
```
     0     1           2       3       4    5       6                                   7
Func(bool, int*, int*, float&, Vec&,   Vec, Color,  const Color&, const Color&, float&, const Vec&)
    _N     PEAH  1     AEAM    AEAUVec U1   W4Color AEBW42        6             2       AEBU1
[0]                                [1]        [2]
```
The mangled name for the function is:
```
64-bit: ?Func@@YAH_NPEAH1AEAMAEAUVec@@U1@W4Color@@AEBW42@62AEBU1@@Z

NAME      = Func
NAMESPACE = -
CALL_CONV = A
RET_Q     = -
RET_TYPE  = H
PARAMS    = _N PEAH 1 AEAM AEAUVec@@ U1@ W4Color@@ AEBW42@ 6 2 AEBU1@
TERMIN    = @Z

32-bit: ?Func@@YAH_NPAH1AAMAAUVec@@U1@W4Color@@ABW42@62ABU1@@Z
```
Let us look at another example.
```C++
namespace Math {
  struct Vec{...};
}

Math::Vec Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
```
First step: associate numbers with named and unnamed parameters that require more than 1 character for encoding (char, int, and float are not in this list):
```
               0                                    1                 2
Math::Vec Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
```
Parameters encoded with a single character can already be translated:
```
               0                                    1                 2
Math::Vec Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
                           D     M      M      H  
```
We can also proceed to encode the first occurrence of repeating and non-repeating parameters:
```
               0                                    1                 2
Math::Vec Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
               PEAH         D     M      M      H   AEBUVec@Math@@    UVec@Math@@
```
The next step is to map indices to the names we have so far. Note that you can already translate the first piece of the symbol at the very beginning, since it always starts with the function name and return type. In this case:
```
?Func@@YA?AUVec@Math@@
 [0]        [1] [2]
```
This means that the tokens `Math` and `Vec` are already present in the ASCII string — we are not encountering them for the first time among the parameters, and they can already be numbered. As a result, `AEBUVec@Math@@` changes to `AEBU12@` and `UVec@Math@@` changes to `U12@`:
```
               0                                    1                 2
Math::Vec Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
               PEAH         D     M      M      H   AEBU12@           U12@
```
Only repeating parameters are missing. In our case, only the second `int*`, which is equal to parameter 0:
```
     0                                    1                 2
Func(int*, int*, char, float, float, int, const Math::Vec&, Math::Vec);
     PEAH  0     D     M      M      H    AEBU12@           U12@
```
The ASCII string for the symbol is:
```
?Func@@YA?AUVec@Math@@PEAH0DMMHAEBU12@U12@@Z

NAME      = Func
NAMESPACE = -
CALL_CONV = A
RET_Q     = ?A (default)
RET_TYPE  = UVec@Math@@
PARAMS    = PEAH 0 D M M H AEBU12@ U12@
TERMIN    = @Z
```
The next function accepts arrays as parameters:
```C++
namespace util {
  enum Color{...};

  class String{...};
}

namespace geom {
  struct Offset{...};
}

util::String F(int*, int[], util::Color [][3], const util::Color&, geom::Offset, geom::Offset&)
```
Associating numbers with named and unnamed parameters that require more than 1 character for encoding:
```
               0     1      2                 3                   4             5
util::String F(int*, int[], util::Color[][3], const util::Color&, geom::Offset, geom::Offset&)
```
We can translate unnamed parameters first:
```
               0     1      2                 3                   4             5
util::String F(int*, int[], util::Color[][3], const util::Color&, geom::Offset, geom::Offset&)
               PEAH  QEAH
```
There are no repeating parameters, only repeating names. The first row of numbers is therefore unused. Let us translate named and unnamed parameters and see if we can replace some tokens later:
```
util::String F(int*, int[], util::Color[][3],   const util::Color&, geom::Offset,  geom::Offset&)
               PEAH  QEAH   QEAY02W4Color@util@ AEBW4Color@util@@   UOffset@geom@@ AEAUOffset@geom@@
```
The first piece of the ASCII symbol is:
```
?F@@YA?AVString@util@@
 0       1      2
```
It appears we can already use `2` for `util`. Moving on, the first `Color` token appears in the third parameter. Since `Color` has not been seen yet, we have no choice but to write its name for the third parameter — but from that point on, we can assign the number 3 to it:
```
?F@@YA?AVString@util@@...Color...
 0       1      2        3
```
The same applies to `Offset` and `geom`, first seen at the fifth parameter. The first time they appear, no numbers are available, so we write plain names for the fifth parameter. After that point, we can assign numbers as well:
```
?F@@YA?AVString@util@@...Color...Offset@geom@
 0       1      2        3       4      5
```
Now we have enough information to shorten some tokens:
```
util   = 2 (since first parameter)
Color  = 3 (after the third parameter)
Offset = 4 (after the fifth parameter)
geom   = 5 (after the fifth parameter)
```
```
util::String F(int*, int[], util::Color[][3], const util::Color&, geom::Offset,  geom::Offset&)
               PEAH  QEAH   QEAY02W4Color@2@  AEBW432@            UOffset@geom@@ AEAU45@
```
The ASCII string for the symbol is:
```
?F@@YA?AVString@util@@PEAHQEAHQEAY02W4Color@2@AEBW432@UOffset@geom@@AEAU45@@Z

NAME      = F
NAMESPACE = -
CALL_CONV = A
RET_Q     = ?A (default)
RET_TYPE  = AVString@util@@
PARAMS    = PEAH QEAH QEAY02W4Color@2@ AEBW432@ UOffset@geom@@ AEAU45@
TERMIN    = @Z
```
One more example before moving on to member functions.
```C++
namespace N1 {
  struct Color {...};
}

namespace N2 {
  union U32 {...};

  struct H {...};

  class String{...}
}

const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2], N2::U32, N2::H*, N2::H*, N2::String)
```
The first step is always to associate an identifier with named and unnamed parameters that require more than 1 character for encoding:
```
                0              1                                   2                3        4               5  
const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2], N2::U32, N2::H*, N2::H*, N2::String)
```
Unnamed parameters can already be translated. In addition, only the second and sixth arguments repeat, so we can set the second `const N1::Color&` = 1 and the second `N2::H*` = 4:
```
                0              1                                   2                3        4               5  
const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2], N2::U32, N2::H*, N2::H*, N2::String)
                QEAY19BE@H                       1                                                   4
```
We can discard the first row of numbers and focus on named parameters. Translate the first occurrence of each named parameter:
```
const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2],  N2::U32,  N2::H*,    N2::H*, N2::String)
                QEAY19BE@H     AEBUColor@N1@@    1                 QEAY01UColor@N1@@ TU32@N2@@ PEAUH@N2@@ 4       VString@N2@@
```
The initial ASCII string for the symbol is:
```
?Get@@YA?BUH@N2@@
 0         1 2
```
Note that the structure `H` is a named parameter but contains only 1 character. This does not matter, however, because it is still a named parameter and also conflicts with `int` = H — so it receives a number too.\
We proceed step by step to avoid confusion. For now, we can replace H → 1 and N2 → 2:
```
const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2],  N2::U32, N2::H*,  N2::H*, N2::String)
                QEAY19BE@H     AEBUColor@N1@@    1                 QEAY01UColor@N1@@ TU32@2@  PEAU12@  4       VString@2@
```
We cannot shorten the second argument `AEBUColor@N1@@` because it first appears at exactly that position. However, we can add that piece to the current ASCII string to obtain additional numbers for use with similar tokens:
```
?Get@@YA?BUH@N2@@...AEBUColor@N1
 0         1  2         3     4
```
Now we also have Color → 3 and N1 → 4 to use after the second parameter:
```
const N2::H Get(int[][10][20], const N1::Color&, const N1::Color&, N1::Color[2][2], N2::U32, N2::H*,  N2::H*, N2::String)
                QEAY19BE@H     AEBUColor@N1@@    1                 QEAY01U34@       TU32@2@  PEAU12@  4       VString@2@
```
The final ASCII string for the symbol `Get` is:
```
?Get@@YA?BUH@N2@@QEAY19BE@HAEBUColor@N1@@1QEAY01U34@TU32@2@PEAU12@4VString@2@@Z

NAME      = Get
NAMESPACE = -
CALL_CONV = A
RET_Q     = ?B (const)
RET_TYPE  = UH@N2@@
PARAMS    = QEAY19BE@H AEBUColor@N1@@ 1 QEAY01U34@ TU32@2@ PEAU12@ 4 VString@2@
TERMIN    = @Z
```
###### MEMBER FUNCTIONS
Member functions are slightly different. Since they can have different access specifiers, an additional table is required:
```
[ACCESS]
         private  protected  public
default     A        I        Q
static      C        K        S
virtual     E        M        U
```
I also repeat here some of the tables encountered so far that we will use again. Calling convention, return object qualifier, and object qualifier (the object that invokes the function):
```
[CALL_CONV]

__cdecl       A
__stdcall     G
__thiscall    E (A for 64-bit build)
__fastcall    I (A for 64-bit build)
__vectorcall  Q
```
```
[RET_Q]

default         ?A
const           ?B
volatile        ?C
const volatile  ?D
```
```
[OBJ_Q]

default        A
const          B
volatile       C
const volatile D
```
The template for a member function:
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [64-BIT: E] [OBJ_Q] [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
Examples:
```

namespace Util {
  struct Vector{...}
  enum class Color{...}

class SomeClass
{
public:
  int GetValue(){...} //?GetValue@SomeClass@Util@@QEAAHXZ

NAME      = GetValue@SomeClass@
NAMESPACE = Util@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = H
PARAMS    = X

protected:
  float DoSomething(int*, int*, float, const Vector&, const Vector&) 

?DoSomething@SomeClass@Util@@IEAAMPEAN0MAEBUVector@2@1@Z

NAME      = DoSomething@SomeClass@
NAMESPACE = Util@
ACCESS    = I
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = M
                             0                  1 
util::SomeClass::DoSomething(int*, int*, float, const Vector&, const Vector&) 
                                   0                           1
DoSomething@SomeClass@util
0           1         2

PARAMS = PEAH 0 M AEBUVector@2@ 1

private:                0                       1               2
  Color Calculate(char, enum Color, enum Color, float&, float&, bool, bool)
                                    0                   1             2
                                  
  ?Calculate@SomeClass@Util@@AEAA?AW4Color@2@DW432@0AEAM1_N2@Z 
   0         1         2              3

NAME      = Calculate@SomeClass@
NAMESPACE = Util@
ACCESS    = A
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = ?A
RET_TYPE  = W4Color@2@
PARAMS    = D W432@ 0 AEAM 1 _N 2
```
```
namespace global {
  enum  Type {...}
  union M16  {...}
}

namespace tt {
  struct Color{...}
}

class Path
{
private:
  const global::Type Func(int, const tt::Color [][4], global::M16, bool) const

?Func@Path@@AEBA?BW4Type@global@@HQEAY03$$CBUColor@tt@@TM16@3@_N@Z

?Func@Path@@...W4Type@global@@
 0    1          2    3  

NAME      = Func@Path@@
NAMESPACE = -
ACCESS    = A
64-BIT    = E
OBJ_Q     = B
CALL_CONV = A
RET_Q     = ?B
RET_TYPE  = W4Type@global@@
PARAMS    = H QEAY03$$CBUColor@tt@@ TM16@3@ _N 

virtual tt::Color GetColor(global::M16, const tt::Color&, char, char, int*)

?GetColor@Path@@MEAA?AUColor@tt@@TM16@global@@AEBU23@DDPEAH@Z   

?GetColor@Path@@...UColor@tt@
 0        1         2     3        

NAME      = GetColor@Path@@
NAMESPACE = -
ACCESS    = M
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = ?A
RET_TYPE  = UColor@tt@@
PARAMS    = TM16@global@@ AEBU23@ D D PEAH
```
###### SPECIAL FUNCTIONS
Special functions are not special in any profound sense — I use the term to refer to functions that are neither global nor common member functions. In this category are constructors and operators. These functions have a special prefix; yet another table is required. Let us start with constructors and operators:
```
constructor     ?0
destructor      ?1
operator new    ?2  
operator delete ?3
operator =      ?4
operator >>     ?5
operator <<     ?6
operator !      ?7
operator ==     ?8
operator !=     ?9
operator []     ?A
operator TYPE   ?B
operator ->     ?C
operator *      ?D
operator ++     ?E
operator --     ?F
operator -      ?G
operator +      ?H
operator &      ?I
operator ->*    ?J
operator /      ?K
operator %      ?L
operator <      ?M
operator <=     ?N
operator >      ?O
operator >=     ?P
operator ,      ?Q
operator ()     ?R
operator ~      ?S
operator ^      ?T
operator |      ?U
operator &&     ?V
operator ||     ?W
operator *=     ?X
operator +=     ?Y
operator /=     ?_0
operator %=     ?_1
operator >>=    ?_2
operator <<=    ?_3
operator &=     ?_4
operator |=     ?_5
operator ^=     ?_6
```
###### OPERATOR FUNCTIONS
Global and member function templates:
```
? [NAME] @ [NAMESPACE] @ Y [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [64-BIT: E] [OBJ_Q] [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
```
struct Vector{...};

bool operator == (const Vector& a, const Vector& b)
            
??8@YA_NAEBUVector@@0@Z
            0 

NAME      = ?8 (operator ==)
NAMESPACE = -
CALL_CONV = A  (default)
RET_Q     = -
RET_TYPE  = _N (bool)
PARAMS    = AEBUVector@@ 0
```
```
struct Vector {
  float& operator[](int i){...}
};

??AVector@@QEAAAEAMH@Z

NAME      = ?A (operator [])
NAMESPACE = Vector@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = AEAM
PARAMS    = H
```
```

namespace Util {
class String
{
public:
  String::String() ??0String@Util@@QEAA@XZ

NAME      = ?0 (constructor)
NAMESPACE = String@Util@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = X (void)


  ~String() ??1String@Util@@QEAA@XZ

NAME      = ?1 (destructor)
NAMESPACE = String@Util@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = X (void)


  String(const char*) ??0String@Util@@QEAA@PEBD@Z

NAME      = ?0 (constructor)
NAMESPACE = String@Util@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = PEBD


  String::String(const String&) ??0String@Util@@QEAA@AEBV01@@Z
                                   0      1
NAME      = ?0 (constructor)
NAMESPACE = String@Util@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = AEBV01@


  String& operator + (const String&) ??HString@Util@@QEAAAEAV01@AEBV01@@Z
                                        0      1
NAME      = ?H (operator +)
NAMESPACE = String@Util@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = AEAV01@
PARAMS    = AEBV01@


protected:
  operator int() ??BString@Util@@IEAAHXZ 

NAME      = ?B (operator TYPE)
NAMESPACE = String@Util@@
ACCESS    = I
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = H
PARAMS    = X
```
```
class Shape
{
public:
  Shape() ??0Shape@@QEAA@XZ
  
NAME      = ?0 (constructor)
NAMESPACE = Shape@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = X

  Shape(Shape&&) ??0Shape@@QEAA@$$QEAV0@@Z 
                    0
NAME      = ?0 (constructor)
NAMESPACE = Shape@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = $$QEAV0@@  


  virtual ~Shape() ??1Shape@@UEAA@XZ 

NAME      = ?1 (destructor)
NAMESPACE = Shape@@
ACCESS    = U
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = -
RET_TYPE  = -
PARAMS    = X (void)


  const float operator^(const Shape& a) const ??TShape@@QEBA?BMAEBV0@@Z
                                                 0
NAME      = ?T (operator ^)
NAMESPACE = Shape@@
ACCESS    = Q
64-BIT    = E
OBJ_Q     = B (const)
CALL_CONV = A
RET_Q     = ?B (const)
RET_TYPE  = M
PARAMS    = AEBV0@@
```
###### TEMPLATES
Template encoding is very similar to that of functions. In the end, once the compiler instantiates a template, it becomes a normal callable function. Let us start with global functions. The only difference is the addition of template parameters between the name and the namespace:
```
? ?$ [NAME] @ [T_PARAMS] @ [NAMESPACE] @ Y [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
```
namespace test {
  template<typename T>
  void Func(T x)
}

Func<int>(int) ??$Func@H@test@@YAXH@Z

NAME      = Func
T_PARAMS  = H
NAMESPACE = test@
CALL_CONV = A (default)
RET_Q     = -
RET_TYPE  = X (void)
PARAMS    = H

Func<char>(char) ??$Func@D@test@@YAXD@Z

NAME      = Func
T_PARAMS  = D
NAMESPACE = test@
CALL_CONV = A (default)
RET_Q     = -
RET_TYPE  = X (void)
PARAMS    = D
```
```
template<typename T1, typename T2>
int F1(const T1&, const T1&, T2*, int, int);

               0                         1
F1<char,float>(const char&, const char&, float*, int, int) ??$F1@DM@@YAH AEBD0PEAMHH@Z 

NAME      = F1
T_PARAMS  = DM
NAMESPACE = -
CALL_CONV = A (default)
RET_Q     = -
RET_TYPE  = H
PARAMS    = AEBD 0 PEAM H H
```
Note that a function whose arguments or return type are template objects is not itself a template function. The format for the ASCII symbol falls back to that of normal member functions in this case, but template objects are still constructed using the same rules.
```
? [NAME] @ [NAMESPACE@] @ [ACCESS] [64-BIT: E] [OBJ_Q] [CALL_CONV] [RET_Q] [RET_TYPE] [X or PARAMS...@] Z
```
```
namespace stuff {
  template<typename T>
  struct String{...}
}

template<typename T>
class Something
{
public:  
  stuff::String<T> Do(const stuff::String<T>&, int, const char[][32][32])

?Do@?$Something@D@@QEAA?AU?$String@D@stuff@@AEBU23@HQEAY1CA@CA@$$CBD@Z
 0    1                     2        3 

NAME      = Do@?$Something@D@@
NAMESPACE = ?$Something@D@@ (template Something<char>)
ACCESS    = Q
64-BIT    = E
OBJ_Q     = A
CALL_CONV = A
RET_Q     = ?A (default; not omitted for classes, structures, or unions)
RET_TYPE  = U?$String@D@stuff@@ (struct template stuff::String<char>)
PARAMS    = AEBU23@ H QEAY1CA@CA@$$CBD  
```
