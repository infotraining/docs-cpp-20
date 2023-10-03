# Programowanie na etapie kompilacji

C++20 znacznie rozszerza możliwości programowania na etapie kompilacji. Wykorzystywane są do tego zmienne lub funkcje z dodanymi modyfikatorami:

* `constexpr` - wprowadzony w C++11
* `consteval`
* `constinit`

## consteval

Jest nowym słowem kluczowym w C++20. Określa, że funkcja jest tzw. *immediate function*. Każde wywołanie funkcji zwraca wartość wyliczoną na etapie kompilacji.

```{attention}
**`consteval` gwarantuje ewaluację na etapie kompilacji !!!**
```

W praktyce od C++20 możemy mówić o trzech kategoriach funkcji:

* funkcjach runtime - wywołanie odbywa się w czasie wykonania programu

``` c++
int runtime_func(int x)
{
    std::cout << runtime_func(x); // Ok
    return x * x;
}
```

* funkcjach `constexpr` - wywołanie może odbyć się w runtime lub w czasie kompilacji 

```c++
int x = 665;
const int y = 665;

constexpr int constexpr_func(int x) 
{
    return x * x;
}

std::cout << constexpr_func(x); // Ok - called at runtime
constexpr int value_1 = constexpr_func(x); // ERROR!!! - runtime argument
constexpr int value_2 = constexpr_func(42); // OK - called at compile time constexpr int value_3 = constexpr_func(y);  // OK - called at compile time
```

* funkcjach `consteval` - wywołanie może odbyć się **tylko** na etapie kompilacji

```c++
int x = 665;

consteval int consteval_func(int x)
{
    return x * x;
}

std::cout << consteval_func(42); // Ok - called at compile time
std::cout << consteval_func(x); // ERROR!!! - runtime argument
```

### Ograniczenia funkcji consteval

Jeśli w trakcie wykonywania funkcji `consteval` nastąpi próba wykonania funkcji runtime'owej zgłaszany jest błąd kompilacji:

``` c++
void compile_time_error()
{}

consteval int next_two_digit_value(int value)
{
    if (value < 9 || value >= 99)
    {
        compile_time_error();
    }

    return ++value;
}

constexpr int n1 = next_two_digit_value(9); // Ok -> n1 == 10
constexpr int n2 = next_two_digit_value(42); // Ok -> n2 == 43
constexpr int n3 = next_two_digit_value(99); // Compile-time ERROR
constexpr int n4 = next_two_digit_value(5); // Compile-time ERROR
```

### std::is_constant_evaluated()

Pomocną funkcją, która pozwala określić w jakim kontekście ewaluowana jest funkcja (czy w runtime'ie czy na etapie kompilacji), jest funkcja `std::is_constant_evaluated()`

Funkcja zwraca ``true`` jeśli:

* jest stałym wyrażeniem
* jest użyte w kontekście `constexpr`, `consteval` lub inicjalizacji stałej
* jest użyte w trakcie inicjalizacji zmiennej na etapie kompilacji

``` c++
constexpr int len(const char* s)
{
    if (std::is_constant_evaluated())
    { 
        // compile-time friendly code
        int idx = 0;
        while (s[idx] != '\0')
            ++idx;
        return idx;        
    }
    else
    {
        return std::strlen(s); // function called at runtime
    }
}

int l1 = len("hello"); // run-time branch in action
constexpr int l2 = len("hello"); // compile-time branch in action
```

## Ograniczenia dla constexpr & consteval w C++20

* funkcje `constexpr` (i `consteval`) mogą:
  * być funkcjami wirtualnymi
  * używać `dynamic_cast<>`
  * używać `typeid()`
  * mieć bloki `try/catch` (ale nie `throw`)
  * alokować pamięć przy pomocy `std::allocator`
  * zmieniać stan unii - `union`

* `std::vector<>` & `std::string` są `constexpr`
  * można używać tych kontenerów w trakcie kompilacji
  * nie można użyć utworzonego na etapie kompilacji kontenera w runtime'ie (na razie)

### Ograniczenia dla wektorów i stringów

* Obiekty mogą żyć tylko na etapie kompilacji
  * są tworzone i niszczone na etapie kompilacji
  * muszą mieć destruktor ``constexpr``
* Do zarządzania pamięcią mogą być użyte tylko alokatory używające:
  * `std::allocator<T>::allocate()` do przydziału pamięci
  * `std::allocator<T>::deallocate()` do zwolnienia pamięci

``` c++
constexpr std::string s1{"short"}; // may compile with SSO
constexpr std::string s2{"a long string without SSO"}; // ERROR
constexpr auto length = std::string{"hello"}.size(); // OK
```

Przykład użycia `std::string` w funkcjach `constexpr`:

```c++
constexpr std::string get_str()
{
    std::string s1 {"hello"};
    std::string s2 {"world"};
    std::string s3 = s1 + " " + s2 + "!";
    return s3;
}

constexpr auto get_array()
{
    constexpr auto N = get_str().size();
    std::array<char, N> arr {};
    std::copy_n(get_str().data(), N, std::begin(arr));
    return arr;
}

static constexpr auto text_bytes = get_array();
constexpr std::string_view text_sv{text_bytes.data(), text_bytes.size()};
static_assert(text_sv == "hello world!"sv);
```

## constinit

Jednym z powtarzających się problemów w dużych projektach C++ jest tzw. "static initialization order fiasco". Problem ten wynika z nieokreślonej kolejności inicjalizacji zmiennych statycznych znajdujących się w różnych jednostkach translacji (plikach `.cpp`).

Modyfikator `constinit` gwarantuje, że dana zmienna jest inicjalizowana w czasie kompilacji.

```c++
constexpr std::array values{1, 2, 3};
    
static constinit std::array data = values; // initialized at compile time

data[1] = 42; // but still variable - not constexpr

assert(data == std::array{1, 42, 3});
```

`constinit` może być połączony z deklaracją `thread_local`. W takim przypadku, ponieważ wartość, którą inicjalizujemy zmienną lokalną wątku jest określona na etapie kompilacji, kompilator może uniknąć generowania kodu, który ma zagwarantować inicjalizację thread-safe.

```c++
// tls_definitions.cpp
thread_local constinit int tls1{1};
thread_local int tls2{2};

// main.cpp
extern thread_local constinit int tls1;
extern thread_local int tls2;

int get_tls1() {
    return tls1;  // pure TLS access
}

int get_tls2() {
    return tls2;  // has implicit TLS initialization code
}
```