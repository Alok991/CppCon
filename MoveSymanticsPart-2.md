# Move Symantics Part-2
URL : [Link to Video](https://www.youtube.com/watch?v=pIzaZbKUw2s&list=PLX-5eF57P5eEHJCMYVIOL9u9ZQtKas8Ut&index=4)

## Index
* Forwarding References
* Perfect Forwarding
* The Perils of Forwarding References Overloading with Forwarding References
* Move Semantics Pitfalls

## Forwarding References
Forwarding reference(also known as universal ref) only if in this form 
```C++
template< typename T >
void f( T&& x );       // Forwarding reference

auto&& var2 = var1;    // Forwarding reference
```


Forwarding references represent ...
* an lvalue reference if they are initialized by an lvalue; 
* an rvalue reference if they are initialized by an rvalue.

Rvalue references are forwarding references if they involve type deduction or appear in exactly the form T&& or auto&&.

> && also represents an r-value reference but that is different from forwarding reference.

```C++
template< typename T >
void foo( T&& ) 
{
    puts( ”foo(T&&)” ); 
}
int main() 
{
    foo( Widget{} ); // r-value ref ; prints foo(T&&)
    Widget w;
    foo(w);          // l-value ref ; also prints foo(T&&)
}
```

## Perfect Forwarding

Perfect forwarding problem: suppose we want to forward the arguments as it is to next funciton

```C++
namespace std 
{ 
    template<typename T, ???>
    unique_ptr<T> make_unique(???) 
    {
        return unique_ptr<T>(new T(???)); 
    }
} // namespace std
```

In above code we want to forward ??? type params to unique_ptr constructor, but this problem is impossible without forwarding ref.
Now we will see example of how we can tackle this without forwarding ref and what are their drawback.

### 1. Pass by copy
```C++
namespace std 
{
    template<typename T, typename Arg>
    unique_ptr<T> make_unique(Arg arg) 
    {
        return unique_ptr<T>(new T(arg)); 
    }
} // namespace std
std::make_unique<int>( 1 ); // Cheap extra copy 
std::make_unique<Widget>( w ); // Expensive extra copy
```

* this works pretty often
* might now work for types that are not copyable
* This creates an overhead of copy
Hence this is not perfect

### 2. Pass by ref

```C++
namespace std 
{
    template<typename T, typename Arg>
    unique_ptr<T> make_unique(Arg& arg) 
    {
        return unique_ptr<T>(new T(arg)); 
    }
} // namespace std

std::make_unique<int>( 1 ); // Compilation error, rvalue
```

* This works for all non-const ref and all const L-value ref
* But This wont work for r-value because **r-value does not bind to and L-value ref to non-const**

Hence wont work either.

### 3. Pass by const L-value Ref 

```C++
namespace std 
{
    template<typename T, typename Arg>
    unique_ptr<T> make_unique(Arg const& arg) 
    {
        return unique_ptr<T>(new T(arg)); 
    }
} // namespace std
struct Example { Example( int& ); };
int i{ 1 };
std::make_unique<Example>( i ); // Always adds const
```

* This works for both r-value and L-value ref.
* But it will always add const :(

Hence These wont work perfectly either

### 4. Pass by forwarding ref
Lets try our brand new ref and see if that works

```C++
namespace std 
{
    template<typename T, typename Arg>
    unique_ptr<T> make_unique(Arg&& arg) 
    {
        return unique_ptr<T>(new T(arg)); 
    }
} // namespace std
```
* This accepts r- value and L-value both like prev case
* does not add anything extra to type
* But once the parameter arg is inside the function(make_unique) scope it becomes an L-value again ( it has a name now :( ), Hence we miss the oppostunity to move.
and get a perfomance gain, and if we do a std::move on arg, we will move an L-value too which might be used in caller's scope 
* So only if we could use std::move but unlike std::move it should not move L-value to R-value ref

### std::forward to rescue :)

std::forward conditionally(unline std::move which cast unconditionally) casts its input into an rvalue reference 
* If the given value is an lvalue, cast to an lvalue reference
* If the given value is an rvalue, cast to an rvalue reference
* std::forward does not forward anything in reality it just cast into correct type
```C++
template< typename T >
T&& std::forward( std::remove_reference_t<T>& t ) noexcept 
{
    return static_cast<T&&>( t ); 
}
```

Now lets use this std::forward in out make_unique example

```C++
namespace std 
{
    template<typename T, typename Arg>
    unique_ptr<T> make_unique(Arg&& arg) 
    {
        return unique_ptr<T>(new T(std::forward<Arg>(arg))); 
    }
} // namespace std
```

* Here the value arg does not help us it will always be L-value, so we passed Arg as type info which the std::forward can use and give constructor of T a correct Type with value arg

```C++
// a more generalised correct form of make_unique
namespace std 
{
    template<typename T, typename... Args>
    unique_ptr<T> make_unique(Args&&... args)
    {
        return unique_ptr<T>(new T(std::forward<Args>(args)...));
    }
} // namespace std
```

Lets revisit out std::move once again
* std::move unconditionally casts its input into an rvalue reference
* std::move does not move anything just type cast it to R-value ref

```C++
template< typename T >
std::remove_reference_t<T>&& move( T&& t ) noexcept
{
    return static_cast<std::remove_reference_t<T>&&>( t ); 
}
```

So now the corrected defenition of std::move, it takes a forwarding reference, this is why it can take lvalue or rvalue anything , and outputs a rvalue

## The Perils of Forwarding References

Lets take few example code and understand forward better

```C++

struct Person 
{
    Person( const std::string& name );     // (1)
    template< typename T > Person( T&& );  // (2)
    struct Person {
        Person( const std::string& name );     // (1)
        template< typename T > Person( T&& );  // (2)
 };
int main() 
{
    // Try to guess which constructor will be called for each of below case
   Person p1( "Bjarne" );   // --> ( 1 )
   std::string name( "Herb" );
   Person p2( name );       // --> ( 2 )
   Person p3( p1 );         // --> ( 3 )
}

// ANSWER
// ( 1 )calls constructor (2); // argument type is char[7]
// ( 2 )calls constructor (2); // argument type is NOT const
// ( 3 )calls constructor (2), not copy ctor; // argument type is NOT const ; Note it does not call copy constructor
};
```
> Lets take few more example and understand whats happening here better :) (Dont worry)

### Overloading with Forwarding References
Lets make few functions and understand which one will be called

```C++
// Function with lvalue reference (1) 
void f( Widget& );

// Function with lvalue reference-to-const (2) 
void f( const Widget& );

// Function with rvalue reference (3) 
void f( Widget&& );

// Function with rvalue reference-to-const (4) 
void f( const Widget&& );

// Function template with forwarding reference (5)
template< typename T >
void f( T&& );

// Function template with rvalue reference-to-const (6) 
template< typename T >
void f( const T&& );

```
> NOTE : ( 6 ) is not forwading ref as it does not match with out forwading ref syntax(HINT : it has const)

Lets assume we have a function as below, which f will be called
```C++
void g() 
{
    Widget w{};
    f( w ); 
}
```

Answer : preference is 1, 5, 2 
1 because it is l-value ref , if it was not available then universal ref could act as l-value ref . if 5 is not available then 2 will be used as it has l-value ref but with a const hence least preferred. Rest all takes rvalue ref only


```C++
void g() 
{
    const Widget w{};
    f( w ); 
}
```

Answer : 2 and then 5
2 matches the argument type(const widget&) perfectly hence no doubt, and 5 can take l-value ref with T deduced to be const widget, Rest are either not const or takes r-value ref only


```C++
widget getWidget();
void g() 
{
    f( getWidget() ); 
}
```

Answer : 3,5,4,6,2

3 matches the argumetn type perfectly (r-value ref), then 5 can act as r-value but need template deduction hence 3 was prefered before it. then 4 is prefered followed by 6 (specialised function are prefered over template derived one) and lastly const l-value ref can also be used for r-value ref arg types.

```C++
const widget getWidget();
void g() 
{
    f( getWidget() ); 
}
```
Answer : 4,6,5,2

4 matches the arg type perfectly, then 6 is more specialised template for const r-value ref hence prefered over 5, then 5 is used and lastly 2 as it is const l-value ref.

> **Effective Modern C++, Item 26: Avoid overloading on universal references (Scott Meyers)**

Reason for above advice is that it overtakes most of the other functions.

## Move Semantics Pitfalls

### Example 1
```C++
class A 
{
    public:
    template< typename T >
    A( T&& t ) : b_( std::move( t ) )
    {}
    private:
    B b_;
};
```
Question : What is the problem with above code.
Answer : ctor of A takes an forward ref. Hence when we pass an l-value, it moves unconditionally (transfer content, take onwership) but it could be used in caller's scope. Hence we should only move if an r-value ref was passed hence we should use `std::forward` instead of `std::move`.

### Example 2
```C++
template< typename T >
class A 
{
    public:
    A( T&& t ) : b_( std::forward<T>( t ) )
    {}
    private:
    B b_;
};
```
Question: What is problem in above code?
Answer:  First ctor does not take forward ref it is r-value ref, So instead of `std::forward` we should used `std::move`, although forward will do the same but more.

### Example 3 

```C++
class A 
{
    public:
    template< typename T >
    A( T&& t )
    : b_( std::forward<T>( t ) )
    , c_( std::forward<T>( t ) )
    {}
    private:
    B b_;
    C c_;
};
```
Question: What is problem in above code?
Answer : We cannot move from same value twice. what you should instead is below
```C++
    template< typename T >
    A( T&& t )
    : b_(t)
    , c_( std::forward<T>( t ) )
    {}
```

### Example 4
```C++
template< typename... Args >
std::unique_ptr<Widget> create( Args&&... args )
{
    auto uptr( std::make_unique<Widget>(
    std::forward<Args>(args)... ) );
    return std::move( uptr );
}
```
Question: What is the problem here ?
Answer: It prevents RVO (return value optimisation), when we retunr a move RVO is turned off
instead use it like `return std::make_unique<Widget>( std::forward<Args>(args)... );`
Now it returns an r-value ref with RVO.

### Example 5
```C++
template< typename... Args >
std::unique_ptr<Widget>&& create( Args&&... args )
{
    return std::make_unique<Widget>(
    std::forward<Args>(args)... );
}
```
Question: What is the problem here ?
Answer : It return a ref to local object.

> **Core Guideline F.45: Don’t return a T&&**

### Example 6

```C++
template< typename T >
void foo( T&& )
{
    if constexpr( std::is_integral_v<T> )
    {
    // Deal with integral types
    }
    else
    {
    // Deal with non-integral types
    }
}
```
Question: What is the problem here ?
Answer : Here T is not a concrete type, deduction will make it widget& or int& hence `std::is_integral_v` will always be false. Use below code by removing the ref.
```C++
template< typename T >
void foo( T&& )
{
    using NoRef = std::remove_reference_t<T>;
    if constexpr( std::is_integral_v<NoRef> )
    {
    // Deal with integral types
    }
    else
    {
    // Deal with non-integral types
    }
}
```
**ALL DONE** :)