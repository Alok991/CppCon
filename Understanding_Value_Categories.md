# Understanding Value Categories
URL : [Link to Video](https://www.youtube.com/watch?v=XS2JddPq7GQ&list=PL5qoVlA-tv09ykIIPHP9N6vgJaFPnYWCa&index=6)

## Index
**// YET TO BE MADE**

### What is Value Categories?
Value categoris are not language features but semantic properties of C++ expressions, It helps us understand how user-defined data-types can behave like builtin types.

### Value Categories in C
* l-values -> are values which can come on left side of an expression.
* r-values -> are values which can come on right side of an expression.

> Value categores in C++ are much more complicated. This Talk takes us through how varous values categories came into existence.

#### why do we need two categories in first place ?
* So that compilers can assume rvalue dont necessarily occupy space
* This offers freedom in machine code generation.
```C++
int n;  // declarationg for an int obj
n = 1;  // an assignment expression
```

In asm this might look like
```AS
one :           ;a label for following location in datastore
    .word 1     ;allocate storage for holding value 1

mov n, one      ; copy value at tag one to n
```
or like this 
```
mov n #1
```
So here 1 as r-value doesnot have any data storage but is a part of instruction.


So in general we have this def of rvalue and lvalue
* lvalue is an expression that refres to an object and rvalue is an expression not an lvalue.

This is true only for non-class types. :(

Literal likes 3, 3.443 and char literals like 'a' are rvalues, as they dont occupy any space

However string literals like "abcd" etc are lvalue as they occupy space.

### **l-value to r-value conversion**
```C++
// Example 1
int m,n = 5;
m = n;  //here n is l-value but used as r-value
```
This is known as lvalueToRvalue conversion

The concept of rvalue and lvalue apply to all expressions not just assignment
for example binary operator+ must have both as suitable types.
```C++
// Example 2
int x;
// both of these are valid
x+2; // lvalue + rvalue
2+x; // rvalue + lvalue
```
#### But what about the result in above expressions? 
The result of x+2 (2+x) will be placed in a compiler generated temp obj, often a cpu register
such temp obj are rvalues.

### Data Storage for expressions
* Conceptually rvalues(of non-class types) dont occupy data storage in program but sometimes they can. In example 1, if we assign a very large number to m i.e. `m=10e7`
* But C++ insist that you program assuming rvalue never takes memory
* Conceptually lvalues always occupy data space in program (compiler can eliminate but it wont make any diff to us)

## R-Value of class Types(class, struct or union)
Conceptually They do occupy space
```C++
struct S
{
    int x,y;
};
S s = {1,3};

int i = s.x;    // in this expression s is an lvalue
```
Asm for above code can be as follows
```AS
main: # @main
  push rbp
  mov rbp, rsp
  xor eax, eax
  mov rcx, qword ptr [.L__const.main.s]
  mov qword ptr [rbp - 8], rcx
  mov ecx, dword ptr [rbp - 8]
  mov dword ptr [rbp - 12], ecx     // load ecx (which contains s.x) into i which is basically base + offset calc
  pop rbp
  ret
.L__const.main.s:
  .long 1 # 0x1
  .long 3 # 0x3
```
In above exmaple compiler does the `base + offset` calculation and set to i

Now consider another example 
```C++
S foo();
int j = foo().x;    // access x member of an r-value of class type
```

Again compiler has to do the `base + offset` calculation to get value of `foo().x`, therefore the return value of foo() must have an base address
Hence return type of foo() will have storage too and r-value of this types must be trated differently

## Non-Modifiable L-values
* Not all l-values can appear on the left side of an assignment.
```C++
char const name[] = "Alok";
name[0] = 'D';      // Error
```

Here name is an l-value but cant be assigned.

So rvalue and lvalue provides a vocabulary for describing behavioral diff, such as in const obj and enum constants.
See Example below
```C++
enum {max =  100};      // unnamed enum values converts to int in expressions

int n = MAX /2;         // here MAX is an R-value its a true contant and dont have space

MAX += 100;             // Error
int* p = &MAX;          // Error, it does not have an addr
```

but if we define MAX as below we can do all these things
```C++
int const MAX = 100;       // Now this is an const l-value

MAX += 100;                 // Error
int const* p = &MAX;        // Allowed as MAX lives as an obj and have addr
```

from a programmers point of view both are constants and can be used in exact same way but differs in their categories.
> constexpr also occupy space ```constexpr int m = 10; const int* i = &m; ```

### **Recap**

|category | can_take_address_of | can_assign_to |
|---------|---------------------|---------------|
|lvalue|yes|yes|
|non-modifiable l-value| yes | no|
|non-class r-value|no|no|

For class rvalue we will study a bit more :p

## Reference Types
The concepts of Lvalues and Rvalues help us exaplain concepts of reference in C++

References under the hood are just like constant pointer.
but by using references we can make non-builtin types behave like builtin types.(we will se how ..)

```C
enum month { jan, feb, mar, ... dec, month_end};
typedef enum month month;

for(month m = jan ; m< month_end ; ++m)
{
    //...
}
```
Above code compiles and execute in `C` but not in `C++`, reason being C++ enum are treated as distinct types

So we will need to overload `++` operator for month type

```C++
month operator++(month m)
{
    // ++ operator increments the variable and returns too
    return (m = static_cast<month>(m+1)); 
}
```
Here we did not increments m but copied it and incremented that copy. It wont work
Also this allows `++jan` which should not be allowed, but ++42 must work and it wont work in here.
We can try same function with pass by poiner but it wont work because ++&m is wrong and does not behave like builtin types anymore.
Also We cant pass pointer types to operator overload.

So Standard introduced l-value ref.
```C++
month& operator++(month &m)
{
    // ++ operator increments the variable and returns too
    return (m = static_cast<month>(m+1)); 
}
```

This works and feel like builtin types, but it still does not work for 42++;

What we can do is `ref to const T`. ref to const T binds to a non-lvalue . The compiler constructs a __temp__ object and bind it to that. So that ref has something to bind to.
```C++
double const& m = 3;    // allowed by making __temp__ object and bind to it;, when m is destroyed the compiler destroys m too
```

##### So why does ref to const behaves this way ?
So that pass by ref behaves completely like pass by value, i.e. foo(x) and foo(1) both works, 
It wont change arguments just like pass by copy and it does not modify

## Two types of rValue
So rvlues does not occupy storage unless they bind to a __temp__ obj

We can differentiate between them as 

**prvalue** -> pure r-Value , which does not occupy space
**xvalue**  -> expiring value, which does occupy space

The __temp__ object is created from prvalue to xvalue via Temporary materialization conversion
```C++
int f =9;
f+10;   // lvalue + rvalue => return rvalue

int operator+(int const&, int const&);
```

### C++11 introduced rvalue references
So ref till now becomes lVlaue ref
```C++
int&& i = 10;
```

RVlaue ref only binds to rvalue
Binding an rvalue Reference to an rValue triggers Temprorary materializaion conversion.
Modern C++ implements move operators in terms of rValue ref to avoid copy

```C++
string s1,s2,s3;

s1=s2;        //string::operator=(string const&);
s1=s2+s3;     //string::operator=(string&&)
```

When we look inside the `string::operator=(string&&)`
```C++
string& string::operator+(string && s)
{
  string temp(s);   // calls string::operator=(string const &);  s is now lvalue
}
```

So above `s` is being treated as l-vlaue reference to const, Hence s in the function argument s has to become xvalue, because it needs to have space

### Lvalue as Xvalue

Sometimes we want to move from lvalue, like swap function
```C++
void swap(T a, T b)
{
    T t{a};
    a=b;
    b=t;
}
```
Here we know there is no need to preserve a and we can move , and its safe to move from an Lvalue if its going to expire
So if its going to expiry lets make it xValue, but compiler can deduce it, we need to inform it by using std::move


### Graph for various values
```
                      expression
                          /\
                         /  \
                        /    \
                    gLvalue  rvalue
                    / \       / \
                   /   \     /   \
                  /     \   /     \
                lvalue  xvalue   prValue
```
`gValue -> generalised Lvalue`