# Core C++ - nowości

## Range-Based For z sekcją inicjującą

* Przed C++20 użycie obiektów tymczasowych w pętli *range-based for* mogło prowadzić do UB

``` c++
std::optional<std::vector<int>> load_data();

for(int i : load_data().value()) // UB!!!
{
    //...
}
```

* Od C++20 możemy użyć sekcji inicjującej

``` c++
for(auto&& data = load_data(); int i : data.value())
{
    //...
}
```

## Deklaracja using + scoped enumeration

* Deklaracja `using` może zostać użyta dla typów wyliczeniowych (*scoped enums*)

``` c++
enum class DownloadStatus { not_started, pending, done };
```

``` c++
using enum DownloadStatus;

DownloadStatus s = not_started;

switch (s)
{
case not_started:
    std::cout << "not started!\n";
    break;
case pending:
    std::cout << "in progress\n";
    break;
case done:
    std::cout << "done\n";
    break;
}
```

## Atrybuty w C++20

### nodiscard z uzasadnieniem

* Atrybut `nodiscard` może przyjąć parametr, podający uzasadnienie ostrzeżenia

``` c++
[[nodiscard("Always check the status")]] DownloadStatus download_file(
    const std::string& url)
{
    using DownloadStatus::done;

    //...

    return done;
}
```

### likely vs. unlikely

* W określonych sytuacjach możemy podpowiedzieć kompilatorowi, który branch w instrukcji warunkowej powinien preferować

``` c++
int foo(int n)
{
    if (n <= 0) [[likely]]
    {
        return n;
    }
    else
    {
        return n * n;
    }
}

int bar(int n)
{
    if (n <= 0) [[unlikely]]
    {
        return n;
    }
    else
    {
        return n * n;
    }
}
```

### no_unique_address

* Ulepszenie tzw. *Empty base class optimization*

``` c++
struct Empty  // sizof(Empty) == 1
{};

struct Value // sizeof(Valueue) == 4
{
    int i;
};

struct EmptyAndValue // sizeof(EmptyAndValue) == 8
{
    Empty e;
    int i;
};

struct EmptyWithAttrAndValue // sizeof(EmptyWithAttrAndValue) == 4
{
    [[no_unique_address]] Empty e;
    int i;
};

struct ValueAndEmptyWithAttr // sizeof(ValueueAndEmptyWithAttr) == 4
{
    int i;
    [[no_unique_address]] Empty e;
};
```

## Typ char8_t dla UTF-8

* Do obsługi UTF-8 dostępne są:
  * nowy typ znakowy - `char8_t` - litearał: `u8'a'`
  * nowy typ tekstowy - `std::u8string` - literał: `u8"text"s`
  * nowy widok tekstu - `std::u8string_view` - literał `u8"text"sv`

```c++
char8_t c = u8'$';
std::u8string str1 = u8"Krak\u00F3w";

const char8_t* str_utf8 = u8"Krak\u00F3w"; // "Kraków" w UTF-8

std::cout << str_utf8 << "\n"; // ERROR - << jest usunięty dla UTF-8
std::cout << reinterpret_cast<const char*>(str_utf8) << "\n"; // OK
```

``` c++
using namespace std::literals;
auto str2 = u8"Krak\u00F3w"s; // std::u8string
auto sv = u8"Gda\u0144sk"sv;   // std::u8string_view

std::cout << reinterpret_cast<const char*>(sv.data()) << "\n";
```

```c++
const char8_t cEuro = u8'€'; // ERROR - multibyte character
const char8_t* sEuro = u8"\u20AC"; // OK
```

* Mając nowy typ możemy przeciążać funkcje

``` c++
void save(const char* s) 
{
    store_in_file(convert_to_utf8(s));
};

void save(const char8_t* s) 
{
    store_in_file(s);
};  

//...

save("text");
save(u8"text");
```

* Nowy typ może spowodować błędy kompilacji w starszym kodzie:

``` c++
// iterate over directory entries:
for (const auto& entry : fs::directory_iterator(path)) 
{
    std::string name = entry.path().u8string(); // OK in C++17, ERROR since C++20
    //...
}
```

* Rozwiązanie problemy z wykorzystaniem *feature test macro*

``` c++
// iterate over directory entries (C++17 and C++20):
for (const auto& entry : fs::directory_iterator(path)) 
{
#ifdef __cpp_char8_t
    std::u8string name = entry.path().u8string(); // OK since C++20
#else
    std::string name = entry.path().u8string(); // OK in C++17
#endif
    //...
}
```

## Agregaty w C++20

* Designated initializers
* Inicjalizacja z użyciem `()`
* Domyślne reguły CTAD
* Agregaty jako NTTP

### Agregaty - designated initializers

``` c++
struct Person
{
    int id;
    std::string name{"unknown"};
    double salary;
};

Person p1{42, "John", 6650.2}; // classic initialization

Person p2{.id = 123, .salary = 5666.1}; // new syntax with designated initializers

Person p3{.name = "Adam", .salary = 10'000.1}; 

Person p4{.salary = 100.1, .name = "Adam"}; // ERROR - wrong order

Person p5{665, .salary = 12'000.0 }; // ERROR - mixed style of initialization
```

### Agregaty - inicjalizacja ()

``` c++
struct Person
{
    int id;
    std::string name{"unknown"};
    double salary;
};

Person p1{1, "John", 667.5};
Person p2(1, "John", 667.5); //OK - since C++20

// since C++20 std::make_unique<T> works with aggregates
auto ptr_p = std::make_unique<Person>(1, "John", 667.5);
assert(ptr_p->id == 1);

Person p3 = (2, "Adam"); // ERROR 
Person p3(.salary=10'000.0); // ERROR - no designated initializers with ()
```

#### Agregaty - inicjalizacja () i tablice

``` c++
int arr1[]{1, 2, 3, 4}; // OK since C++11
int arr2[](1, 2, 3, 4); // OK since C++20
int arr3[] = {1, 2, 3, 4};  OK
int arr4[] = (1, 2, 3, 4); // ERROR

std::array<int, 4> a1{1, 2, 3, 4}; // OK
std::array<int, 4> a2(1, 2, 3, 4); // ERROR

```

### CTAD dla agregatów

W C++17 aby zastosować dla agregatów Class Template Argument Deduction (CTAD) należało napisać podpowiedź dedukcyjną:

``` c++
template <typename T1, typename T2>
struct Pair
{
    T1 fst;
    T2 snd;
};

// deduction guide required for CTAD in C++17
template <typename T1, typename T2>
Pair(T1, T2) -> Pair<T1, T2>; 
```

``` c++
Pair p1{1, "text"}; // Pair<int, const char*>
```

Od C++20 nie jest to wymagane. Odpowiednie (domyślne) podpowiedzi dedukcyjne wygeneruje kompilator:

``` c++
template <typename T1, typename T2>
struct Pair
{
    T1 fst;
    T2 snd;
};

// no need for custom deduction guide
// compiler provide
// appropriate version
```

``` c++
Pair p1{1, "text"}; // Pair<int, const char*>
```

## Feature Test Macros

* Nagłówek `<version>`
* Ułatwiają pisanie kodu, który ma być kompilowany w wielu standardach:
  * [lista feature'ów](https://en.cppreference.com/w/cpp/feature_test):


``` c++
#ifndef __cpp_lib_as_const
namespace FutureStd
{
    template<typename T>
    const T& as_const(T& t) 
    {
        return t;
    }
}
#endif

#ifdef __cpp_lib_as_const
  auto print = [&coll = std::as_const(coll)] {
#else
  auto print = [&coll = FutureStd::as_const(coll)] {
#endif
  //... rest of implementation
};
```

## std::source_location

* Zastępstwo dla makr typu `__FILE__`, `__LINE__` i `__func__`

``` c++
#include <source_location>

template <typename T>
void foo_location(T value)
{
    auto sl = std::source_location::current();

    std::cout << "file: " << sl.file_name() << "\n";
    std::cout << "function: " << sl.function_name() << "\n";
    std::cout << "line/col: " << sl.line() << "\n";
}
```
@[1,6,8-10]

```
file: /workspaces/cpp-20/code/core-features/core_features_cpp20.cpp
function: void foo_location(T) [with T = std::__cxx11::basic_string<char>]
line/col: 301
```

## Operacje bitowe

* Nowe API dla operacji bitowych
* Nagłówek `<bit>`

|       Operation       |                       Meaning                        |
| :-------------------: | :--------------------------------------------------: |
|    `rotl(value, n)`     |      Yields val with n bits rotated to the left      |
|    `rotr(value, n)`     |     Yields val with n bits rotated to the right      |
|  `countl_zero(value)`   |  Yields number of leading (most significant) 0 bits  |
|   `countl_one(value)`   |  Yields number of leading (most significant) 1 bits  |
|  `countr_zero(value)`   | Yields number of trailing (least significant) 0 bits |
|   `countr_one(value)`   | Yields number of trailing (least significant) 1 bits |
|    `popcount(value)`    |         Yields number of 1 bits in the value         |
| `has_single_bit(value)` |   Yields whether val is a power of 2 (one bit set)   |
|   `bit_floor(value)`    |          Yields previous power-of-two value          |
|    `bit_ceil(value)`    |            Yields next power-of-two value            |
|   `bit_width(value)`    |  Yields number of bits necessary to store the value  |

## std::endian

* Pozwala na sprawdzenie rodzaju kolejności bajtów (*endianness*) w trakcie kompilacji

``` c++
#include <bit>

if constexpr(std::endian::native == std::endian::big) 
{
    // handle big-endian
}
else if constexpr(std::endian::native == std::endian::little) {
    // handle little-endian
}
else 
{
    // handle mixed endian
}
```