# Coroutines in C++20

## Subroutine vs. Coroutine

Coroutine is a generalization of a subroutine:

A **subroutine**:

* can be invoked by its caller
* can return control back to its caller

A **coroutine** has both properties, but also:

* can suspend its execution and return control to its caller
* can resume execution after being suspended

In C++ function can be either a subroutine or a coroutine:

|         |     Subroutine      |           Coroutine            |
| ------- | ------------------- | ------------------------------ |
| Invoke  | Function call `f()` | Function call `f()`            |
| Return  | `return statement;` | `co_return statement;`         |
| Suspend |                     | `co_await expression;`         |
| Resume  |                     | `coroutine_handle<>::resume()` |

Since C++20 a function is a coroutine it its body contains:

* `co_return` statement

  ```cpp
  lazy<int> f() 
  {
      co_return 7;
  }
  ```

* or `co_await` expression

  ```cpp
  cppcoro::task<> tcp_echo_server(socket_t& socket) 
  {
      using buffer_t = std::array<uint8_t, 1024>;
      buffer_t buffer{};
      for (;;) 
      {
          size_t bytes_read = co_await socket.async_read_some(buffer.data());
          co_await async_write(socket, buffer.data(), bytes_read));
      }
  }
  ```

* or `co_yield` expression

  ```cpp
  cppcoro::generator<int> iota(int n = 0) 
  {
    while(true)
        co_yield n++;
  }
  ```

Compiler transforms a coroutine function into a state machine.

Coroutine behaves as if its body were replaced by:

````{panels}
Coroutine
^^^
```cpp
ReturnType coroutine_f(Params)
{
    function_body();
}
```
---
Unrolled code
^^^
```cpp
ReturnType coroutine_f(Params)
{
    using promise_type = std::coroutine_traits<ReturnType, Params>::promise_type

    promise_type promise(promise_constructor_arguments);

    try
    {
        co_await promise.initial_suspend(); // initial suspend point
        function_body();  
    }
    catch(...)
    {
        if (!initial_await_resume_called)
            throw;
        promise.unhandled_exception();
    }

final-suspend: // final suspend point
    co_await promise.final_suspend();
}
```
````

where:

* the `await-expression` containing the call to `initial_suspend` is the *initial suspend point*
* the `await-expression` containing the call to `final_suspend` is the *final suspend point*
* `initial-await-resume-called` is initially `false` and is set to `true` immediately before the evaluation of the await-resume expression of the initial suspend point
* `promise-type` denotes the *promise type*

## Mechanics of coroutines

### Function call

When a function is being called, the compiler constructs a *stack frame*. The stack frame includes space for:

* arguments of the function call
* local variables
* return value
* temporary storage for registers

### Coroutine call

Invoking a coroutine forces the compiler to construct a *coroutine stack frame* that contains space for:

* formal parameters
* local variables
* selected temporaries
* execution state for when the coroutine is suspended (registers, instruction pointers, etc.)
* the *promise* that is used to return a value or values to the caller

In general *coroutine stack* must be dynamically allocated:

* when the coroutine is suspended it loses its local stack - dynamic allocation allows to preserve the local state
* operator new is used by default
  - it can be overloaded for specific coroutines, to allow customization
* dynamic allocation can sometimes be elided by the compiler

```{note}
Creation of the coroutine frame occurs before the coroutine starts running
```

The compiler passes a handle to this coroutine frame to the caller of the coroutine.

### Destruction of a coroutine state

The coroutine state is destroyed when control flows off the end of the coroutine or the `destroy()` member function of a coroutine handle that refers to the coroutine is invoked.

```{note}
If `destroy()` is called for a coroutine that is not suspended, the program has undefined behavior.
```

## Handle of the coroutine

A *coroutine handle* is an object that wraps up a pointer to a coroutine frame. A coroutine frame is allocated on a heap.

We can distinguish two general cases for a coroutine handle.

* The first is a coroutine handle to a coroutine that returns `void`:

```cpp
template <typename _PromiseT = void>
struct coroutine_handle;

template <>
struct coroutine_handle<void> // no promise access
{ 
    coroutine_handle() noexcept; // it allows to construct empty coroutine_handle
    coroutine_handle(nullptr_t) noexcept;
    coroutine_handle& operator=(nullptr_t) noexcept;
    explicit operator bool() const noexcept; // conversion - if handle is empty
        
    static coroutine_handle from_address(void* _Addr) noexcept; // conversion function - allows to convert a pointer to handle
    void* address() const noexcept; // returns pointer to coroutine handle

    void operator()() const; // it allows to resume the execution
    void resume() const; // the same as operator()() - resumes the execution

    void destroy(); // allows to manually destroy the coroutine
        
    bool done() const; // tests whether the coroutine has completed execution
private:
    void* ptr_; // a pointer to a coroutine stack frame
};
```

* The second is a couroutine handle to a coroutine that returns some value. For this this handle we can provide a specialization of a `coroutine_handle` template:

```cpp
template <typename Promise>
struct coroutine_handle : coroutine_handle<void>
{
    Promise& promise() const noexcept; // gets the promise back

    static coroutine_handle from_promise(Promise&) noexcept; // returns a handle from the promise object 
};
```

```{note}
Because a promise object is constructed somewhere in a coroutine frame, knowing the promise object, we can obtain the proper `coroutine_handle`.
```

### Identifying a specific coroutine activation frame

You can obtain a `coroutine_handle` for a coroutine in two ways:

* It is passed to the `await_suspend()` method during a `co_await` expression.
* If you have a reference to the coroutine’s promise object, you can reconstruct its coroutine_handle using `coroutine_handle<Promise>::from_promise()`

The `coroutine_handle` of the awaiting coroutine will be passed into the `await_suspend()` method of the awaiter after the coroutine has suspended at the `<suspend-point>` of a `co_await` expression. You can think of this `coroutine_handle` as representing the continuation of the coroutine in a [continuation-passing](https://en.wikipedia.org/wiki/Continuation-passing_style) style call.

Note that the coroutine_handle is NOT and RAII object. You must manually call `.destroy()` to destroy the coroutine frame and free its resources. Think of it as the equivalent of a void* used to manage memory. This is for performance reasons: making it an RAII object would add additional overhead to coroutine, such as the need for reference counting.

## Promise object

* Promise object is an object that allows to pass values from a coroutine to the caller
* Allows compile time checks using the `std::coroutine_traits<T...>` trait class
* Is allocated on a coroutine frame that is allocated on a heap using operators `new`/`delete` by default
* The interface of a promise object contains:
  - constructor/destructor (RAII)
  - `get_return_object()` - this function is used to initialize the result of a call to a coroutine
  - `get_return_object_on_allocation_failure()`
  - functions that allow to pass values from the coroutine to a caller
    - `co_return`: `return_value()`, `return_void()`
    - `co_yield`: `yield_value()`

## co_await

The unary operator `co_await` suspends a coroutine and returns control to the caller.

`co_await` expression is unrolled by a compiler to the following code:

````{panels}
Expression
^^^
```cpp
auto r = co_await expr;
```
---
Unrolled code
^^^
```cpp
auto r = {
    auto&& awaiter = expr;

    if (!awaiter.await_ready())
    {
        __builtin_coro_save(); // frame->suspend_index = n;
        awaiter.await_suspend(<coroutine_handle>);
        __builtin_coro_suspend(); // jmp epilog
    }

resume_label_n:
    awaiter.await_resume();
};
```
````

The expression that follows the `co_await` operator is converted to an *awaitable object*.

If the coroutine was suspended in the `co_await` expression, and is later resumed, the resume point is immediately before the call to `awaiter.await_resume()`

### Awaitable object

In order to await for something awaitable type must obey the `awaitable` concept:

```cpp
template <typename T>
concept awaitable = requires(T obj) {
    { obj.await_ready(); } -> std::convertible_to<bool>;
    { obj.await_suspend(coroutine_handle<>); };
    { obj.await_resume(); }
};
```

When the awaiter object is created, then the `awaiter.await_ready()` is called. This is a short-cut to avoid the cost of suspension if the result is ready or can be completed synchronously.

 If the returned result is `false`, then the coroutine is suspended and `awaiter.await_suspend(coroutine_handle<P>)` is called. Inside this function the suspended coroutine state is observable via coroutine handle. The responsibility of this function it to schedule the coroutine to resume on some executor (or to be destroyed).
* if `await_suspend()` returns `void`, control is immediately returned to the caller/resumer of the current coroutine (this coroutine remains suspended), otherwise
* if `await_suspend()` returns `bool`,
  * the value `true` returns control to the caller/resumer of the current coroutine
  * the value `false` resumes the current coroutine
* if `await_suspend()` returns a coroutine handle for some other coroutine, that handle is resumed (by a call to `handle.resume()`) (this may chain to eventually cause the current coroutine to resume)
* if `await_suspend()` throws an exception, the exception is caught, the coroutine is resumed, and the exception is immediately re-thrown

Finally, `awaiter.await_resume()` is called, and its result is the result of the whole `co_await expr` expression.

```{note}
Because the coroutine is fully suspended before entering `awaiter.await_suspend()`, that function is free to transfer the coroutine handle across threads, with no additional synchronization. 
For example, it can put it inside a callback, scheduled to run on a `threadpool` when async I/O operation completes. In that case, since the current coroutine may have been resumed and thus executed the awaiter object's destructor, all concurrently as `await_suspend()` continues its execution on the current thread, `await_suspend()` should treat `*this` as destroyed and not access it after the handle was published to other threads.
```

#### The simplest awaitables

##### suspend_always

```cpp
struct suspend_always
{
    bool await_ready() noexcept
    {
        return false; // I don't have a value yet
    }

    void await_suspend(coroutine_handle<>) noexcept { }

    void await_resume() noexcept { }
};
```

##### suspend_never

```cpp
struct suspend_never
{
    bool await_ready() noexcept
    {
        return true;
    }

    void await_suspend(coroutine_handle<>) noexcept { }

    void await_resume() noexcept { }
};
```

(co-return)=
## co_return

Returning a value from the coroutine can be done using `co_return` statement.

When a coroutine reaches the `co_return` statement, it performs the following:

* calls `promise.return_void()` for 
  - `co_return;`
  - `co_return expr;` where `expr` has type `void`
  - falling off the end of a void-returning coroutine. If the *promise type* has no `Promise::return_void()` the behavior of the program is undefined

* or calls `promise.return_value(expr)` for `co_return expr;` where `expr` has non-void type
* destroys all variables with automatic storage duration in reverse order they were created
* calls `promise.final_suspend()` and `co_await`s the result

## Yielding a value - co_yield

A *yield expression* has a following form:

````{panels}
Expression
^^^
```cpp
co_yield expression;
```
---
Unrolled code
^^^
```cpp
co_await p.yield_value(expression);
```
````

## Customization points for a coroutine

The interface of **promise** object specifies methods for customizing the behavior of the coroutine. The library writer is able to customize:

* what happens when the coroutine is called
* what happens when the coroutine returns (either by normal return or via unhandled exception)
* behavior of any `co_await` or `co_yield` expression within the coroutine body

### Allocating a coroutine frame

The compiler needs to allocate memory for a coroutine frame. To achieve this it generates a call to `operator new`.

If the `promise_type`  defines a custom `operator new` then that is called, otherwise global `operator new` is called.

The size passed to operator new is not `sizeof(Promise)` but is rather the size of the entire coroutine frame and is determined automatically by the compiler based on the number and sizes of parameters, size of the promise object, number and sizes of local variables and other compiler-specific storage needed for management of coroutine state.

The compiler is free to elide the call to operator new as an optimization if:

* it is able to determine that the lifetime of the coroutine frame is strictly nested within the lifetime of the caller; and
* the compiler can see the size of coroutine frame required at the call-site.

```cpp
struct MyPromise
{
  void* operator new(std::size_t size)
  {
    void* raw_mem = my_custom_allocate(size);
    if (!raw_mem) 
        throw std::bad_alloc{};
    return raw_mem;
  }

  void operator delete(void* raw_mem, std::size_t size)
  {
    my_custom_free(raw_mem, size);
  }
};
```

### Obtaining the return object

The first thing a coroutine does with a promise object is obtain the `return-object` by calling `promise.get_return_object()`.

The `return-object` is the value that is returned to the caller of the coroutine function then the coroutine first suspends or after it runs to completion and execution returns to the caller.

### The initial-suspend point

The next thing the coroutine executes once the coroutine frame has been initialized and the return object has been obtained is execute the statement `co_await promise.initial_suspend();`.

This allows the author of the `promise_type` to control whether the coroutine should suspend before executing the coroutine body that appears in the source code or start executing the coroutine body immediately.

If the coroutine suspends at the initial suspend point then it can be later resumed or destroyed at a time of your choosing by calling `resume()` or `destroy()` on the coroutine’s coroutine_handle.

The result of the `co_await promise.initial_suspend()` expression is discarded so implementations should generally return void from the `await_resume()` method of the awaiter.

For many types of coroutines, the `initial_suspend()` method either returns:

* `std::suspend_always` (if the operation is lazily started) or
* `std::suspend_never` (if the operation is eagerly started)

### Returning to the caller

When the coroutine function reaches its first `<return-to-caller-or-resumer>` point (or if no such point is reached then when execution of the coroutine runs to completion) then the return-object returned from the `get_return_object()` call is returned to the caller of the coroutine.

The type of the `return-object` doesn’t need to be the same type as the `ReturnType` of the coroutine function. An implicit conversion from the `return-object` to the `ReturnType` of the coroutine is performed if necessary.

### Returning from the coroutine using `co_return`

[See co_return behavior](co-return)

### Handling exceptions for the coroutine body

If an exception propagates out of the coroutine body then the exception is caught and the `promise.unhandled_exception()` method is called inside the `catch` block.

Implementations of this method typically call `std::current_exception()` to capture a copy of the exception to store it away to be later re-thrown in a different context.

### The final-suspend point

Once execution exits the user-defined part of the coroutine body and the result has been captured via a call to `return_void()`, `return_value()` or `unhandled_exception()` and any local variables have been destructed, the coroutine has an opportunity to execute some additional logic before execution is returned back to the caller/resumer.

The coroutine executes the `co_await promise.final_suspend();` statement.

This allows the coroutine to execute some logic, such as publishing a result, signalling completion or resuming a continuation. It also allows the coroutine to optionally suspend immediately before execution of the coroutine runs to completion and the coroutine frame is destroyed.

```{note}
It is undefined behavior to `resume()` a coroutine that is suspended at the `final_suspend` point. The only thing you can do with a coroutine suspended here is `destroy()` it.
```

```{note}
It is recommended that you structure your coroutines so that they do suspend at `final_suspend` where possible. 

This is because this forces you to call `.destroy()` on the coroutine from outside of the coroutine (typically from some RAII object destructor) and this makes it much easier for the compiler to determine when the scope of the lifetime of the coroutine-frame is nested inside the caller. This in turn makes it much more likely that the compiler can elide the memory allocation of the coroutine frame.
```

### Customizing the behavior of co_await

The promise type can optionally customize the behavior of every `co_await` expression that appears in the body of the coroutine.

By simply defining a method named `await_transform()` on the promise type, the compiler will then transform every `co_await <expr>` appearing in the body of the coroutine into `co_await promise.await_transform(<expr>)`.

This has a number of important and powerful uses:
* It lets you enable awaiting types that would not normally be awaitable.
* It lets you disallow awaiting on certain types by declaring `await_transform` overloads as ` = delete`.

  ```cpp
  template<typename T>
  class generator_promise
  {
    ...
  
    // Disable any use of co_await within this type of coroutine.
    template<typename U>
    std::suspend_never await_transform(U&&) = delete;
  
  };
  ```

* It lets you adapt and change the behavior of normally awaitable values

### Customizing the behavior of co_yield

The final thing you can customize through the promise type is the behavior of the `co_yield` keyword.

If the `co_yield` keyword appears in a coroutine then the compiler translates the expression `co_yield <expr>` into the expression `co_await promise.yield_value(<expr>)`. 

The promise type can therefore customize the behavior of the `co_yield` keyword by defining one or more `yield_value()` methods on the promise object.
