# Move Symantics Part-1
URL : [Link to Video](https://www.youtube.com/watch?v=St0MNEU5b0o&list=PLbS5yOWDP6_o31dg3nvQ_LmRnR4x_5d28&index=2)

## Index
* The Basics of Move Semantics
* The New Special Member Functions
* The Move Constructor
* The Move Assignment Operator
* Parameter Conventions

### Basics of Move Semantics

```C++
Example 1
//what we want here is deep copy.
std::vector<int> v1{1,2,3,4,5};
std::vector<int> v2{};
v2=v1;


Example 2
//what we want here is deep copy.
std::vector<int> createVector()
{
    return std::vector<int>{1,2,3,4,5};
}
std::vector<int> v2{}

v2= createVector(); // creates a __tmp__ vector (R-Value : something which does not have a name or something whose address cant be taken, R-values can become L-values later on)
```

In above case if we do a deep copy, __tmp__ is anyway destroyed at the end, hence we have wasted a lot of space and time in copying. what we can do is transfer __tmp__ to v1 and then __tmp__ is removed of ownership of underlying memory.
This is possble because no one knows about __tmp__ other than this statement.

### std::move
```C++
template< typename T >
std::remove_reference_t<T>&& move( T&& t ) noexcept
{
    return static_cast<std::remove_reference_t<T>&&>( t );
}
```
In Example 1, we want to copy the contents of v1 (_begin, _end, _capacity) that is memory, we will have to use std::move. std::move simply typecasts the L-values to R-values even if its no possible, and then invoking the move semantics. The problem here is now v1 and v2 both have ownership of underlying memory which is problematic.

```C++
// Copy assignment operator
// (takes an lvalue)
vector& operator=(const vector& rhs);
```
This is invoked for all L-values like v2=v1


```C++
// Move assignment operator
// (takes an rvalue)
vector& operator=(vector&& rhs);
```
This is invoked for all R-values like v2=createVector() and v2=std::move(v1)


Rule of zero
If you can avoid def the move function (constructor, assginment)
We can avoid them and let compiler make them if all the data members are movable like int, std::string, unique_ptr, but if any of the data memebers are non-movable by default like a raw void* or int* pointer, then we will have to define it ourself

Lets see how to define
### Move Constructor
The Goal (w is where we are moving from)
* Transfer the content of w into this
* Leave w in a valid but undefined state

```C++
class Widget 
{
    private:
        int i{ 0 };
        std::string s{};
        int* pi{ nullptr };
    public:
        // …
        // Move constructor
        // when this function is called we know w in callee scope is R-value but whenwe are under move constructor scope it has a name `w` and hence it becomes a L-value in here, So we do s(w.s) during initialising phase w.s will actully be copied to s and not moved. int i however will always be capied as it is not movable
        Widget( Widget&& w ) noexcept // always do noexcept otherwise some functions might deliberatly call copy instead of move like vector's pushback
            :i(std::move(w.i)) // still copied
            ,s(std::move(w.s)) // moved
            ,pi(std::move(w.pi)) // pointer value is copied , so we have two var refereing to same location , we might free it twice, hence we need to explicitly transfer the ownership below
        {
            w.pi = nullptr; // we could have used pi( std::exchange(w.pi,nullptr) ) during init and we wont have to do this here.
            w.i = 0; // optional
        }
        // …
};
```

### Move Assignment Operator
* Clean up all visible resources
* Transfer the content of w into this
* Leave w in a valid but undefined state
```C++
class Widget 
{
    private:
        int i{ 0 };
        std::string s{};
        int* pi{ nullptr };
    public:
        // …
        // Move assignment operator
        Widget& operator=( Widget&& w ) noexcept 
        {
            delete pi; // always free resource we own
            i = std::move(w.i);
            s = std::move(w.s); // same as before
            pi = std::move(w.pi);
            w.pi = nullptr;
            return *this;
        }
        // …
};
```

### some Important Rules
* The default move operations are generated if no copy operation or destructor is user-defined
* The default copy operations are generated if no move operation is user-defined
> Note: =default and =delete count as user-defined!
