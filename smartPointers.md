# Smart Pointers
URL : [Link to Video](https://www.youtube.com/watch?v=xGDLkt-jBJ4&list=PL5qoVlA-tv09ykIIPHP9N6vgJaFPnYWCa&index=3)

## Index
* Unique ownership with std::unique_ptr 
* Shared ownership with std::shared_ptr 
* make_shared and make_unique 
* std::weak_ptr 
* std::enable_shared_from_this 

### Smart pointers look familiar, by design
* Initialization (and assignment) works as you’d expect
```C++
std::shared_ptr<String> p = std::make_shared<String>("Hello");
auto q = p;
p = nullptr;
```
* comparison and dereferencing also works like normal pointers
```C++
if (q != nullptr) 
{
    std::cout << q->Length() << *q << '\n';
}
```

### Unique Pointer
* Use std::unique_ptr for exclusive-ownership resource management for example an heap allocated object or a singleton object. 
* unique_ptr is moveable (move-only)
* There’s a specialization for array types. `std::unique_ptr<T[]>`
* You can supply an callable object (eg a functor) that to be called while destroying the object being managed `std::unique_ptr<T, Deleter>`
```C++
// unique_ptr is always a template of two parameters. If you provide no second parameter, it is defaulted to std::default_delete<T>.
template<class T, class Deleter = std::default_delete<T>>
class unique_ptr 
{
    // Even this part is customizable. If Deleter::pointer names a type, then this member will be of that type instead of T*.
    T *p_ = nullptr;
    Deleter d_;
    ~unique_ptr() 
    {
        if (p_) d_(p_);
    }
};

// a functor which will be used
template<class T>
struct default_delete 
{
    void operator()(T *p) const 
    {
        // ... custom clean here(closing file or socket) ...
        delete p;
    }
};
```

#### Rules of thumb for smart pointers
* Treat smart-pointer types just like raw pointer types.
    * Pass by value.
    * Return by value (of course).
    * Passing a pointer by reference is usually a code smell.
    * Same goes for smart pointers.

* A function taking a unique_ptr by value shows transfer of ownership.
    * Even shows exactly what responsibility is being transferred, because the responsibility is encoded in the deleter type.
    * Usually the responsibility is simply “to call delete.”
* Smart pointers are frequently implementation details and glue.
    * To bake unique_ptr or shared_ptr into your interface might be a code smell. Try to deal in business classes.

### std::shared_ptr<T>

Use std::shared_ptr for shared-ownership resource management. shared_ptr looks similar to unique_ptr on the surface... But it is vastly more complicated on the inside! shared_ptr expresses shared ownership. Specifically, reference-counting.
```C++
std::unique_ptr<int> uptr = std::make_unique<int>(42);
std::shared_ptr<int> sptr = std::make_shared<int>(42);
```

shared_ptr has two components 
* ptr to T Object 
* ptr to controlled object control block containing 
    * reference count
    * weak ref count
    * custom deleter?
    * ptr to controlled object

##### Why does the control block need a pointer to the controlled object, when the shared_ptr itself already holds a pointer?
We need to look at the class layout for this.
Objects of `class A : public B` whill have parts from both `A` and `B`, with a little offset in memory. 
```
Object of class A
{
    0xabc [...A...] // memory to A specific
    0xpqr [...B...] // memory to B specific
}
```
Lets assume this object is managed by a shared_ptr, and one of the shared_ptr is of type A and other is of type B both pointing to same object, then they will be pointing a bit offset and hence deleteing a ptr might not delete other one.

> Your shared_ptr instances are essentially participating in ref-counted ownership of the control block. The control block itself is sole arbiter of what it means to “delete the controlled object.”

Unique_ptr can be converted to shared_ptr, but vice-versa is not true
```C++
std::shared_ptr<Widget> sptr = std::make_unique<Widget>();
```

### Weak Pointers
Use std::weak_ptr for shared_ptr-like pointers that can dangle
* A weak pointer is just like a shared pointer but does not increase the ref-count of the shared_ptr control block but increments weak-ref counts
* when the object os destroyed (ref-count == 0), the control block remains, hence the weak_ptr knows the obj is being destroyed and it is dangling now.

#### You can’t dereference a weak_ptr
weak_ptr are not pointer (or smart pointers) they are proxy objects to get a shared ptr. If the weak_ptr is dangling we cant get shared_ptr.
```C++
; // increments ref-count
if (std::shared_ptr<T> sptr = wptr.lock()) 
{
    // ... use sptr ...
}
```

### What if a Widget is its own ticket?
What if you want the rule to be “If you hold a raw pointer, you’re entitled to receive a shared_ptr”?

Yes we can subclass from stl supplied base class
```C++
class Listener : public std::enable_shared_from_this<Listener> 
{
    void on_accept(std::error_code);
    void run() 
    {
        acceptor_.async_accept(socket_,
            // here we are extending the life of this pointer
            [self = shared_from_this()](std::error_code ec)  
            {
                self->on_accept(ec);
            }
        );
    }
};

...
// somewhere in code-base 
std::make_shared<Listener>(...)->run();
```
> https://stackoverflow.com/questions/712279/what-is-the-usefulness-of-enable-shared-from-this 

see second, third answer in this