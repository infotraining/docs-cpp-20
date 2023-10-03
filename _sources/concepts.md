# Koncepty i ograniczenia szablonów

## Koncepty - słownik

* **Requirement / Wymaganie**
  * Wyrażenie definiujące wymaganie dla kodu
    * operacja musi być prawidłowa składniowo
    * określony typ musi być zdefiniowany (lub zwrócony)

* **Concept / Koncept**
  * Nazwa dla jednego lub wielu wymagań

* **Constraint / Ograniczenie**
  * Ograniczenia dla kodu generycznego (funkcji lub klasy)
    * definiowane z użyciem konceptów lub wymagań ad-hoc
    * wyrażenie logiczne ewaluowane na etapie kompilacji

## Koncept

* jest szablonem
* nazwanym zbiorem wymagań zdefiniowanych dla parametrów szablonu
* id-expression ewaluowane do wartości logicznej `bool`
* musi być zdefiniowany na poziomie przestrzeni nazw
* nie może być rekursywny
* nie są dozwolone żadne specjalizacje (oryginalna definicja nie może być zmieniana)

Koncepty najczęściej mogą być definiowane przy pomocy traits'ów:

``` c++
template <typename T>
concept Integral = std::is_integral_v<T>;

template <typename T>
concept SignedIntegral = Integral<T> && std::is_signed_v<T>;

template <typename T>
concept UnsignedIntegral = Integral<T> && !SignedIntegral<T>;
```

oraz przy pomocy wyrażenia `requires`:

```c++
template<typename T>
concept Hashable = requires(T a) {
    { std::hash<T>{}(a) } -> std::convertible_to<std::size_t>;
};
```

### Użycie konceptów

Koncepty mogą być używane jako:

* nazwane wyrażenia - *id-expression*
  
  ```c++
  static_assert(Hashable<Gadget>, "type must be hashable");
  ```

* deklaracje typów parametrów szablonów
  
  ```c++
  template <std::integral T>
  T square(T x)
  {
      return x * x;
  }
  ```

* ograniczenia dla placeholdera `auto`

  ```c++
  std::integral auto square(std::integral auto x)
  {
      return x * x;
  }
  ```

* składnik definicji złożonych wymagań dla typów
 
  ```c++
  template <typename T>
  concept PrintableRange = std::ranges::range<T> && requires {
      std::cout << std::declval<std::ranges::range_value_t<T>>();
  };
  
  void print(PrintableRange auto const& rng)
  {
      for(const auto& item : rng)
          std::cout << item << " ";
      std::cout << "\n";
  }
  
  print(std::vector{1, 2, 3});
  
  print(std::map<int, std::string>{ {1, "one"}, {2, "two"} }); // ERROR! `std::map<int, std::string>` nie spełnia wymagań definiowanych przez koncept
  ```

### Użycie konceptów w klasach

Koncepty mogą być używane w klasach szablonowych:

* jako ograniczenie parametru szablonu

``` c++
template <Integral T>    
class C2
{};
```

* jako ograniczenie dla specjalizacji częściowej

``` c++
template <typename T>  // primary template
class C3
{};

template <Integral T>  // partial specialization
class C3<T>
{};
```

* jako klauzula requires w metodach klasy

``` c++
template <typename T>
class C4
{
    void foo() const requires Integral<T>;
};
```

``` c++
C4<int> c4i;
c4i.foo(); // OK

C4<double> c4d;
c4d.foo(); // ERROR - double is not integral type
```

### Koncepty + placeholder auto

Od C++20 możemy poprzedzić deklarację z placeholderem `auto` lub `decltype(auto)` konceptem ograniczającym:

``` c++
auto get_id()
{
    static unsigned int id_gen{};
    return ++id_gen;
}

std::unsigned_integral auto id = get_id(); // OK
```

Jeśli przed placeholderem `auto` występuje koncept `C<A...>`, to dedukowany typ `T` musi spełniać ograniczenia zdefiniowane wyrażeniem `C<T, A...>`:

``` c++
std::unsigned_integral auto get_id()
{
    static size_t id_gen{};
    return ++id_gen;
}

std::convertible_to<uint64_t> auto id64 = get_id();
```

Możemy także użyć konceptu, aby wprowadzić ograniczenie dla parametru szablonu, który nie jest typem (NTTP):

```c++
template <typename T, std::integral auto N>
struct Array
{
    T items[N];
    
    //...
};

Array<int, 10> arr1 = {};
Array<int, true> arr2 = {}; // ERROR
```

## Ograniczenia szablonów

Ograniczenie szablonu (*template constraint*) jest sekwencją logicznych wyrażeń (ewaluowanych do `true` lub `false`), które na etapie kompilacji określają czy szablon jest instancjonowany, czy też nie.

Ograniczenia:

* Pomagają **zrozumieć** jakie wymagania muszą zostać spełnione przez parametr szablonu (dzięki temu otrzymujemy lepsze komunikaty błędów)
* Mogą zostać użyte do **zablokowania instancjonowania kodu** w przypadku, gdy nie ma to sensu
* Mogą zostać użyte do **przeładowania** lub **specjalizacji** kodu - różny kod jest kompilowany w zależności od typu

### Ograniczenie

* może być wyrażeniem Boolean (ad-hoc) ewaluowanym na etapie kompilacji
* wyrażeniem `requires` specyfikującym wymagane operacje i typy
* konceptem (zdefiniowanym wcześniej z użyciem słowa kluczowego `concept`)

```c++
template <typename T>
    requires 
        (sizeof(T) > 4) // ad-hoc Boolean expression
        && requires { typename T::value_type; } // requires expression
        && std::input_iterator<T> // concept
```

### Koniunkcja ograniczeń

Koniunkcja ograniczeń tworzona jest przy pomocy operatora `&&`:

``` c++
template <typename T>
constexpr bool get_value() { return T::value; }

template <typename T>
    requires (sizeof(T) > 1) && (get_value<T>())
void f(T);   // #1

void f(int); // #2
```

Gdy wywołujemy:

```c++
f('a');
```

Kompilator sprawdza ograniczenia dla funkcji #1. Lewy operand koniunkcji `sizeof(char) > 1` nie jest spełniony i w efekcie wyrażenie `get_value<T>()` nie jest sprawdzane (*short circuited*). Ponieważ szablon #1 nie spełnia ograniczeń ostatecznie wywołana jest funkcja #2.

### Dysjunkcja ograniczeń

Dysjunkcja ograniczeń tworzona przy pomocy operatora `||` w wyrażeniu ograniczającym. Jest spełniona, jeśli spełniony jest jeden lub drugi operand ograniczenia (*short-circuited*)

```c++
template<typename T>
    requires (std::is_pointer_v<T> || std::is_same_v<T, std::nullptr_t>)
void foo(T ptr)
{
    //...
} 
```

### Wyrażenie requires

* Umożliwia zwięzłe zdefiniowanie wymagań dotyczących argumentów szablonu, które mogą być sprawdzone na etapie kompilacji:

``` c++
template <typename T>
concept PointerLike = 
    requires (T ptr) {
        typename T::element_type;
        { *ptr } -> std::same_as<typename T::element_type&>;
    };
```

* W wyrażeniu `requires` możemy wyspecyfikować:
  * wymaganą definicje typu
  * wyrażenie, które musi być prawidłowe
  * wymagania dla typów zwracanych jako rezultat ewaluacji wyrażenia

#### requires requires

Wyrażenie `requires` często jest używane po klauzuli `requires` w definicji ograniczeń szablonu. Umożliwia to zdefiniowanie ograniczeń ad-hoc:

``` c++
template <typename T>
    requires 
        requires (T x) { x + x; }
T add(T a, T b) { return a + b; }
```

#### Parametry wyrażenia requires

W wyrażeniu `requires` możemy wprowadzić listę lokalnych parametrów:

* parametry nie mogą mieć wartości domyślnych
* lista parametrów nie może kończyć się elipsą `...`
* parametry nie są instancjonowane ani linkowane - służą tylko definicji ograniczeń

``` c++
template <typename T>
concept Indexable = 
    requires (T obj, size_t index) {
        typename T::value_type;
        { obj[index] } -> std::convertible_to<typename T::value_type&>;
    };
```

#### Ewaluacja wyrażenia requires

Wyrażenie `requires` jest wyrażeniem *prvalue* typu `bool`.

* Jeśli podstawienie argumentów szablonu do wyrażenia `requires` skutkuje:
  * nieprawidłowymi typami oraz wyrażeniami
  * lub naruszeniem semantycznych ograniczeń
* to wyrażenie `requires` jest ewaluowane do wartości `false` i nie powoduje, że program jest traktowany jako *ill-formed*

* Jeśli podstawienia typów i weryfikacja semantycznych wymagań przebiegnie pomyślnie wyrażenie jest ewaluowane do wartości `true`

* Sprawdzanie wymagań dokonywane jest w zadeklarowanej kolejności (leksykalnie)

#### Wymagania proste

Proste wymagania sprawdzają poprawność (*well-formed*) podanego wyrażenia:

``` c++
template <typename T>
concept Addable = 
    requires(T a, T b) {
        a + b;
    };
```

Koncept `Addable<T>` jest `true`, jeśli wyrażenie `a + b` jest poprawnym składniowo 
 wyrażeniem.

Przykład kilku prostych wymagań:

```c++
template <typename T1, typename T2>
... requires(T1 p, T2 value) {
    *p; // operator* has to be supported for T2
    p[0]; // operator[] has to be supported for int as an index
    p->get(); // calling a member function get() has to be valid
    *p > value; // comparing the result of operator* with a value of T2 type is possible
    p == nullptr; // support that we can compare a T2 with a nullptr
};
```

Należy uważać na traity w wyrażeniach `requires`:

```c++
template<typename T>
... requires {
        std::integral<T>; // OOPS: does not require T to be integral
        ...
    };
```

W powyższym przykładzie sprawdza, czy wyrażenie `std::integral<T>` jest poprawne składnio, zamiast sprawdzić czy typ `T` spełnia wymaganą cechę (*trait*). 

Aby sprawdzić trait należy użyć konceptu:

```c++
template<typename T>
... std::integral<T> && requires {
        ...
    };
```

lub zastosować wyrażenie `requires`:

```c++
template<typename T>
... requires {
        requires std::integral<T>;
        ...
    };
```

#### Wymagania typu

Wymagania typu sprawdzają poprawność typu:

``` c++
template<typename T, typename T::type = 0> struct S;
template<typename T> using Ref = T&;

template<typename T> concept C = requires {
    typename T::inner; // #1 - required nested member name
    typename S<T>; // #2 - required class template specialization
    typename Ref<T>; // #3 - required alias template substitution, fails if T is void
};
```

* #1 - wymaganie posiadania przez typ `T` zagnieżdżonego typu `inner`
* #2 - wymaganie istnienia specjalizacji szablonu `S<T>` - nie musi być to typ kompletny
* #3 - wymaganie możliwości podstawienia typu `T` do aliasu szablonu `Ref`

Wymagania typu mogą sprawdzać jedynie nazwy nadane typom (klasom, wyliczeniom, aliasom):

```c++
template <typename T>
... requires {
    typename int; // ERROR: invalid type requirement
    typename T&;  // ERROR: invalid type requirement 
}
```

#### Wymagania złożone (Compound requirements)

Wymagania złożone umożliwiają sprawdzenie określonych właściwości wyrażenia poddanego ewaluacji. Sprawdzane wyrażenie umieszczane jest w bloku `{}` a następnie można dodać:

* klauzulę `noexcept`, aby sprawdzić czy wyrażenie nie rzuci wyjątku
* `-> type_constrained`, aby sprawdzić czy typ zwracany z wyrażenia spełnia koncept

``` c++
requires {
    { Expr1 };
    { Expr2} noexcept;
    { Expr3 } -> ConceptA;
    { Expr4 } noexcept -> ConceptB<B1, ..., Bn>;
    //...
};
```

Wymagania z określeniem zwracanego typu wykorzystują koncepty i są rozwijane do:

``` c++
requires {
    { Expr1 };
    { Expr2 } noexcept;
    { Expr3 } -> ConceptA<decltype((Expr3))>;
    { Expr4 } noexcept -> ConceptB<decltype((Expr4)),B1, ..., Bn>;
    //...
};
```

Przykłady wymagań złożonych:

``` c++
template <typename T>
concept PostFixIncrementable = requires (T obj) {
    { obj++; }
};
```

W powyższym przykładzie następuje sprawdzenie, czy `obj++` jest poprawnym wyrażeniem. Jest to równoważne prostemu wymaganiu:

``` c++
template <typename T>
concept PostFixIncrementable = requires (T obj) {
    obj++;
};
```

Inny przykład wymagań złożonych:

``` c++
template <typename T>
concept Indexable = requires (T obj, size_t n) {
    { obj[n] } -> std::same_as<typename T::reference>;
    { obj.at(n) } -> std::same_as<typename T::reference>;
    { obj.size() } noexcept -> std::convertible_to<size_t>;
    { obj.~T() } noexcept;
};
```

#### Wymagania zagnieżdżone

Wymagania można zagnieżdżać, co ilustruje poniższy przykład:

``` c++
template <typename T>
concept AdditiveRange = requires (T&& c) {
    std::ranges::begin(c);
    std::ranges::end(c); 
    typename std::ranges::range_value_t<T>; // type requirement
    requires requires(std::ranges::range_value_t<T> x) { x + x; }; // nested requirement
};

template <AdditiveRange Rng>
auto sum(const Rng& data)
{
    return std::accumulate(std::begin(data), std::end(data), 
        std::ranges::range_value_t<Rng>{});
}

assert(sum(std::vector{1, 2, 3}) == 6);

assert(sum({ "one", "two", "three" }) == "onetwothree"s);
```

### Subsumacja konceptów

Relacja subsumacji zachodzi, gdy dany koncept specyfikuje dodatkowe ograniczenia w stosunku do innego konceptu (lub innych konceptów).

Przy wyborze funkcji ze zbioru funkcji przeciążonych, funkcja generyczna posiadająca więcej ograniczeń jest preferowana w stosunku do funkcji mniej ograniczonej (jeśli spełnione są ograniczenia w obu przypadkach)

W poniższym przykładzie `ShapeWithColor` subsumuje koncept `Shape`:

```c++
template <typename T>
concept Shape = requires(T obj)
{
    obj.draw();
};

template <typename T>
concept ShapeWithColor = Shape<T> &&
    requires (T obj, Color c) {
        obj.set_color(c);
        { obj.get_color() } -> std::convertible_to<Color>;
    };
```

W wyniku subsumacji `ShapeWithColor` konceptu `Shape` nie ma problemu przy wywołaniu funkcji przeciążonych:

```c++ code-noblend
template <Shape T>
void render(T shp)  // #1
{
    shp.draw();
}

template <ShapeWithColor T>
void render(T shp)  // #2
{
    shp.set_color(Color{0, 255, 22});
    shp.draw();
}

struct Rect
{
    int width, height;
    Color color;

    void draw() const
    {
        std::cout << "Drawing Recatangle(width=" << width << ", height=" << height << "\n";        
    }

    const Color& get_color() const
    {
        return color;
    }

    void set_color(const Color& new_color)
    {
        color = new_color;
    }
};

//...

render(Rect{100, 200, Color(255, 0, 128)}); // render#2 called - more constrained function
```

Subsumacja działa tylko **dla konceptów**. Nie działa w sytuacji gdy ograniczenia są definiowane bez użycia konceptów.

```c++
template <typename T>
    requires std::is_convertible_v<T, int>
void print(T item)
{}

template <typename T>
    requires std::is_convertible_v<T, int> && (sizeof(T) >= 4)
void print(T item)
{}

print(4); // ERROR - call to 'print' is ambiguous
```

#### Przykłady subsumacji konceptów

* `std::random_access_range` subsumuje `std::bidirectional_range`
* oba subsumują `std::forward_range`
* wszystkie trzy subsumują `std::input_range`
* wszystkie subsumują `std::range`

* `std::sortable` subsumuje `std::permutable`
* oba subsumują `std::indirectly_swappable`

### Koncepty semantyczne

Koncepty mogą sprawdzać ograniczenia *składniowe* lub *semantyczne*

#### Ograniczenia składniowe - syntactic constraints

* Umożliwiają sprawdzenie na etapie kompilacji, czy spełnione są określone wymagania
  * Czy dana operacja jest wspierana przez typ `T`?
  * Czy dana operacja zwraca jako rezultat określony typ?

Koncept składniowy `std::invocable` definiuje możliwość wywołania funkcji `F` z argumentami `Args...`:

```c++ code-noblend
template< class F, class... Args >
concept invocable =
    requires(F&& f, Args&&... args) {
       std::invoke(std::forward<F>(f), std::forward<Args>(args)...);
    };
```

#### Ograniczenia semantyczne - semantic constraints

* Definiują dodatkowo wymagania, które mogą być tylko sprawdzone podczas wykonania programu
  * Czy dana operacja zwraca zawsze ten sam rezultat dla określonego parametru?

Koncept semantyczny `std::regular_invocable`

```c++ code-noblend
template< class F, class... Args >
concept regular_invocable = std::invocable<F, Args...>;
```

gwarantuje, że wywołanie funkcji nie zmieni stanu obiektu `F` oraz stanu argumentów `Args...`

#### Koncepty semantyczne - wykorzystanie w API

Koncepty semantyczne służą do lepszego dokumentowania API funkcji generycznych:

```c++ code-noblend
template <std::weakly_incrementable T>
void my_algorithm_1(T first, T last)  // single-pass algorithm
{}

template <std::incrementable T>
void my_algorithm_2(T first, T last) // multi-pass algorithm
{}
```

```c++
my_algorithm_1(std::istream_iterator<int>{std::cin}, std::istream_iterator<int>{});

my_algorithm_2(std::istream_iterator<int>{std::cin}, std::istream_iterator<int>{});
```

W powyższym przykładzie przekazanie iteratora strumienia wejścia do funkcji `my_algorithm_2` nie spełnia wymagań związanych z typem iteratora. Wymaganie to nie może być sprawdzone przez kompilator, ponieważ jest to wymaganie stricte semantyczne, związane z możliwością przejścia przez zakres przy pomocy iteratora wiele razy (iterator strumienia nie ma takiej własności).

Ograniczenia definiowane przy pomocy konceptów semantycznych nie powodują błędów kompilacji.