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
```
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