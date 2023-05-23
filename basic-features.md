# Nowości w składni języka

## Operatory równości i porównań

### Operatory == i !=

Aby sprawdzać równość lub nierówność obiektów, od C++20 wystarczy zaimplementować w klasie tylko `operator ==`.
Wynika to z faktu, że sprawdzając nierówność, w sytuacji gdy klasa nie ma zaimplementowanego operatora `!=`, kompilator może **przepisać wyrażenie** `a != b`:

* do wyrażenia `!(a == b)`
* lub do wyrażenia `!(b == a)`

```c++
struct Value
{
    int value;

    bool operator==(const Value& other) const
    {
        return value == other.value;
    }
};

Value v1{20};
Value v2{20};
Value v3{42};

assert(v1 == v2); // no re-writing
assert(v1 != v3); // rewriting to !(v1 == v3)
```