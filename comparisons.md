# Porównania

## Bezpieczne porównania liczb całkowitych

* Porównania liczb całkowitych mogą dawać zaskakujące rezultaty (efekt konwersji typów)

``` c++
int x = -42;
unsigned y = 665;

if (x < y) // Ups!
    do_something();
```

* Od C++20 mamy dostępną rodzinę funkcji umożliwiających bezpieczne porównania:

``` c++
#include <utility>

if (std::cmp_less(x, y)) // now OK
    do_something();

if (std::in_range<uint16_t>(x))
   do_something();
```

### Funkcje cmp_***

|            Funkcja             |                    Efekt                       |
| :----------------------------: | :--------------------------------------------: |
|     `std::cmp_equal(x, y)`     |         Yields whether x is equal to y         |
|   `std::cmp_not_equal(x, y)`   |       Yields whether x is not equal to y       |
|     `std::cmp_less(x, y)`      |        Yields whether x is less than y         |
|  `std::cmp_less_equal(x, y)`   |  Yields whether x is less than or equal to y   |
|    `std::cmp_greater(x, y)`    |       Yields whether x is greater than y       |
| `std::cmp_greater_equal(x, y)` | Yields whether x is greater than or equal to y |
|     `std::in_range<T>(x)`      |  Yields whether x is a valid value for type T  |

## Porównania w C++20

C++20 zmienia sposób pisania operatorów porównań w klasach/strukturach

* Mamy nowy dwuargumentowy `operator <=>` umożliwiający przeprowadzenie porównań porządkujących (`<`, `>`, `<=`, `>=`)
* Kompilator jest w stanie przepisać wyrażenia z operatorami porównań - *rewriting expressions*
* Dodatkowo `operator ==` oraz `operator <=>` można zdefiniować jako `default`

### Równość i porządek w C++20

C++20 do ustalania równości używa `operator ==` - jest on używany w operacjach `==` i `!=`.

Nowy `operator <=>` pozwala zdefiniować porządek obiektów tzw. *ordering*. Zdefiniowany operator `<=>` może być wykorzystany operacjach: `<`, `>`, `<=`, `>=`

### Sprawdzanie równości - `==`

* Aby sprawdzić równość (lub nierówność) dwóch obiektów **wystarczy** zaimplementować `operator==`
* Jeśli kompilator nie znajdzie pasującej deklaracji dla wyrażenia `a!=b`, to spróbuje przepisać wyrażenie:
  * do postaci `!(a==b)`
  * lub `!(b==a)`

``` c++
Type a, b;

a != b // compiler tries: a != b, !(a == b), !(b == a)
```

* Przepisywanie działa też w sytuacji gdy operandy są różnych typów

``` c++
bool operator==(const TypeA&, const TypeB&);

// lub

class TypeA {
public:
   bool operator==(const TypeB&) const;
};

//...

TypeA a;
TypeB b;

a == b; // OK - perfect match
b == a; // OK - rewritten as: a == b
a != b; // OK - rewritten as: !(a == b)
b != a; // OK - rewritten as: !(a == b) 
```

+++

## Definiowanie porządku - `<=>`

* `operator <=>` jest nowym dwuargumentowym operatorem umożliwiającym wszystkie porównania między obiektami
* Może zostać zdefiniowany jako `default`

``` c++
#include <compare> 

class Value
{
private:
    int value_;

    constexpr Value(int v) noexcept : value_{v}
    {      
    }

    auto operator<=>(const Value& other) const = default;
};
```

### Porównania przed C++20

Przed C++20 aby zdefiniować wszystkie operatory porównań dla klasy, należało zaimplementować 6 funkcji:

``` c++
class Name
{
    std::string last_;
    std::string name_;
public:
    // API...
    
    friend bool operator==(const Name& a, const Name& b) {
        return a.last == b.last && a.first == b.first;
    }

    friend bool operator<(const Name& a, const Name& b) {
        return a.last < b.last 
            || (a.last == b.last && a.first < b.first);
    }

    friend bool operator!=(const Name& a, const Name& b) {
        return !(a == b);
    }

    friend bool operator>(const Name& a, const Name& b) {
        return b < a;
    }

    friend bool operator>=(const Name& a, const Name& b) {
        return !(a < b);
    }

    friend bool operator<=(const Name& a, const Name& b) {
        return !(b < a);
    }
};
```

### Porównania po C++20

Od C++20 wystarczy zdefiniować jedną:

``` c++
class Name
{
    std::string last_;
    std::string name_;
public:
    // API...

    auto operator<=>(const Name& other) const = default;
};
```

### Operator `<=>`

* **Three-way comparison operator**
* Nowy dwuargumentowy operator umożliwiający wszystkie porównania między obiektami
* Może zostać zdefiniowany jako `default`
* Wykonuje tzw. *three-way comparison*, które zwraca wartość, która może być porównana do wartości `0`
  * jeśli `x <=> y == 0`, to `x` i `y` są równe lub równoważne
  * jeśli `x <=> y < 0`, to `x` jest mniejsze od `y`
  * jeśli `x <=> y > 0`, to `x` jest większe od `y`
* Wynik działania wyrażenia `x <=> y` nie jest wartością całkowitą!!!
  * Typ zwracany reprezentuje tzw. **kategorię porównań** - *comparison category*

``` c++
#include <compare> 

class Value
{
private:
    int value_;

    constexpr Value(int v) noexcept : value_{v}
    {      
    }

    auto operator<=>(const Value& other) const = default;

    /***************************************************************************
    auto operator==(const Value& other) const = default; // implicitly generated
    ***************************************************************************/
};
```

#### Domyślne operatory porównań

* Aby zdefiniować `operator<=>` jako `default` należy dodać nagłówek `<compare>`
* Domyślna implementacja deleguje wywołanie operatora `<=>` dla wszystkich klas bazowych i pól składowych
* Są `noexcept` jeśli porównanie składowych nie rzuca wyjątków
* Są `constexpr` jeśli porównanie może odbyć się na etapie kompilacji
* W rezultacie *rewritingu*, dostępne są niejawne konwersje dla pierwszego operandu

#### Użycie operatora `<=>`

* Operator `<=>` służy do implementacji typów
* Programista nigdy nie powinien wywoływać operatora `<=>` bezpośrednio

``` c++
#include <compare>

Value v1{3};
Value v2{4};

auto result = v1 <=> v2; // avoid direct call

bool result = v1 < v2;   // prefer standard comparisons
```

#### Reversing & Rewriting

* Reguły odwracania i przepisywania operatorów zapewniają symetrię i heterogeniczność przy porównaniach

|  Source   |       Alt 1       |      Alt 2       |
| :-------: | :---------------: | :--------------: |
| `a == b`  |     `b == a`      |                  |
| `a != b`  |    `!(a == b)`    |   `!(b == a)`    |
| `a <=> b` | `0 <=> (b <=> a)` |                  |
|  `a < b`  |  `(a <=> b) < 0`  | `0 < (b <=> a)`  |
| `a <= b`  | `(a <=> b) <= 0`  | `0 <= (b <=> a)` |
|  `a > b`  |  `(a <=> b) > 0`  | `0 > (b <=> a)`  |
| `a >= b`  | `(a <=> b) >= 0`  | `0 >= (b <=> a)` |

#### Kategorie porównań dla typów 

* **strong ordering** (aka *total ordering*)
  * dowolna wartość danego typu jest *mniejsza*, *równa* lub *większa* od innej wartości tego typu (uwzględniając samą siebie)
  * wartości:
    * `std::strong_ordering::less`
    * `std::strong_ordering::equal` (`std::strong_ordering::equivalent`)
    * `std::strong_ordering::greater`
* **weak ordering**
  * dowolna wartość danego typu jest *mniejsza*, **równoważna** lub *większa* od innej wartości tego typu (uwzględniając samą siebie)
  * np. `case-insensitive strings`
  * wartości:
    * `std::weak_ordering::less`
    * `std::weak_ordering::equivalent`
    * `std::weak_ordering::greater`
* **partial ordering** 
  * dowolna wartość danego typu może być *mniejsza*, **równoważna** lub *większa* od innej wartości tego typu (uwzględniając samą siebie)
  * ale możliwa jest też sytuacja, że dla dwóch wartości nie można określić porządku
  * np. typy zmiennoprzecinkowe (każde porównanie z `NaN` zwraca `false`)
  * sortowanie obiektów może być niemożliwe (jeśli w zbiorze wartości jest `NaN`)
  * wartości:
    * `std::partial_ordering::less`
    * `std::partial_ordering::equivalent`
    * `std::partial_ordering::greater`
    * `std::partial_ordering::unordered`

``` c++
int x = 42, y = 665;
x <=> y    // yields std::strong_ordering::less 
x <=> 42.0 // yields std::partial_ordering::equivalent
&x <=> &x  // yields std::strong_ordering::equal
&x <=> nullptr // ERROR - three-way comparison with nullptr not supported

std::string{"hi"} <=> "hi" // yields std::strong_ordering::equal
std::pair{42, 0.0} <=> std::pair{42, 7.7} // yields std::partial_ordering::less
```

* Dozwolona jest niejawna konwersja do typu słabszej kategorii:
  * `std::strong_ordering -> std::weak_ordering -> std::partial_ordering`

``` c++
if (x <=> y == std::partial_ordering::equivalent) // always OK
{
    //
}

if (x <=> y == std::strong_ordering::equal) // might not compile
{
    //
}

if (x <=> y == 0) // always OK
{
    //
}
```

#### Typy zmiennoprzecinkowe - operator `<=>`

* Gdy `a` i `b` są `float` lub `double`
  * `a <=> b` daje w wyniku `std::partial_ordering`
    * "normalne" liczby są całkowicie uporządkowane (*totally ordered*)
    * `+0/-0` są traktowane jako równoważne
    * `NaN` są nieuporządkowane
* Częściowe uporządkowanie zwykle nie wystarcza - STL wymaga mocniejszej kategorii
* Pomocne są wtedy szablony:
  * `std::strong_order(a, b)`
    * wywołuje `a <=> b` i zwraca `std::strong_ordering`
    * dla `float` lub `double` - promocja do `std::strong_ordering`  
      * "normalne" liczby są całkowicie uporządkowane (*totally ordered*)
      * `+0/-0` są traktowane jako nie równe
      * `NaN` są całkowicie uporządkowane
  * `std::weak_order(a, b)`
    * wywołuje `a <=> b` zwraca `std::weak_ordering`
    * dla `float` lub `double` - promocja do `std::weak_ordering` 
      * "normalne" liczby są całkowicie uporządkowane (*totally ordered*)
      * `+0/-0` są traktowane jako równe
      * `NaN` są traktowane jako równe, całkowicie uporządkowane względem innych wartości
  * `compare_XXX_order_fallback()` - gdy `<=>` nie jest zdefiniowany ustalają porządek za pomocą operatorów `==` i `<`

#### Domyślny operator `<=>`

* Leksykograficznie porównuje klasy bazowe oraz składowe
* Jeżeli `operator<=>` został zdefiniowany jako domyślny i wywołany został jeden z operatorów porównań, to:
  * jeżeli istnieje zdefiniowany `operator<=>` dla danej składowej lub klasy pochodnej, jest on wywołany
  * w przeciwnym razie wywołane są `operator==` i `operator<` w celu sprawdzenia, czy:
    * obiekty są *równe/równoważne* (`operator==` zwraca wtedy `true`)
    * obiekty są mniejsze (`less`) lub większe (`greater`)
    * obiekty są nieuprządkowane (`unordered`) - tylko w przypadku gdy sprawdzane jest częściowe uporządkowanie (*partial_ordering*)



``` c++
struct Base
{
    std::string value;

    bool operator<=>(const Base& other) const = default;
};

struct Derived : Base
{
    std::vector<int> data;

    auto operator<=>(const Derived& other) const = default;
};

int main()
{
    Derived d1{"compare", {1, 2, 3}};
    Derived d2{"compare", {1, 2, 4}};

    assert(d2 > d1);
    static_assert(std::is_same_v<decltype(d1 <=> d2), std::strong_ordering>);
}
```

* W przypadku gdy klasa bazowa nie implementuje operatora `<=>` (ale posiada operatory `==` i `<`) i klasa pochodna deklaruje domyślną wersję `<=>` z automatyczną dedukcją typu `auto`, kompilator zgłosi błąd, ponieważ nie jest w stanie określić kategorii porównania:

``` c++
struct Base
{
    std::string value;

    bool operator==(const Base& other) const { return value == other.value; }
    bool operator<(const Base& other) const { return value < other.value; }
};

struct Derived : Base
{
    std::vector<int> data;

    auto operator<=>(const Derived& other) const = default; // ERROR
};

int main()
{
    Derived d1{"compare", {1, 2, 3}};
    Derived d2{"compare", {1, 2, 4}};

    assert(d2 > d1);
    static_assert(std::is_same_v<decltype(d1 <=> d2), std::strong_ordering>);
}
```

Pomocne w takim wypadku jest jawne określenie kategorii operatora `<=>` w klasie pochodnej:

``` c++
struct Base
{
    std::string value;

    bool operator==(const Base& other) const { return value == other.value; }
    bool operator<(const Base& other) const { return value < other.value; }
};

struct Derived : Base
{
    std::vector<int> data;

    std::strong_ordering operator<=>(const Derived& other) const = default;
};

int main()
{
    Derived d1{"compare", {1, 2, 3}};
    Derived d2{"compare", {1, 2, 4}};

    assert(d2 > d1);
    static_assert(std::is_same_v<decltype(d1 <=> d2), std::strong_ordering>);
}
```

#### Implementacja operatora `<=>`

##### Składowa floating point
 
W przypadku, gdy jako typ zwracany deklarujemy `auto`, dedukowaną kategorią porównanie jest `std::partial_ordering`:

``` c++
struct Temperature
{
    double value;

    auto operator<=>(const Temperature& other) const = default;
};

//...

auto cmp_result = Temperature{10.0} <=> Temperature{20.0};
assert(cmp_result == std::partial_ordering::less);
static_assert(std::is_same_v<decltype(cmp_result), std::partial_ordering>);
```

Możliwa jest jawna promocja do kategorii `std::strong_ordering` przy pomocy funkcji `std::strong_order()`:

``` c++
struct Temperature
{
    double value;

    std::strong_ordering operator<=>(const Temperature& other) const
    {
        return std::strong_order(value, other.value);
    }

    bool operator==(const Temperature& other) const = default;
};

//...

auto cmp_result = Temperature{10.0} <=> Temperature{20.0};
assert(cmp_result == std::strong_ordering::less);
static_assert(std::is_same_v<decltype(cmp_result), std::strong_ordering>);
```

##### Przypadek ogólny

Aby zaimplementować operator `<=>` dla typu, który posiada wiele składowych musimy zwykle zaimplementować ciąg porównań z wykorzystaniem `<=>`:

``` c++
class Person
{
    int id_;
    std::string name_;

    auto operator<=>(const Person& other) const // yields std::strong_ordering
    {
        auto cmp_id = id_ <=> other.id_; 
        if (cmp_id != 0) 
            return cmp_id;
        return name_ <=> other.name_;
    }
};
```

##### Konflikt kategorii

Jeśli atrybuty należą do innych kategorii porównań zgłaszany jest konflikt.

``` c++
class Gadget
{
    std::string name_;
    double price_;

    auto operator<=>(const Gadget& other) const // ERROR - different types deduced
    {
        auto cmp_name = name_ <=> other.name_; 
        if (cmp_name != 0) 
            return cmp_name; // returns strong_ordering for string
        return price_ <=> other.price_; // returns partial_ordering for double
    }
};
```

Rozwiązaniem konfliktu jest:

* konwersja do najsłabszego kryterium porównania:

``` c++
class Gadget
{
    std::string name_;
    double price_;

    std::partial_ordering operator<=>(const Gadget& other) const
    {
        auto cmp_name = name_ <=> other.name_; 
        if (cmp_name != 0) 
            return cmp_name; // strong_ordering converted to partial_ordering
        return price_ <=> other.price_; // returns partial_ordering for double
    }
};
```

* wykorzystanie cechy `std::common_comparison_category<>`

``` c++
class Gadget
{
    std::string name_;
    double price_;

    auto operator<=>(const Gadget& other) const
        -> std::common_comparison_category_t<
                decltype(name_ <=> other.name_),
                decltype(price_ <=> other.price_)>
    {
        auto cmp_name = name_ <=> other.name_; 
        if (cmp_name != 0) 
            return cmp_name; // converted to common_comparison_category_t
        return price_ <=> other.price_; // converted to common_comparison_category_t
    }
};
```

* lub jeśli chcemy dostarczyć silniejszej kategorii porównania, to **zmapowanie** zwracanych kategorii z porównań składowych na **kategorię docelową** (z uwzględnieniem obsługi błędów w runtimie)

``` c++
class Gadget
{
    std::string name_;
    double price_;

    std::strong_ordering operator<=>(const Gadget& other) const
    {
        auto cmp_name = name_ <=> other.name_; 
        if (cmp_name != 0) 
            return cmp_name; // return strong_ordering
        
        auto cmp_price = price_ <=> other.price_; // std::partial_ordering
        
        // map partial_ordering -> strong_ordering
        assert(cmp_price != std::partial_ordering_unordered); // RUNTIME ERROR
        
        return cmp_price == 0 ? std::strong_ordering::equal : 
                                cmp_price > 0 ? std::strong_ordering::greater
                                              : std::strong_ordering::less;
    }
};
```

#### Operator `<=>` - kod generyczny

W kodzie szablonowym `operator <=>` wymaga szczególnej uwagi. Zamiast bezpośrednio wywoływać operator `<=>` lepiej użyć:

* obiektu funkcyjnego `std::compare_three_way`, który wywołuje `operator <=>` dla podanych argumentów, ale dla typów wskaźnikowych gwarantuje silne uporządkowanie (*strict total order*)

* lub `std::strong_order`

``` c++
template <typename T>
struct Wrapper
{
    T value;

    auto operator<=>(const Wrapper& v) const noexcept(noexcept(v <=> v))
    {
        return std::compare_three_way{}(value, v); 
               // defines a total order for raw pointers 
               // (which is not the case for operators <=> or <)        
    }

    bool operator==(const Wrapper& other) const = default;
};
```

Gdy elementy, które chcemy porównać są przechowywane w iterowalnym obiekcie (np. kontenerze lub liście inicjalizacyjnej), to należy użyć algorytmu `std::lexicographical_compare_three_way()`:


``` c++
template <typename T> 
struct Data
{
    std::vector<T> values;

    Data(std::initializer_list<T> il) : values{il}
    {}

    auto operator<=>(const Data& other) const
    {
        return std::lexicographical_compare_three_way(
            values.begin(), values.end(), 
            other.values.begin(), other.values.end(), std::strong_order);
    }
};
```
