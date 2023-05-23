# Biblioteka Ranges

## Ranges w C++20

W skład bibilioteki Ranges wchodzą:
* Algorytmy
* Koncepty
* Narzędzia pomocnicze
  * funkcje, punkty customizacji, typy, cechy typów
* Widoki - *Views*
  * `lightweight ranges`
  * Potoki - `operator |`

## Ranges - koncepty

|        Concept        |                      (in std::ranges) Requires                      |
| --------------------- | ------------------------------------------------------------------- |
| `range`               | Iterable from begin to end                                          |
| `output_range`        | Range to write to element values                                    |
| `input_range`         | Range to read element values from                                   |
| `forward_range`       | Range you can iterate over multiple times to read values            |
| `bidirectional_range` | Range you can iterate over backwards                                |
| `random_access_range` | Range with random access (jump back and forth between elements)     |
| `contiguous_range`    | Range with all elements in contiguous memory                        |
| `sized_range`         | Range with constant-time size()                                     |
| `view`                | Range that is cheap to copy or move and assign                      |
| `viewable_range`      | Range that is convertible to a view (with std::ranges::all())       |
| `borrowed_range`      | Range with iterators that are not tied to the lifetime of the range |
| `common_range`        | Ranges where begin and end (sentinel) have the same type            |

Użycie konceptów z biblioteki Ranges:

``` c++
void print(auto& obj)
{
    std::cout << obj << "\n";
}

void print(std::ranges::input_range auto const& coll)
{
    for(auto const& item : coll)
        std::cout << item << " ";
    std::cout << "\n";
}

print(665);

std::vector vec{1, 2, 3};
print(vec);

print("ranges");
```

``` bash
665
1 2 3
r a n g e s
```

## Ranges - narzędzia

### Typy pomocnicze dla zakresów

* `std::ranges::iterator_t<>`
* `std::ranges::sentinel_t<>`
* `std::ranges::range_difference_t<>`
* `std::ranges::range_size_t<>`
* `std::ranges::range_value_t<>`
* `std::ranges::range_reference_t<>`
* `std::ranges::range_rvalue_reference_t<>`
* `std::ranges::borrowed_iterator_t<>`
* `std::ranges::borrowed_subrange_t<>`
* `std::ranges::views::all_t<>`

### Customization Point Objects

* `std::ranges::begin()`
* `std::ranges::end()`
* `std::ranges::cbegin()`
* `std::ranges::cend()`
* `std::ranges::empty()`
* `std::ranges::size()`
* `std::ranges::ssize()`
* `std::ranges::data()`
* `std::ranges::cdata()`

## Ranges - algorytmy

* Odpowiedniki standardowych algorytmów
  * zakresy zamiast par iteratorów jako argumenty
  * ograniczenia parametrów szablonów (użyte koncepty)
* Brak wsparcia:
  * dla algorytmów numerycznych (np. `accumulate()`)
  * dla współbieżności

``` c++ code-noblend
std::vector vec{5, 1, 0, -4, 42, 665, 8};

std::ranges::sort(vec); // sorting vec (all range)
std::ranges::sort(vec, std::greater{}); // sorting range with comparer
```

### Projekcje

* Algorytmy `Range` akceptują jako jeden z argumentów **projekcję**
  * transformację elementu kolekcji w trakcie wykonania algorytmu

``` c++ code-noblend
std::vector<std::string> words{"c", "c++", "c++20", "template"};

std::ranges::sort(words, 
    [](const auto& a, const auto& b) { return a.size() > b.size()}
);

std::ranges::sort(words, 
    std::greater{}, 
    [](const auto& value) { return value.size();}
);

auto pos = std::ranges::find(words, 5, 
    [](const auto& value) { return value.size();});

assert(*pos == "c++20");
```

### Algorytmy z wartownikiem

**Wartownik** (*sentinel*) - specjalna wartość, która oznacza koniec zakresu (lub przerwanie iteracji)

``` c++
template <auto Value_>
struct EndValue
{
    bool operator==(auto pos) const
    {
        return *pos == Value_;
    }
};
```

Algorytmy z wartownikiem nie wymagają aby para argumentów przekazanych w celu określenia zakresu była tego samego typu.

``` c++
void print(auto first, auto last)
{
    std::cout << "[ ";
    for(auto it = first; it != last; ++it)
    {
        std::cout << *it << " ";
    }
    std::cout << "]\n";
}
```

``` c++
std::vector vec = {1, 5, 2345, 23, 665, -1, 13, 42};

print(vec.begin(), vec.end()); // using iterators - classic approach

print(vec.begin(), EndValue<665>{}); // using iterator + sentinel
```

``` c++
std::ranges::sort(vec.begin(), EndValue<13>{});
```

#### Algorytmy - przed C++20

``` c++
std::transform(vec.begin(), vec.end(), // source - pair of iterators
               vec.begin(), // destination
               [](int x) { return x * x;}); // transformation
```

``` c++
std::transform(std::execution::par, // since C++17 - parallel version
               vec.begin(), vec.end(), // source - pair of iterators
               vec.begin(), // destination
               [](int x) { return x * x;}); // transformation
```

#### Algorytmy - od C++20

``` c++
std::ranges::transform(vec.begin(), EndValue<42>{}, // supporting sentinels
                       vec.begin(), // destination
                       [](int x) { return x * x;}); // transformation

std::ranges::transform(vec.begin(), EndValue<42>{}, // supporting sentinels
                       vec.begin(), // destination
                       [](int x) { return 2 * x; }, // projection
                       [](int x) { return x * x;}); // transformation

std::ranges::transform(vec, // range
                       vec.begin(), // destination
                       [](int x) { return x * x;}); // transformation

std::ranges::transform(vec, // range
                       vec.begin(), // destination
                       [](int x) { return 2 * x; }, // projection
                       [](int x) { return x * x;}); // transformation
```

## Podzakresy

Klasa `std::ranges::subrange` łączy iterator z wartownikiem tworząc podzakres (widok - *view*)

``` c++ code-noblend
void print(std::ranges::input_range auto&& rng)
{
    print(rng.begin(), rng.end());
}

auto partial_data = 
    std::ranges::subrange(vec.begin(), EndValue<665>{});

print(partial_data);

print(std::ranges::subrange(vec.begin(), EndValue<665>{}));
```

## Widoki - Views

* Widok (View)
  * lekki (tani w kopiowaniu/przenoszeniu, przypisywaniu i niszczeniu)
  * zwykle z semantyką referencyjną
  * może być nieskończony (generuje sekwencję elementów)

* Ograniczenia dla widoków (*requirements*)
  * `std::ranges::range<T>`
  * `std::ranges::moveable<T>`
  * operacje kopiowania i przenoszenia oraz destruktor są *O(1)*

### Widoki / Adaptory / Fabryki

* **Adaptor** (*range adaptor*)
  * odpowiedni obiekt funkcyjny, który umożliwia utworzenie widoku z zakresu poprzez wywołanie funkcji
  * dostępny dla (prawie) każdego widoku

* **Fabryka zakresu** (*range factory*)
  * funkcja, która tworzy widok generujący wartości


| Adaptor / Factory  |                                Typ                                |                                                                          Efekt                                                                           |
| ------------------ | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `all(rng)`         | wariant: <br/> + typ `rng`<br/> + `ref_view`<br/> + `owning_view` | Zwraca zakres jako widok:<br/> + `rng` jeśli jest to widok<br/> + `ref_view` jeśli `rng` jest lvalue<br/> + zwraca `owning_view` jeśli `rng` jest rvalue |
| `counted(rng)`     | wariant: <br/> + `std::span`<br/> + `subrange`                    | Tworzy widok z iteratora i licznika elementów<br/> + zwraca `std::span` jeśli `rng` jest ciągłym i wspólnym zakresem<br/> + lub zwraca `subrange`        |
| `iota(fst)`        | `iota_view`                                                       | Tworzy widok z nieskończoną sekwencją liczb całkowitych [fst, infinity)                                                                                  |
| `iota(fst, lst)`   | `iota_view`                                                       | Tworzy widok ze skończoną sekwencją liczb całkowitych [fst, lst)                                                                                         |
| `single(val)`      | `single_view`                                                     | Tworzy widok z tylko jednym elementem `val`                                                                                                              |
| `empty<T>`         | empty_view                                                        | Tworzy pusty widok elementów typu `T`                                                                                                                    |
| `take(num)`        | varies                                                            | The first `num` elements                                                                                                                                 |
| `take_while(pred)` | `take_while_view`                                                 | All leading elements that match a predicate                                                                                                              |
| `drop(num)`        | varies                                                            | All except the first `num` elements                                                                                                                      |
| `drop_while(pred)` | `drop_while_view`                                                 | All except leading elements that match a predicate                                                                                                       |
| `filter(pred)`     | `filter_view`                                                     | All elements that match a predicate                                                                                                                      |
| `transform(func)`  | `transform_view`                                                  | The transformed values of all elements                                                                                                                   |
| `elements<idx>`    | `elements_view`                                                   | The idxth member/attribute of all elements                                                                                                               |
| `keys`             | `elements_view`                                                   | The first member of all elements                                                                                                                         |
| `values`           | `elements_view`                                                   | The second member of all elements                                                                                                                        |
| `reverse`          | varies                                                            | All elements in reversed order                                                                                                                           |
| `join`             | `join_view`                                                       | All elements of multiple ranges                                                                                                                          |
| `split(sep)`       | `split_view`                                                      | All elements of a range split into sub-ranges                                                                                                            |
| `lazy_split(sep)`  | `lazy_split_view`                                                 | All elements of an input/const range split into subranges                                                                                                |
| `common`           | varies                                                            | All elements with same type for iterator and sentinel                                                                                                    |

### Tworzenie widoków

* Przekazanie zakresu jako parametru do adaptora |

``` c++
auto first_3 = std::views::take(coll, 3);
```

* Pipe zakresu z adaptorem |

``` c++
auto first_3 = coll | std::views::take(3);
```

* Przekazanie zakresu jako parametru do typu widoku |

``` c++
std::ranges::take_view first_3{coll, 3};
```

### Widoki i semantyka referencji

* Widoki mają semantykę referencji - odnoszą się do zakresów przechowujących dane
* Użycie widoku jest bezpieczne tylko wtedy, gdy referencje do nich są prawidłowe

``` c++
auto get_values()
{
    std::vector coll{1, 2, 3, 4, 5};
    return coll | std::views::drop(2); 
}

auto rng = get_values(); // !!!ERROR!!!: returning reference to local range
```

Przykłady widoków ilustrujące zachowanie referencyjne:

``` c++
std::vector vec = {3, 2, 1, 5, 4, 7, 6, -1, 9, 10};
print(vec); // 3, 2, 1, 5, 4, 7, 6, -1, 9, 10

std::ranges::subrange rng{coll.begin(), EndValue< -1 >{}};
print(vec); // 3, 2, 1, 5, 4, 7, 6, -1, 9, 10

std::ranges::sort(rng); // sorts elements up to -1
print(vec); // 1, 2, 3, 4, 5, 6, 7, -1, 9, 10

std::views::drop(rng, 2)[0] += 2;
print(vec); // 1, 2, 5, 4, 5, 6, 7, -1, 9, 10
```

### Widoki i modyfikacje elementów

* Widoki są obiektami *read-only*, *stateless* i *equality preserving*
* Nigdy nie powinny modyfikować przekazanego argumentu
* Funkcje pomocnicze (predykaty lub funkcje transformujące) **nigdy** nie powinny modyfikować argumentów

``` c++
std::views::transform([] (auto& val) { // better declare val as const&
    ++val; // ERROR: undefined behavior
})
```

* Jest dozwolone użycie widoków w celu ograniczenia zakresu elementów, które chcemy modyfikować

``` c++
for (auto& item : rng | std::views::drop(5))
{
    item = 0;
}
```

### Widoki i leniwa ewaluacja

* Widoki wykorzystują model *pull*
* Przetwarzanie (filtrowanie) jest wykonywane dopiero w momencie, kiedy iterujemy po elementach widoku (*lazy evaluation*)
* W rezultacie widoki mogą operować na nieskończonych sekwencjach

### Widoki - caching

* W celu optymalizacji wydajności widoki częściowo cache'ują rezultat swojego działania (np. wywołania `begin()`)
* Po wywołaniu `begin()` modyfikacja początkowych elementów zakresu może unieważnić widok
* Odczyt widoku może mieć wpływ na to czy widok jest prawidłowy, jeśli odwołuje się do modyfikowanego zakresu
* Caching może wymagać aby widok, po którym iterujemy nie był `const`
  * w rezultacie nie można iterować po widoku, który jest przekazany do funkcji jako `const auto&`

``` c++
std::list coll{1, 2, 3, 4, 5};
auto v = coll | std::views::drop(2);
print(v); // init begin() with 3

coll.push_front(0); // coll now is: 0 1 2 3 4 5
print(v); // begin() is still 3, so prints: 3 4 5
```

### Fiasko const&

Funkcje szablonowe, które używają jako parametru `const T&` lub `const auto&` nie są kompatybilne z niektórymi widokami:

``` c++
void print(const auto& coll); // not callable for all views

void print(const std::ranges::input_range auto& coll); // not callable for all views

template<std::ranges::random_access_range T>
void foo(const T& coll); // not callable for all views
```

Dla wszystkich widoków działa `auto&&` - *uniwersalna referencja*

``` c++ code-noblend
void print(auto&& rng)
{
    for (const auto& elem : rng) 
    {
        std::cout << elem << ' ';
    }
    std::cout << '\n';
}

std::vector vec{1, 2, 3, 4, 5, 6, 7, 8, 9};
print(vec | std::views::take(3)); // OK
print(vec | std::views::drop(3)); // OK

std::list lst{1, 2, 3, 4, 5, 6, 7, 8, 9};
print(lst | std::views::take(3)); // OK
print(lst | std::views::drop(3)); // OK
```

### Widoki - const_iterator & cbegin() 

* Widoki nie dostarczają typu `const_iterator`

``` c++ code-noblend
for(decltype(vw)::const_iterator it = vw.begin(); // ERROR for a view
    it != vw.end(); ++it)
{
    do_something(*it);
}
```

* Nie posiadają metod `cbegin()` i `cend()`

``` c++ code-noblend
some_algorithm(vw.cbegin(), vw.cend()); // ERROR for a view
```

```{warning}
Funkcje `std::cbegin()` i `std::cend()` nie działają prawidłowo dla widoków!!!

```c++ 
for(auto it = std::cbegin(rng); it != std::cend(rng); ++it)
{
    some_function(*it); // it does not respect the constness
}
```

```

### Podstawowe widoki

* `std::views::all(rng)` - jeśli mamy pojedynczy obiekt range
  * zwraca `rng`
    * lub `std::ranges::ref_view{rg}`
    * lub `std::ranges::owning_view{rg}`

* `std::views::counted(it, count)` - jeśli mamy iterator i liczbę elementów
  * zwraca `std::span{rg}`
  * lub `std::ranges::subrange{rg}`

* Jeśli mamy parę iteratorów
  * `std::subrange{begin, end}`
  * lub `std::span{begin, end}` - w przypadku, gdy pamięć jest ciągła
  

### Użycie widoków

``` c++
std::vector vec{665, 345, 42, 1, 4, -1, 55, 69};
print(vec);

std::ranges::sort(std::views::take(vec, 4)); // sort first 4 elements 
print(vec);

std::ranges::sort(std::views::drop(vec, 3)); // sort elements starting from 4th
print(vec);

std::ranges::sort(std::views::counted(vec.begin() + 1, 5), std::greater{});
print(vec);

print(std::views::transform(vec, [](int x) { return x * x;})); // sort middle items

print(std::views::take(vec, 3));
```

## Widoki - pipelines

``` c++
std::vector vec{665, 345, 42, 1, 4, -1, 55, 69};
print(vec);

std::ranges::sort(vec | std::views::take(vec, 4)); // sort first 4 elements 
print(vec);

std::ranges::sort(vec | std::views::drop(vec, 3)); // sort elements starting from 4th
print(vec);

std::ranges::sort(std::views::counted(vec.begin() + 1, 5), std::greater{});
print(vec);

print(vec 
    | std::views::transform([](int x) { return x * x;}) 
    | std::views::drop(3)                               
    | std::views::take(3));                             
```

## Ranges - nowe typy iteratorów

### std::counted_iterator

* Iterator umożliwiający iterację po maksymalnej liczbie elementów:

``` c++
for (std::counted_iterator pos{coll.begin(), 5}; pos.count() > 0; ++pos) 
{
    std::cout << *pos << '\n';
}
```

### std::default_sentinel

* Iterator, który nie implementuje żadnych operacji
* Typ: `std::default_sentinel_t`
* Pełni rolę `dummy sentinel` - jest wykorzystywany przez inne iteratory (np. `counted_iterator`) do implementacji sprawdzenia, czy jest już koniec sekwencji

``` c++
for (std::counted_iterator pos{coll.begin(), 5}; pos != std::default_sentinel; ++pos) 
{
    std::cout << *pos << '\n';
}
```

### std::unreachable_sentinel

* Wartownik, który nie może zostać osiągnięty
* Używany do implementacji nieskończonych zakresów
* Lub w sytuacji, gdy jesteśmy pewni, że nigdy nie dojdziemy do końca zakresu

``` c++
std::vector vec = {4, 5, 665, 42, 55, 667};

auto pos_42 = std::ranges::find(vec.begin(), std::unreachable_sentinel, 42);
```

### std::common_iterator

* Używany, aby zharmonizować typ dwóch iteratorów

``` c++
template <typename TBegin, typename TEnd>
void some_algorithm(TBegin bgn, TEnd end)
{
    if constexpr(std::same_as<TBegin, TEnd>)
    {
        algorithm(bgn, end);
    }
    else 
    {
        using TIterator = std::common_iterator<TBegin, TEnd>;
        algorithm(TIterator{bgn}, TIterator{end});
    }
}
```

### Borrowed iterator

* Wiele algorytmów zwraca iterator wskazujący na element(y) zakresu

``` c++
std::vector get_data(); // fwd declaration

auto pos = find(get_data(), 665);

std::cout << *pos << '\n';
```

* Aby chronić użytkownika przed *wiszącymi iteratorami* (*dangling iterators*) biblioteka wprowadza koncept *borrowed iterator*
* Używając typu `std::ranges::borrowed_iterator_t<>` algorytmy mogą zadeklarować zwracany iterator jako *pożyczony* (*borrowed*) - taki iterator może być bezpiecznie użyty po wywołaniu funkcji

* Przykładowa deklaracja algorytmu `find()` w bibliotece Ranges

``` c++
template<std::ranges::input_range Rg,
typename T,
typename Proj = identity>
...
constexpr std::ranges::borrowed_iterator_t<Rg>
find(Rg&& r, const T& value, Proj proj = {});
```

* Pozwala na sprawdzenie na etapie kompilacji, czy przekazywany zakres jest *prvalue*, jeśli tak, to zwracany typ staje się *wiszącym iteratorem* i powoduje błąd kompilacji:

``` c++
std::vector get_data(); // fwd declaration

auto pos = std::ranges::find(get_data(), 665); // returns iterator to temporary vector
std::cout << *pos << '\n'; // compile-time ERROR
```

* Wersja OK - ale może być problem z iteracją

``` c++
const auto& data = get_data();
auto pos = std::ranges::find(data, 665);
std::cout << *pos << '\n'; // OK
```

* Wersja OK - uniwersalna referencja

``` c++
auto&& data = get_data();
auto pos = std::ranges::find(data, 665);
std::cout << *pos << '\n'; // OK
```

## Borrowed ranges

* Typy zakresów też mogą być *borrowed*, co oznacza, że zwrócone iteratory są wciąż prawidłowe, nawet jeśli sam zakres już nie istnieje
* W ogólności kontenery oraz większość widoków **nie są** *borrowed*

### Borrowed views

* Cała informacja jest przechowywana w iteratorach
  * `std::ranges::iota_view`
  * `std::ranges::empty_view`

* Widoki, które odwołują się do innych zakresów (iteratory bezpośrednio wskazują na zakres)
  * `std::ranges::subrange`
  * `std::ranges::ref_view`
  * `std::span`
  * `std::string_view`

````{warning}
Iteratory borrowed mogą zostać iteratorami wiszącymi, jeśli zakres do którego się odwołują przestanie istnieć
Efektem jest albo błąd kompilacji:

``` c++
auto pos = std::ranges::find(std::views::take(std::vector {0, 8, 15}, 2), 8);
std::cout << *pos << '\n'; // COMPILER ERROR         
```

lub, niestety, runtime-error:

``` c++
auto pos = std::ranges::find(std::views::counted(std::vector {1, 2, 3, 4}.begin(), 3), 2);
std::cout << *pos << '\n'; // RUNTIME ERROR
```
````