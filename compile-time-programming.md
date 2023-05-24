# Programowanie na etapie kompilacji

## consteval

* Określa, że funkcja jest tzw. *immediate function*
* Każde wywołanie funkcji zwraca stałą czasu kompilacji
* **Gwarantuje ewaluację na etapie kompilacji !!!**

``` c++
int runtime_func(int x)
{
    return x * x;
}

constexpr int constexpr_func(int x) 
{
    return x * x;
}

consteval int consteval_func(int x)
{
    return x * x;
}
```

``` c++
int x = 665;

std::cout << runtime_func(x); // Ok
std::cout << constexpr_func(x); // Ok
std::cout << consteval_func(x); // Error!!!

std::cout << runtime_func(42); // Ok
std::cout << constexpr_func(42); // Ok
std::cout << consteval_func(42); // Ok - called at compile time
```

### Ograniczenia funkcji consteval

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
@[1-2,6-9,16-17]

### std::is_constant_evaluated()

* Zwraca ``true`` jeśli
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
  * nie można użyć utworzonego na etapie kompilacji kontenera w runtimie (na razie)

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