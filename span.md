# std::span

* Instancje `std::span<>` odwołują się do zewnętrznej sekwencji elementów
* Elementy muszą być umieszczone w sposób ciągły w pamięci
* Dostęp do elementów jest *read/write*
* Rozmiar sekwencji może być określony przez stałą lub ustalony jako otwarty i doprecyzowany później
* Instancje `std::span<>` są lekkimi obiektami - kopiowanie jest tanie
* `std::span<>` posiada semantykę referencyjną, co może stwarzać problemy
  * użycie jest równie niebezpieczne co *raw-pointerów*
  * łamie *const correctness*

## Stały zakres (fixed extent)

* Rozmiar zakresu jest podany w deklaracji typu
* Konstruktor wymaga podania iteratora lub wskaźnika do początku zakresu i rozmiaru
  * rozmiar przekazany w konstruktorze **musi się równać** rozmiarowi przekazanemu jako parameter typu
* Domyślny konstruktor nie jest dostępny (chyba, że rozmiar wynosi `0`)
* Funkcja składowa `size()` zawsze zwraca ten sam rozmiar

```c++
std::vector<std::string> words = {"one", "two", "three", "four", "five"};

std::span<std::string, 5> all_words{words}; // OK - refers to all elements

std::span<const std::string, 3> first3{words.begin(), 3}; // span refers to first 3 elements
static_assert(first3.size() == 3);

std::span<std::string, 3> last3{words.end() - 3, 3};
last3[2] = "FIVE"

std::span<std::string, 3> wrong1{words.begin(), 5}; // runtime ERROR (UB)
std::span<std::string, 5> wrong2{words.begin(), 3}; // runtime ERROR (UB)
std::span<std::string, 5> wrong3{words, 5}; // compile-time ERROR
```

## Zakres dynamiczny (dynamic extent)

* `std::span` może zostać użyty bez podawania rozmiaru w trakcie deklaracji typu
  * tworzony jest wtedy obiekt, dla którego zakres jest określany dynamicznie (w *run-timie*)

```c++
std::vector vec = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

std::span<int> all_numbers1{vec}; // std::span<int, dynamic_extent>

std::span all_numbers2{vec};      // CTAD -> std::span<int, dynamic_extent>

std::span<int> sp3{vec.begin() + 2, 3}; // span with dynamic extent

std::span last3{vec.end() - 3, 3}; // with CTAD
```

* Rzeczywista definicja szablonu `span` wygląda tak

```c++
namespace std 
{
    template<typename ElementType, size_t Extent = dynamic_extent>
    class span 
    {
       //...
    };
}

std::span <const std::string> sp; // std::span<const std::string, dynamic_extent>
sp = vec; // assigning vec to sp (all items)
print_span(sp, "sp");
```

## std::span - wydajność

* Wydajność 
  * nie wymagają dynamicznej alokacji pamięci
  * dostęp do elementów jest realizowany za pomocą wskaźnika (wymagana ciągłość elementów w pamięci)
  * jedyny narzut jest związany z utworzeniem obiektu

* Spans vs. Subranges
  * podzakresy (`std::ranges::subrange`) nie wymagają ciągłego ułożenia elementów w pamięci
  * dostęp do elementów podzakresu jest realizowany za pomocą iteratora -> niższa wydajność

## Inicjalizacja - obiekty tymczasowe

* **Nigdy** nie inicjuj instancji `std::span` obiektem tymczasowym -> **UB**
  * span odwołuje się wtedy do zniszczonego obiektu (*dangling pointer*)

```c++
std::span<int, 3> sp3 = std::array{1, 2, 3}; // UB
std::span<int, 5> sp5 = load_data() // UB
```

## const correctness

* Deklaracja obiektu `std::span` jako stałego nie oznacza brak możliwości modyfikacji jego elementów
  * Obiekty `std::span` charakteryzyją się płytką stałością (*shallow constness*)

```c++
std::array data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

const std::span<int, 3> sp3{data.begin(), 3};
sp3[0] = 42; // OK - span is const, but an array under is not

assert(data[0] == 42);
```

* Jeśli chcemy zdefiniować span z opcją *read-only*, to musimy dodać `const` do typu elementu

```c++
std::array data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

std::span<const int, 3> sp3{data.begin(), 3};
sp3[0] = 42; // Error - span is read-only
```

## Przekazywanie obiektów span do funkcji

* `std::span` przekazujemy przez wartość

```c++
template <typename T, size_t N>
void print_span(std::span<T, N> sp, std::string_view info)
{
    std::cout << info << ": ";
    for (const auto& item : sp)
        std::cout << item << " ";
    std::cout << "\n";
}
```

## std::span - API

* Typy zagnieżdżone

|        Typ         |             Definicja             |
| ------------------ | --------------------------------- |
| `element_type`     | `T`                               |
| `value_type`       | `std::remove_cv_t<T>`             |
| `size_type`        | `std::size_t`                     |
| `difference_type`  | `std::ptrdiff_t`                  |
| `pointer`          | `T*`                              |
| `const_pointer`    | `const T*`                        |
| `reference`        | `T&`                              |
| `const_reference`  | `const T&`                        |
| `iterator`         | implementation-defined            |
| `reverse_iterator` | `std::reverse_iterator<iterator>` |


* Operacje

|           Typ            |                                         Definicja                                          |
| ------------------------ | ------------------------------------------------------------------------------------------ |
| `empty()`                | zwraca `true` jeśli zakres jest pusty                                                      |
| `size()`                 | zwraca ilość elementów                                                                     |
| `size_bytes()`           | zwraca ilość pamięci zajętej przez elementy                                                |
| `[]`                     | dostęp za pomocą indeksu                                                                   |
| `front(), back()`        | dostęp do pierwszego/ostatniego elementu                                                   |
| `begin(), end()`         | zwraca iteratory                                                                           |
| `rbegin(), rend()`       | zwraca iteratory wsteczne                                                                  |
| `subspan(offset, count)` | zwraca `std::span`, który jest widokiem na `count` elementów zaczynających się od `offset` |
| `first()`                | zwraca `std::span`, który jest widokiem na `N` pierwszych elementów                        |
| `last()`                 | zwraca `std::span`, który jest widokiem na `N` ostatnich elementów                         |
| `data()`                 | zwraca wskaźnik do elementów                                                               |
| `as_bytes()`             | zwraca widok reprezentujący pamięć -`std::span<const std::byte>`                           |
| `as_writable_bytes()`    | zwraca widok reprezentujący pamięć -`std::span<std::byte>`                                 |