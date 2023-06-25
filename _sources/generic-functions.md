# Funkcje szablonowe z auto

Od C++14 każda funkcja mogła deklarować typ zwracany jako `auto` lub `decltype(auto)`.

```c++
auto foo() // deduced as: int foo()
{
    return 42;
}
```

Z kolei lambdy generyczne (również od C++14) mogły wykorzystać placeholder `auto` również jako deklarację typu parametru:

```c++
auto printer_lambda = [](const auto& collection) {
    for(const auto& item : collection)
        std::cout << item << " ";
    std::cout << "\n";
};

std::vector words<std::string> = {"This", "is", "test", "of", "generic", "lambda"};

printer_lambda(words);
```

Od C++20 możemy stosować placeholder `auto` również dla nazwanych funkcji.

```c++
void printer(const auto& collection) {
    for(const auto& item : collection)
        std::cout << item << " ";
    std::cout << "\n";
};
```

Kompilator rozwija taką implementację do następującej postaci:

```c++
template <typename T>
void printer(const T& collection) {
    for(const auto& item : collection)
        std::cout << item << " ";
    std::cout << "\n";
};

std::vector<std::string> words = {"This", "is", "test", "of", "generic", "lambda"};

printer(words); // call with template argument deduction
printer<std::vector<std::string>>(words); // call with explicit template argument
```

W przypadku gdy argumentów z `auto` jest więcej, dla każdego `auto` wprowadzany jest nowy parametr typu:

```c++
bool cmp_by_size(const auto& a, const auto& b)
{
    return a.size() < b.size();
}
```

Rozwinięta przez kompilator implementacja ma postać:

```c++
template <typename T1, typename T2>
bool cmp_by_size(const T1& a, const T2& b)
{
    return a.size() < b.size();
}
```

```{warning}
Generyczne wyrażenie lambda nie jest równoważne funkcji szablonowej z `auto`.

Generyczne wyrażenie lambda powoduje powstanie obiektu domknięcia z szablonowym operatorem wywołania funkcji. Różnica jest zauważalna przy przekazywaniu parametrów do algorytmów standardowych:

```c++
std::vector<std::string> words = {"This", "is", "test", "of", "generic", "lambda"};

// sorting: generic function with auto
std::ranges::sort(words, cmp_by_size); // ERROR
std::ranges::sort(words, cmp_by_size<std::string, std::string>); // OK

auto cmp_by_size_lambda = [](const auto& a, const auto& b) { return a.size() < b.size(); };
std::ranges::sort(words, cmp_by_size_lambda); //OK
std::ranges::sort(words, cmp_by_size_lambda<std::string, std::string>); // ERROR
```