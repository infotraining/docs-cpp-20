# Biblioteka Ranges

Biblioteka Ranges wprowadza nowy sposób transformacji kolekcji danych. Zaletą nowego podejścia jest łatwa możliwość kompozycji elementów transformujących.

Klasyczne algorytmy STL nie były łatwe w kompozycji:

```c++
std::vector<int> input = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
std::vector<int> intermediate, output;

std::copy_if(input.begin(), input.end(), std::back_inserter(intermediate),
             [](const int i) { return i % 3 == 0; });

std::transform(intermediate.begin(), intermediate.end(), 
               std::back_inserter(output), [](const int i) {return i * i; });
```

Od C++20 możemy otrzymać ten sam wynik w dużo prostszy sposób:

```c++
std::vector<int> input = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

auto output = input
    | std::views::filter([](const int n) {return n % 3 == 0; }) 
    | std::views::transform([](const int n) {return n * n; });  
```

## Definicja zakresu - koncept std::range

W ogólności zakres jest "czymś", po czym można iterować. 

Zakres jest reprezentowany przez:

* **iterator**, który określa początek zakresu
* i **wartownika** (*sentinel*), który pozwala określić koniec zakresu

```{attention}
Iterator i wartownik mogą (ale nie muszą) być obiektami różnych typów !!!
```

Biblioteka Ranges definiuje koncept `range` w następujący sposób:

```c++
namespace std::ranges
{
    template<class T>
    concept range = requires(T& rg)
    {
        ranges::begin(rg);
        ranges::end(rg);
    };
}
```

## Komponenty biblioteki Ranges

W skład biblioteki Ranges wchodzą:

* Algorytmy
* Koncepty
* Narzędzia pomocnicze
  * funkcje, punkty kastomizacji (*customization points*), typy, cechy typów
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
| `viewable_range`      | Range that is convertible to a view (with `std::ranges::all()`)     |
| `borrowed_range`      | Range with iterators that are not tied to the lifetime of the range |
| `common_range`        | Ranges where begin and end (sentinel) have the same type            |

Użycie konceptów z biblioteki Ranges:

``` c++
void print(auto& obj) // #1
{
    std::cout << obj << "\n";
}

void print(std::ranges::input_range auto const& coll) // #2
{
    for(auto const& item : coll)
        std::cout << item << " ";
    std::cout << "\n";
}

print(665); // calls #1

std::vector vec{1, 2, 3};
print(vec); // calls #2 - more constrained with std::ranges::input_range

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

#### `std::ranges::range_value_t<>`

Do najczęściej wykorzystywanych typów pomocniczych należy trait `std::ranges::range_value_t<>`, który umożliwia określenie typu elementów zakresu: 

```c++
using namespace std;

vector<int> v{1,2,3};
ranges::range_value_t<decltype(v)> elementType = v[2]; // elementType is an int 

cout << elementType << endl; // outputs 3
cout << typeid(elementType).name() << endl; // outputs int
```

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

*Customization point object* jest obiektem funkcyjnym typu literalnego, który wykonując operacje na obiekcie typu zdefiniowanego w programie (lub w bibliotece) sprawdza wymagania składniowe lub semantyczne określone dla tych operacji.

```c++
template <std::ranges::range Rng>
auto only_unique(Rng&& rng)
{
    // copy a range to a vector
    std::vector data(std::ranges::begin(rng), std::ranges::end(rng));

    // sort
    std::ranges::sort(data);
    
    // unique-erase idiom with ranges
    const auto unique_items = std::ranges::unique(data);
    data.erase(unique_items.begin(), unique_items.end());

    return data;
}
```

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

````{warning}
Funkcje `std::cbegin()` i `std::cend()` nie działają prawidłowo dla widoków!!!

```c++ 
for(auto it = std::cbegin(rng); it != std::cend(rng); ++it)
{
    some_function(*it); // it does not respect the constness
}
```

````

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

## Borrowed iterator

Wiele algorytmów zwraca iterator wskazujący na element(y) zakresu:

``` c++
std::vector<int> data{1, 2, 665, 234};
std::vector<int> get_data() { return {1, 2, 665, 234}; } 

auto it_ok = find(data, 665); // passing lvalue - ok
auto it_evil = find(get_data(), 665); // passing rvalue - it_evil is dangling
```

Aby chronić użytkownika przed *wiszącymi iteratorami* (*dangling iterators*) biblioteka wprowadza tzw. *borrowed iterator*.

Używając typu `std::ranges::borrowed_iterator_t<Rng>` algorytmy:

* mogą zadeklarować zwracany iterator jako *pożyczony* (*borrowed*) - taki iterator może być bezpiecznie użyty po wywołaniu funkcji
* lub zadeklarować zwracany iterator jako *wiszący* (*dangling*) - taki iterator powoduje błąd kompilacji

```c++
template< std::ranges::range R >
using borrowed_iterator_t = 
    std::conditional_t<std::ranges::borrowed_range<R>,
        std::ranges::iterator_t<R>, 
        std::ranges::dangling>;
```

Przykładowa deklaracja algorytmu `find()` w bibliotece Ranges może w uproszczeniu wyglądać następująco:

``` c++
template<std::ranges::input_range Rng, typename T, typename Proj = identity>
constexpr std::ranges::borrowed_iterator_t<Rng> find(Rng&& r, const T& value, Proj proj = {});
```

Taka deklaracja pozwala na sprawdzenie na etapie kompilacji, czy przekazywany zakres jest *prvalue*, jeśli tak, to zwracany typ staje się *wiszącym iteratorem* i powoduje błąd kompilacji:

``` c++
std::vector<int> get_data() { return {1, 2, 665, 234}; } 

auto it_evil = find(get_data(), 665); // ERROR! it_evil is std::ranges::dangling
```

Aby uniknąć wiszącego wskaźnika można przypisać rezultat wywołania funkcji `get_data()` do:

* lokalnej referencji `const auto&`

``` c++
const auto& data = get_data();
auto pos = std::ranges::find(data, 665);
std::cout << *pos << '\n'; // OK
```

* lub uniwersalna referencji `auto&&`

``` c++
auto&& data = get_data();
auto pos = std::ranges::find(data, 665);
std::cout << *pos << '\n'; // OK
```

## Borrowed ranges

Wiszące iteratory są potencjalnym problemem dla wielu zakresów. Iteratory kontenerów standardowych stają się iteratorami wiszącymi, gdy niszczony jest kontener, do którego się odwołują.

```c++
auto iter = std::vector{1, 2, 3}.begin(); // iter is dangling
```

Ale nie dotyczy to wszystkich zakresów. Iteratory do `std::string_view` pozostają prawidłowe (odnoszą się do poprawnego bufora znaków) nawet po zniszczeniu samej instancji `std::string_view` (pod warunkiem, że obiekt, do którego odwoływała się instancja `std::string_view` jeszcze istnieje):

```c++
const char* text = "BORROWED";

auto iter = std::string_view{text}.begin();
assert(*iter == 'B'); // OK & safe
```

Typy takie jak `std::string_view` nazywane są **borrowed ranges**. Termin ten oznacza oznacza, że iteratory zwrócone z takiego zakresu są wciąż prawidłowe, nawet jeśli sama instancja zakresu już nie istnieje.

W rezultacie:

* *rvalue* typu `std::vector<T>` - `std::vector<T>&&` - nie jest *borrowed*
* *rvalue* typu `std::string_view` - `std::string_view<T>` - jest *borrowed*

W bibliotece Ranges istnieje koncept `std::ranges::borrowed_range`, który wygląda następująco:

```c++
template<class Rng>
concept borrowed_range =
    ranges::range<Rng> &&
    (std::is_lvalue_reference_v<Rng> ||
     ranges::enable_borrowed_range<std::remove_cvref_t<Rng>>);

template<class Rng>
inline constexpr bool enable_borrowed_range = false;
```

W praktyce oznacza to, że zakres jest *borrowed* jeśli:

* jest lvalue
* lub jest rvalue typu, dla którego specjalizacja `std::ranges::enable_borrowed_range` zwraca `true`
  * cała informacja jest przechowywana w iteratorach
    * `std::ranges::iota_view`
    * `std::ranges::empty_view`
  * typy "widoków", które odwołują się do innych zakresów
    * `std::ranges::subrange`
    * `std::ranges::ref_view`
    * `std::span`
    * `std::string_view`

Jeśli w aplikacji zdefiniowany jest typ zakresu, dla którego iteratory mogą bezpiecznie wisieć (np. `StringRef`), to możemy dostarczyć specjalizację, która
pozwoli bibliotece Ranges traktować ten typ zakresu jako *borrowed*:

```c++
template <>
inline constexpr bool std::ranges::enable_borrowed_range<my::StringRef> = true;
```

## Viewable ranges

Biblioteka Ranges definiuje dwa niezależne koncepty:

* `std::ranges::borrowed_range` - zakres, którego iteratory nie wiszą (zakres lvalue lub zakres rvalue z specjalizacją `enable_borrowed_range`)
* `std::view` - zakres, dla którego operacje kopiowania/przenoszenia/niszczenia mają złożoności O(1)

Funkcja `std::views::all()` spina obie koncepcje razem - przyjmuje jako parametr *borrowed range* i zwraca widok (`ref_view` lub `subrange`).

Razem, obie kategorie typów są nazywane **viewable ranges** i mogą stać po stronie operatora "pipe" `|`:

```c++
auto vec = get_vector();

auto v1 = vec | views::transform(func); // OK: vec is an lvalue

auto v2 = get_span() | views::transform(func); // OK: span is borrowed

auto v3 = subrange(vec.begin(), vec.end()) | views::transform(func) // Ok: subrange is borrowed (and a view)

auto v4 = get_vector() | views::transform(func); // ERROR: get_vector() returns an rvalue vector, which is neither
                                                 // a view nor a borrowed range
```



````{warning}
Iteratory borrowed zostają iteratorami wiszącymi, jeśli zakres do którego się odwołują nie jest *borrowed*
Efektem jest:

* błąd kompilacji:

``` c++
auto pos = std::ranges::find(std::views::take(std::vector {0, 8, 15}, 2), 8);
std::cout << *pos << '\n'; // COMPILER ERROR         
```

* lub, niestety, runtime-error:

``` c++
auto pos = std::ranges::find(std::views::counted(std::vector {1, 2, 3, 4}.begin(), 3), 2);
std::cout << *pos << '\n'; // RUNTIME ERROR
```