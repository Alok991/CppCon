# Type-Deduction
URL : [Link to Video](https://www.youtube.com/watch?v=wQxj20X-tIU)

### Type Deduction works on many topics
* TEMPLATE type (T&&, T&/T*, T)
* C++11 lambda capture
* auto Object
* C++14 lambda init capture
* decltype
* decltype(auto)
* auto return type
* C++14 lambda auto param
#### (auto-related) Template Type Deduction
```C++
template<typename T>
void f(ParamType param);
f(expr);
```
When we deduce type on above we have two kinds of type expr namely
 - T
    -- The deduced type.
 - ParamType
    -- Often different from T (e.g, const T&).(expr type)


Three general cases:
* ParamType is a reference or pointer, but not a universal reference.
* ParamType is a universal reference.
* ParamType is neither reference nor pointer.

#### Reference/Pointer Parameters (non-URef) 
> URef means universal Reference
* If expr’s type is a reference, ignore that
* Pattern-match expr’s type against ParamType to determine T.
```C++
int x = 22; // int
const int cx = x; // copy of int
const int& rx = x; // ref to const view of int
```

Lets deduce some types
```C++
template<typename T>
void f(T& param); // param is a reference

 f(x); // T ≡ int, param's type ≡ int&
 f(cx); // T ≡ const int, param's type ≡ const int&
 f(rx); // T ≡ const int, param's type ≡ const int&
```

```C++
template<typename T>
void f(const T& param);
f(x); // T ≡ int, param's type ≡ const int&
f(cx); // T ≡ int, param's type ≡ const int&
f(rx); // T ≡ int, param's type ≡ const int&
```
Behavior with pointers essentially the same:
```C++
template<typename T>
void f(T* param); // param now a pointer int x = 22; // int

const int *pcx = &x; // ptr to const view of int

f(&x); // T ≡ int, param's type ≡ int*
f(pcx); // T ≡ const int, // param's type ≡ const int*
```

Rules for auto behaves in the same way too
```C++
auto& v1 = x; // v1's type ≡ int& (auto ≡ int)
auto& v2 = cx; // v2's type ≡ const int& , (auto ≡ const int)
auto& v3 = rx; // v3's type ≡ const int& , (auto ≡ const int)
const auto& v4 = x; // v4's type ≡ const int& (auto ≡ int)
const auto& v5 = cx; // v5's type ≡ const int& , (auto ≡ int)
const auto& v6 = rx; // v6's type ≡ const int& , (auto ≡ int)
```

#### Universal References
```C++
template<typename T>
void f(T&& param);
```
If you initialize this reference with l-value then deduced type is l-value reference, similary r-value ref for r-value.
Treated like “normal” reference parameters, except:
 - If expr is lvalue with deduced type E, T deduced as E&. 
 - Reference-collapsing yields type E& for param.
 -  Reference-collapsing Rules
 - - && && -> &&
   - && &  -> &
   - &  && -> &
   - &  &  -> &
```
f(x); // x is lvalue ⇒ T ≡ int&, param's type ≡ int& (only case where a ref type is duduced for Template type T)
f(cx); // cx is lvalue ⇒ T ≡ const int&, , param's type ≡ const int&
f(rx); // rx is lvalue ⇒ T ≡ const int&, , param's type ≡ const int&
f(22); // x is rvalue ⇒ no special handling; , T ≡ int, param’s type is int&&
```

In first case f(x) x is an l-value hence it acts like f(T&& int&) and by refernce collapsing && & gives & (an l-value ref) so param type is int&
similar for f(cx) and f(rx), In case of f(22) 22 is an r-value and normal rules apply

#### By-Value Parameters
* As before, if expr’s type is a reference, ignore that.
* If expr is const or volatile, ignore that.
```C++
template<typename T>
void f(T param); // param passed by value

// we are creating a new obj here hence const, ref etc does not matter
f(x); // T ≡ int, param's type ≡ int
f(cx); // T ≡ int, param's type ≡ int
f(rx); // T ≡ int, param's type ≡ int
```

auto begaves in same way as Templates
```
auto v1 = x; // v1's type ≡ int (auto ≡ int)
auto v2 = cx; // v2's type ≡ int (auto ≡ int)
auto v3 = rx; // v3's type ≡ int (auto ≡ int)
auto v4 = rx; // v4's type ≡ int
auto& v5 = rx; // v5's type ≡ const int&
auto&& v6 = rx; // v6's type ≡ const int& (rx is lvalue)
```

#### const exprs vs. exprs Containing const
Constness of the variable itself is ignored when pass by value is used
```C++
void someFunc(const int * const param1, // const ptr to const
const int * param2, // ptr to const
int * param3) // ptr to non-const
{
    auto p1 = param1; // p1's type ≡ const int* , (param1's constness ignored)
    auto p2 = param2; // p2's type ≡ const int* (here param2 itself is not const hence this const is not ignored)
    auto p3 = param3; // p3's type ≡ int*
}
```

#### Special Cases
- When initializing a reference, array/function type deduced. 
- Otherwise decays to a pointer before type deduction.
```C++
int arr[10];
auto cArr = arr; // cArr's type int*
auto& rArr = arr // rArr's type arr[10]
```

### Auto but with braced initializers 

Braced initializers have no type
Hence 
- Template type deduction fails.
- auto deduces std::initializer_list.

```C++
This code is before C++17
template<typename T>
void f(T param);

f( { 1, 2, 3 } );    // error! type deduction fails
auto x1 { 1, 2, 3 }; // x's type ≡ std::initializer_list<int>
auto x2 = { 1,2,3 }; // x's type ≡ std::initializer_list<int>
```

For C++ 17
```C++
auto x1   { 1, 2, 3 };   // error! direct init w/>1 element
auto x2 = { 1, 2, 3 }; // std::initializer_list<int>
auto x3   { 17 }; 	   // direct init with 1 element, x's type ≡ int
auto x4 = { 17 };      // as in C++14, x's type ≡ std::initializer_list<int>
```

### Lambda Capture Type Deduction
##### Three kinds of capture
- By reference: uses template type deduction (for reference params).
- C++14’s init capture: uses auto type deduction.
- By value: uses template type deduction, except cv-qualifiers are retained

##### By reference(or copy): uses template type deduction (for reference params).
```C++

// Our Lambda
const int cx = 0;
auto mylambda = [cx] { ... };  
// here we are giving cx by copy but still preserve const
// This is only place where const is preserved when pass by copy

// This is compiler's internal representation (compiler generated)
class mylambda_UpToTheCompiler 
{
	private:
	const int cx;   
...
};
```


##### C++14’s init capture: uses auto type deducti
* when using init capture (auto mylambda = [cx=cx] { ... };  )
* Here we are making a new copy and constness is removed
* const retention normally masked by default constness of operator(): See below code
```
{
	int cx = 0; // now non-const
	auto lam = [cx] { cx = 10; }; // error!
...
}
class UpToTheCompiler 
{
    public:
    void operator()() const { cx = 10; } // cause of error
    private:
	int cx;
};
```

### Observing Deduced Types
So the rules for deduction of types are complex, we have a simple solution
#### During compilation
```
template<typename T>
class TD; 	// TD == "Type Displayer"

template<typename T> 
void f(T& param) // this is the function whose types we want to deduce 
{
    TD<T> tType; // cause T to be shown
    TD<decltype(param)> paramType; // ditto for param's type
}
```

#### At Run-Time
At run time things are bit complicated
> Avoid std::type_info::name.
> Language rules require incorrect results in some cases!
* Boost.TypeIndex::type_id_cvr provides accurate information

### decltype Type Deduction
Unlike auto never strips off const/ volatile/reference
```
int x = 10; 		// decltype(x) ≡ int
const auto& rx = x; // decltype(rx) ≡ const int&
```

decltype works as expected for simple Types (int x etc) and gives int but for more complex 
statement it behave in following manner
* decltype(lvalue expr of type T) ≡ T&.  // It slaps an l-value ref on it
 ```
 Example decltype(arr[0]) => int&
```
* decltype(statement) -> statement rules kick in 
```
Example decltype( (x) );  //int&
```

### Function Return Type Deduction (DONT USE THIS)
In C++11:
* Limited: single-statement lambdas only.

In C++14:
* Extensive: all lambdas + all functions.
* Understanding type deduction more important than ever.

Deduced return type specifiers:
*  auto: Use template (not auto!) type deduction rules.
*  No type deduced for braced initializers.
*  decltype(auto): Use decltype type deduction rules.
```C++
auto foo()	// return int
{
	int x = 1;
	return x;
}
decltype(auto) foo()	// return int&
{
	int x = 1;
	return x;
}
decltype(auto) foo()	// return int because of auto in declaration of x
{
	auto x = 1;
	return x;
}
decltype(auto) foo()	// return int&   because return contains an expression
{
	int x = 1;
	return (x);
}
```