# Non Type Template Parameters

Od C++20 możemy w szablonach NTTP (*Non Type Template Parameters*) używać:

* wartości typów zmiennoprzecinkowych (`float`, `double`)
* struktur danych w publicznymi składowymi
* obiektów domknięć (lambd)

## NTTP - floating point values

* stała wartość (constexpr) `double` lub `float` może być parametrem szablonu
* argumenty szablonów są porównywane bitowo (`operator==` nie jest używany)

``` c++
template <auto Factor>
auto multiply_by(auto x)
{
    return Factor * x;
}

//...
std::floating_point auto result_double = multiply_by<2.5>(10);
std::floating_point auto result_float = multiply_by<2.5f>(10);
```

``` c++
template <double Value> MyType {};
static_assert(std::is_same_v(
    MyType<0.1 + 0.3 + 0.00001>, 
    MyType<0.3 + 0.1 + 0.00001>)) // might be true or false
```

## NTTP - struktury

* Typy strukturalne (*structural types*) mogą być NTTP
  * wszystkie niestatyczne składowe są publiczne (i nie są `mutable`)
  * wszystkie klasy bazowe (jeśli są) są publiczne
  * Typ jest typem literalnym (jest agregatem lub ma `constexpr` konstruktor  i nie ma destruktora oraz konstruktora przenoszącego/kopiującego)

``` c++
struct Tax
{
    double value;

    constexpr Tax(double v)
        : value{v}
    {
        assert(v >= 0 && v < 1);
    }
};

template <Tax Vat>
double calculate_gross_price(double price)
{
    return price + price * Vat.value;
}
```

``` c++
constexpr Tax vat_pl{0.23};
constexpr Tax vat_ger{0.19};

double price_netto = 100.0;

assert(calculate_gross_price<vat_pl>(price_netto) == 123.0);
assert(calculate_gross_price<vat_ger>(price_netto) == 119.0);
```

+++

## NTTP - łańcuchy znaków

* Jeśli opakujemy łańcuch znaków w strukturę, to możemy użyć go jako NTTP
* Przykład wrapper'a:

``` c++
template <size_t N>
struct Str {
    char value[N];

    constexpr Str(const char (&str)[N]) {
        std::copy(str, str + N, value);
    }

    friend std::ostream& operator<<(std::ostream& out, const Str& str) {
        out << str.value;

        return out;
    }
};

template <Str StrLiteral>
constexpr auto operator""_str() {
    return StrLiteral;
}
```

* Użycie łańcucha znaków w szablonie

``` c++
template <Str Prefix>
class Logger {
public:
    void log(const std::string& msg)
    {
        std::cout << Prefix << msg << "\n";
    }
};


Logger<">: "> logger1;
logger1.log("Start...");
logger1.log("End...");

constexpr auto log_prefix = "-> "_str;
Logger<log_prefix> logger2;
logger2.log("Start");
logger2.log("Stop");
```

## NTTP - lambdy

* Jako NTTP można użyć też lambd:

``` c++
template <std::invocable auto GetVat>
double calculate_gross_price(double price)
{
    return price + price * GetVat();
}

auto get_vat_pl = [] { return 0.23; };
auto get_vat_ger = [] { return 0.19; };
```

``` c++
double price_netto = 100.0;

assert(calculate_gross_price<get_vat_pl>(price_netto) == 123.0);
assert(calculate_gross_price<get_vat_ger>(price_netto) == 119.0);
```