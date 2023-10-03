# Lambdy w C++20

## Lambdy z parametrami szablonu

* Od C++20 można tworzyć generyczne lambdy wykorzystując w nich listę parametrów szablonu:

``` c++
auto closure = []<typename T>(const T& arg) {
    T temp{};
    //...
};
```

### Ograniczanie typów parametrów

* Daje nam to możliwość ograniczania zakresu typów parametrów dla lambdy generycznej:

``` c++
auto shrink_it = []<typename T>(std::vector<T>& vec) { // only std::vector
    vec.shrink_to_fit();
};
```

### Lepsza dedukcja parametrów

* Można przeprowadzić lepszą dedukcję parametrów lambdy:

``` c++
auto some_lambda = []<typename T, size_t N>(T (&arr)[N]) {
    T target[N];

    //...
};
```

### Wygodniejszy perfect forwarding

* Można wygodniej zastosować perfect forwarding w lambdzie:

``` c++
auto call = []<typename F, typename... Args>(F&& f, Args&&... args) {
    f(std::forward<Args>(args)...);
};
```

zamiast:

``` c++
auto call = [](auto&& f, auto&&... args) {
    f(std::forward<decltype(args)>(args)...);
};
```

## Domyślne konstruktory dla lambd

* Jeśli wyrażenie lambda niczego nie przechwytuje, to tworzona klasa domknięcia posiada konstruktor domyślny:

``` c++
auto cmp_desc = [](const auto& a, const auto& b) { return a > b; };

decltype(cmp_desc) my_comparer; // now OK
```

* Jest to przydatne np. w kontenerach asocjacyjnych:

``` c++
std::set<int, decltype(cmp_desc)> set_numbers = {1, 6, 3, 2, 7, 9};
```

## Przechwytywanie structured bindings

* Dozwolone jest przechwytywanie *structured bindings*:

``` c++
std::map<int, std::string> dict;


for (const auto& [key,value] : dict) {
    auto l = [key, value] { // OK since C++20
        //...
    };
}
```

### Przechwytywanie parametrów paczki szablonu wariadycznego

``` c++
auto create_caller(auto f, auto... args)
{
   return [f, ...args = std::move(args)]() -> decltype(auto) {
        return f(args...);
   }; 
}

auto calculate = create_caller(std::plus{}, 3, 5);

assert(calculate() == 8);
```