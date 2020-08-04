# RAII
URL : [Link to Video](https://www.youtube.com/watch?v=7Qgd9B1KuMQ)

## Index 
* Motivating the special members and the Rule of Three 
* Curious pitfall with (indirect) self-copy 
* RAII and exception safety 
* Deleting, defaulting, and the Rule of Zero 
* Move semantics and the Rule of Five (or Four) 
* Recap and examples of RAII types 

### Classes that manage resources
> A “resource,” for our purposes, is anything that requires special (manual) management

C++ programs can manage many different kinds of “resources.”
* Allocated memory (malloc/free, new/delete, new[]/delete[])
* POSIX file handles (open/close)
* C FILE handles (fopen/fclose)
* Mutex locks (pthread_mutex_lock/pthread_mutex_unlock)
* C++ threads (spawn/join)

### A naïve implementation of vector

```C++
class NaiveVector 
{
    int *ptr_;
    size_t size_;
    public:
    NaiveVector() : ptr_(nullptr), size_(0) {}

    void push_back(int newvalue) 
    {
        int *newptr = new int[size_ + 1];
        std::copy(ptr_, ptr_ + size_, newptr);
        delete [] ptr_;
        ptr_ = newptr;
        ptr_[size_++] = newvalue;
    }
};
```

Above code has a memory leak when the object of this class is destroyed i.e. _ptr is not deleted.

Lets write a destructor, This NaiveVector no longer leaks memory on destruction
```C++
~NaiveVector() { delete [] ptr_; }
```

This class still has many bugs, Lets take a look at them :(

```C++
{
    NaiveVector v;
    v.push_back(1);
    {
        /*
         * This line below invokes the implicitly
         * generated (defaulted) copy
         * constructor of NaiveVector.
         * A defaulted copy constructor
         * simply copies each member.
         * So now both object manages same memory.
         */
        NaiveVector w = v; 
    } // <- At this point w is destroyed resulting in destroying v too, as per our destructor
    std::cout << v[0] << "\n"; // This here will crash because w has destoyed and freed the memory
} // This line will try to free v's resources but they are already freed resuling in double delete
```

#### Copy constructor to Rescue
```C++
NaiveVector(const NaiveVector& rhs) 
{
    ptr_ = new int[rhs.size_];
    size_ = rhs.size_;
    std::copy(rhs.ptr_, rhs.ptr_ + size_, ptr_);
}
```

* Whenever you write a destructor, you probably need to write a copy constructor as well.
* The destructor is responsible for freeing resources to avoid leaks. The copy constructor is responsible for duplicating resources to avoid **double frees**

Lets take more application of our class NaiveVector

```C++
{
    NaiveVector v;
    v.push_back(1);
    {
        NaiveVector w;
        w = v;
    } // destroys w freeing up the v as well
    std::cout << v[0] << "\n"; // seg fault v is free
} // double free of v
```

This above code again have same problems defined previously, because now we are not using copy constructor but copy assignment.

Lets implement the correct behaviour

#### Copy Assignment

```C++
NaiveVector& operator=(const NaiveVector& rhs) 
{
    NaiveVector copy (rhs); // calling copy constructor
    copy.swap(*this); // we need to write swap as well
    return *this;
}
```

* Whenever you write a destructor, you probably need to write a copy constructor and a copy assignment operator

### The Rule of Three

If your class directly manages some kind of resource (such as a new’ed pointer), then you almost certainly need to hand-write three special member functions:
* A destructor to free the resource
* A copy constructor to copy the resource
* A copy assignment operator to free the left-hand resource and copy the right-hand one
> Use the copy-and-swap idiom to implement assignment

##### Why copy and swap?
You might simply overwrite each member one at a time, like this.

```C++
NaiveVector& NaiveVector::operator=(const NaiveVector& rhs) 
{
    delete[] ptr_;
    ptr_ = new int[rhs.size_];
    size_ = rhs.size_;
    std::copy(rhs.ptr_, rhs.ptr_ + size_, ptr_);
    return *this;
}
```

* But this code is not robust against self-assignment. (`delete ptr_` this create the problem)
* looks verbose and ugly :|
* this is also problamatic with recursive data structure
```C++
struct A 
{
    NaiveVector<shared_ptr<A>> m;
};
NaiveVector<shared_ptr<A>> v;
// when the destructor of a parent node is called it frees up all child too :(
```

So should always use Copy and swap idiom
We make a complete copy of rhs before the first modification to *this. So any aliasing relationship between rhs and *this cannot trip us up.

### The Rule of Zero
If your class does not directly manage any resource, but merely uses library components such as vector and string, then you should strive to write no special member functions. 

Default them all!
* Let the compiler implicitly generate a defaulted destructor
* Let the compiler generate the copy constructor
* Let the compiler generate the copy assignment operator
* (But your own swap might improve performance)

### Move assignment and copy constructor

Move operations are required for perfomance reasons most of the time but sometimes copying an obbject does not make sense like lock

```C++
NaiveVector(NaiveVector&& rhs) 
{
    ptr_ = std::exchange(rhs.ptr_, nullptr);
    size_ = std::exchange(rhs.size_, 0);
}

// this looks very similar to copy assignment :)
NaiveVector& NaiveVector::operator=(NaiveVector&& rhs) 
{
    NaiveVector copy(std::move(rhs));
    copy.swap(*this);
    return *this;
}
```

A quick recap
* A destructor to free the resource
* A copy constructor to copy the resource
* A move constructor to transfer ownership of the resource
* A copy assignment operator to free the left-hand resource and copy the right-hand one
* A move assignment operator to free the left-hand resource and transfer ownership of the right-hand one

Now this is no longer NaiveVector :)

### Examples of resource management

#### unique_ptr
unique_ptr manages a raw pointer to a uniquely owned heap allocation
* Destructor frees the resource
    * Calls delete on the raw pointer
* Copy constructor copies the resource
    * Copying doesn’t make sense. We =delete this member function.
* Move constructor transfers ownership of the resource
    * Transfers the raw pointer, then nulls out the right-hand side
* Copy assignment operator frees the left-hand resource and copies the right-hand one
    * Copying doesn’t make sense. We =delete this member function.
* Move assignment operator frees the left-hand resource and transfers ownership of the right-hand one
    * Calls delete on the left-hand ptr, transfers, then nulls out the right-hand ptr

#### shared_ptr

shared_ptr manages a reference count.
* Destructor frees the resource
    * Decrements the refcount (and maybe cleans up if the refcount is now zero)
* Copy constructor copies the resource
    * Increments the refcount
* Move constructor transfers ownership of the resource
    * Leaves the refcount the same, then disengages the right-hand side
* Copy assignment operator frees the left-hand resource and copies the right-hand one
    * Decrements the old refcount, increments the new refcount
* Move assignment operator frees the left-hand resource and transfers ownership of the right-hand one
    * Decrements the old refcount, then disengages the right-hand side

#### unique_lock
unique_lock manages a lock on a mutex.
* Destructor frees the resource
    * Unlocks the mutex if “engaged”
* Copy constructor copies the resource
    * Copying doesn’t make sense. We =delete this member function.
* Move constructor transfers ownership of the resource
    * Leaves the mutex in the same state, then disengages the right-hand side
* Copy assignment operator frees the left-hand resource and copies the right-hand one
    * Copying doesn’t make sense. We =delete this member function.
* Move assignment operator frees the left-hand resource and transfers ownership of the right-hand one
    * Unlocks the old mutex if “engaged”; then transfers ownership from the right-hand side

#### ifstream
ifstream manages a POSIX file handle and an associated buffer.

* Destructor frees the resource
    * Calls close on the handle
* Copy constructor copies the resource
    * We =delete this member function; but you could imagine calling dup on the handle (and giving the copy a fresh buffer, to avoid duplicated output)
* Move constructor transfers ownership of the resource
    * Transfers the handle and the contents of the buffer
* Copy assignment operator frees the left-hand resource and copies the right-hand
    * We =delete this member function; but you could imagine close/dup
* Move assignment operator frees the left-hand resource and transfers ownership of the right-hand one
    * Calls close, then transfers the handle and contents of the buffer