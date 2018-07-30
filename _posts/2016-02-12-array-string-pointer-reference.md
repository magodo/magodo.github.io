---
layout: post
title: "C++ Array, String, Pointer and Reference"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Favor C++ std::string over C-style string

Unless you have a specific, compelling reason to use C-style strings, use std::string instead. std::string is easier, safer, and more flexible

# Pointers

## The address-of operator(&) returns a pointer

It’s worth noting that the address-of operator (&) doesn’t return the address of its operand as a literal. Instead, it returns a pointer containing the address of the operand, whose type is derived from the argument (e.g. taking the address of an int will return the address in an int pointer).

## Declaration

* When declaring a pointer variable, put the asterisk next to the variable name.
* When declaring a function, put the asterisk of a pointer return value next to the type.

## Null Pointer

* Initialize your pointers to a null value (0) if you’re not giving them another value.
* Don't use `NULL` in C++ since it's technically not part of C++(defined by C).
* Starting from C++11:
    * Use `nullptr` to initialize your pointer to a null value (`nullptr` is both a _keyword_ and an _rvaule constant_)
    * C++11 also introduces a new type called `std::nullptr_t` (in header <cstddef>). `std::nullptr_t` can only hold one value: `nullptr`!

# Dynamic memory allocation

## allocate

1. Scalar form(non-array):

{% highlight CPP linenos %}
int *ptr = new int; // dynamically allocate an integer and assign the address to ptr
{% endhighlight %}

2. Array form:

{% highlight CPP linenos %}
int *array = new int[size]; // use array new.  Note that size does not need to be constant!
{% endhighlight %}

### Initialze dynamically allocated arrays

__Initialize to 0__

{% highlight CPP linenos %}
int *array = new int[size]();
{% endhighlight %}

__Initialize to non-zero__

1. Prior to C++11: Loop and initialize manually
2. Since C++11: Use _uniform initialization_
{% highlight CPP linenos %}
int *array = new int[5] { 9, 7, 5, 3, 1 }; // initialize a dynamic array in C++11
{% endhighlight %}

## delete

1. Scalar form(non-array):

{% highlight CPP linenos %}
// assume ptr has previously been allocated with operator new
delete ptr; // return the memory pointed to by ptr to the operating system
ptr = 0; // set ptr to be a null pointer (use nullptr instead of 0 in C++11)
{% endhighlight %}

2. Array form:

{% highlight CPP linenos %}
delete[] array; // use array delete to deallocate array
array = 0; // use nullptr instead of 0 in C++11
{% endhighlight %}

## dangling pointers

Pointers that are pointing to deallocated memory are called __dangling pointer__.

Deallocating memory may create multiple dangling pointers. Consider the following example:

{% highlight CPP linenos %}
#include <iostream>
 
int main()
{
    int *ptr = new int; // dynamically allocate an integer
    int *otherPtr = ptr; // otherPtr is now pointed at that same memory location
 
    delete ptr; // return the memory to the operating system.  ptr and otherPtr are now dangling pointers.
    ptr = 0; // ptr is now a nullptr
 
    // however, otherPtr is still a dangling pointer!
 
    return 0;
}
{% endhighlight %}

Rule: To avoid dangling pointers, after deleting memory, set all pointers pointing to the deleted memory to 0 (or nullptr in C++11).

## no throw "new"

By default, if new fails, a `bad_alloc` exception is thrown. If this exception isn’t properly handled, the program will simply terminate (crash) with an unhandled exception error.

There’s an alternate form of `new` that can be used instead to tell new to return a _null pointer_ if memory can’t be allocated. This is done by adding the `constant std::nothrow` between the new keyword and the allocation type:

{% highlight CPP linenos %}
int *value = new (std::nothrow) int; // value will be set to a null pointer if the integer allocation fails
{% endhighlight %}

Note that if you then attempt to dereference this memory, your program will crash. Consequently, the best practice is to check all memory requests to ensure they actually succeeded before using the allocated memory.

{% highlight CPP linenos %}
int *value = new (std::nothrow) int; // ask for an integer's worth of memory
if (!value) // handle case where new returned null
{
    // do something
}
{% endhighlight %}


## Resizing arrays

Dynamically allocating an array allows you to set the array size at the time of allocation. However, C++ does not provide a built-in way to resize an array that has already been allocated. It is possible to work around this limitation by dynamically allocating a new array, copying the elements over, and deleting the old array. However, this is error prone, especially when the element type is a class (which have special rules governing how they are created).

Consequently, we recommend avoiding doing this yourself. Use `std::vector`(resizable array) instead!

# Reference variables

A __reference__ is a type of C++ variable that acts as an alias to another variable. 

References are declared by using an ampersand (`&`) between the reference type and the variable name:

{% highlight CPP linenos %}
int value = 5; // normal integer
int &ref = value; // reference to variable value
{% endhighlight %}

In this context, the ampersand does not mean “address of”, it means “reference to”.

Note that you can’t initialize a non-const reference with a const object:

{% highlight CPP linenos %}
const int x = 5;
int &ref = x; // invalid, non-const reference to const object
{% endhighlight %}

## References are implicitly const

References are implicitly const. Like normal constant objects, references __must__ be initialized:

{% highlight CPP linenos %}
int value = 5;
int &ref = value; // valid reference
 
int &invalidRef; // invalid, needs to reference something
{% endhighlight %}

References must always be initialized with a valid object. Unlike pointers, which can hold a null value, there is no such thing as a null reference.

Because references are implicitly const, a reference can not be “redirected” (assigned) to another variable. Consider the following snippet:

{% highlight CPP linenos %}
int value1 = 5;
int value2 = 6;
 
int &ref = value1; // okay, ref is now an alias for value1
ref = value2; // assigns 6 (the value of value2) to value1 -- does NOT change the reference!
{% endhighlight %}

## Const references

Although references are implicitly const, when you need to initialize a reference with a const value, you have to declare a const reference:

{% highlight CPP linenos %}
const int value = 5;
const int &ref = value; // ref is a const reference to value
{% endhighlight %}

## Const references to literal values

You can assign const references to literal values, though there is typically not much need to do so:

{% highlight CPP linenos %}
const int &rnRef = 6;
{% endhighlight %}

## Reference as function parameter

Most often, references are used as function parameters because they allow us to pass a parameter to a function without making a copy of the value itself (we just copy the reference):

{% highlight CPP linenos %}
#include <iostream>
 
// ref is a reference to the argument passed in, not a copy
void changeN(int &ref)
{
    ref = 6;
}
 
int main()
{
    int n = 5;
 
    std::cout << n << '\n';
 
    changeN(n);
 
    std::cout << n << '\n';
    return 0;
}
{% endhighlight %}

## References as shortcuts

References can be used for easier access to nested data. Consider the following struct:

{% highlight CPP linenos %}
struct Something
{
    int value1;
    float value2;
};
 
struct Other
{
    Something something;
    int otherValue;
};
 
Other other;
{% endhighlight %}

Let's say we needed to work with the value1 field of the Something struct of other. Normally, we’d access that member as `other.something.value1`. If there are many separate accesses to this member, the code can become messy. References allow you to more easily access the member:

{% highlight CPP linenos %}
int &ref = other.something.value1;
// ref can now be used in place of other.something.value1
{% endhighlight %}

The following two statements are thus identical:

{% highlight CPP linenos %}
other.something.value1 = 5;
ref = 5;
{% endhighlight %}

## References vs pointers

Because references must be initialized to valid objects and can not be changed once set, references are generally much safer to use than pointers. However, they are also a bit more limited in functionality.

If a given task can be solved with either a reference or a pointer, the reference should generally be preferred. Pointers should only be used in situations where references are not sufficient (such as dynamically allocating memory).

# For each loop

This is a new type of loop introduced since __C++ 11__. 

The for-each statement has a syntax that looks like this:

{% highlight CPP linenos %}
for (element_declaration : array)
   statement;
{% endhighlight %}

When this statement is encountered, the loop will iterate through each element in array, assigning the __value__(not reference) of the current array element to the variable declared in `element_declaration`. For best results, `element_declaration` should have the same type as the array elements, otherwise type conversion will occur.

For example:

{% highlight CPP linenos %}
#include <iostream>
 
int main()
{
    int fibonacci[] = { 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89 };
    for (int number : fibonacci) // iterate over array fibonacci
       std::cout << number << ' '; // we access the array element for this iteration through variable number
 
    return 0;
}
{% endhighlight %}

## For each loops and the auto keyword

Because element_declaration should have the same type as the array elements, this is an ideal case in which to use the auto keyword, and let C++ deduce the type of the array elements for us.

Here’s the above example, using auto:

{% highlight CPP linenos %}
#include <iostream>
 
int main()
{
    int fibonacci[] = { 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89 };
    for (auto number : fibonacci) // type is auto, so number has its type deduced from the fibonacci array
       std::cout << number << ' ';
 
    return 0;
}
{% endhighlight %}

## For-each loops and references

In previous _for each loop_,  each array element iterated over will be __copied__ into variable element. Copying array elements can be expensive, and most of the time we really just want to refer to the original element. Fortunately, we can use references for this:

{% highlight CPP linenos %}
    int array[5] = { 9, 7, 5, 3, 1 };
    for (auto &element: array) // The ampersand makes element a reference to the actual array element, preventing a copy from being made
        std::cout << element << ' ';
{% endhighlight %}

In the above example, element will be a reference to the currently iterated array element, avoiding having to make a copy. Also any changes to element will affect the array being iterated over, something not possible if element is a normal variable.

And, of course, it’s a good idea to make your element `const` if you’re intending to use it in a read-only fashion:

{% highlight CPP linenos %}
    int array[5] = { 9, 7, 5, 3, 1 };
    for (const auto &element: array) // element is a const reference to the currently iterated array element
        std::cout << element << ' ';
{% endhighlight %}

Rule: Use __references__ or __const references__ for your element declaration in for-each loops for performance reasons..

## For-each loops works for list-like structures 

For-each loops don’t only work with fixed arrays, they work with many kinds of list-like structures, such as vectors (e.g. std::vector), linked lists, trees, and maps. 

## For-each doesn’t work with pointers to an array

In order to iterate through the array, for-each needs to know how big the array is, which means knowing the array size. Because arrays that have decayed into a pointer do not know their size, for-each loops will not work with them!

{% highlight CPP linenos %}
#include <iostream>
 
int sumArray(int array[])
{
    int sum = 0;
    for (const auto &number : array) // compile error, the size of array isn't known
        sum += number;
 
    return sum;   
}
 
int main()
{
     int array[5] = { 9, 7, 5, 3, 1 };
     std::cout << sumArray(array);
     return 0;
}
{% endhighlight %}

## Can I get the index of an the current element?

For-each loops do not provide a direct way to get the array index of the current element. This is because many of the structures that for-each loops can be used with (such as linked lists) are not directly indexable!

# void pointer

Avoid use `void` pointer unless absolutely necessary, as they effectively allow you to avoid type checking.

* For some case to allow different types of data to be passed into a function, think of __function overloading__ first
* For some case to handle multiple data types, think of __template__ first

# An introduction to `std::array` and `std::vector`

There are built-in types of arrays in C++:

* Fixed array
* Dynamic array

They have downsides: 

* Fixed arrays decay into pointers, losing the array length information when they do
* Dynamic arrays have messy deallocation issues and are challenging to resize without error

## std::array

Since C++ 11, `std:array` provides __fixed array__ functionality that won't decay when passed into a function.

### Declare
Declaring a `std::array` variable is easy:

{% highlight CPP linenos %}
#include <array>
 
std::array<int, 3> array; // declare an integer array with length 3
{% endhighlight %}

Note:

* Just like the native implementation of fixed arrays, the size of a std::array must be set at compile time
* Unlike built-in fixed arrays, with std::array you can not omit the array size when providing an initializer

### Assignment

You can assign values to the array using an initializer list:

{% highlight CPP linenos %}
std::array<int, 5> array { 9, 7, 5, 3, 1 }; // initialization list
array = { 0, 1, 2, 3, 4 }; // okay
array = { 9, 8, 7 }; // okay, elements 3 and 4 are set to zero!
array = { 0, 1, 2, 3, 4, 5 }; // not allowed, too many elements in initializer list!
{% endhighlight %}

### Access element

Accessing array values using the subscript operator works. However, just like built-in fixed arrays, the subscript operator does not do any bounds-checking. If an invalid index is provided, bad things will probably happen.

`std::array` supports a second form of array element access (the `at()` function) that does bounds checking:

{% highlight CPP linenos %}
std::array<int, 5> array { 9, 7, 5, 3, 1 };
array.at(1) = 6; // array element 1 valid, sets array element 1 to value 6
array.at(9) = 10; // array element 9 is invalid, will throw error
{% endhighlight %}

### Size

The size() function can be used to retrieve the size of the array:

{% highlight CPP linenos %}
std::array<double, 5> array { 9.0, 7.2, 5.4, 3.6, 1.8 };
std::cout << array.size();   // prints 5
{% endhighlight %}

Because `std::array` doesn’t decay to a pointer when passed to a function, the `size()` function will work even if you call it from within a function.

Note that the standard library uses the term “size” to mean the array length — do not get this confused with the results of `sizeof()` on a native fixed array, which returns the actual size of the array in memory (the size of an element multiplied by the array length).

## std::vector

Since C++ 03, `std::vector` provides __dynamic array__ functionality that handles its own memory management. This means you can create arrays that have their length set at runtime, without having to explicitly allocate and deallocate memory using new and delete.

### Declare

Declaring a `std::vector` is simple:

{% highlight CPP linenos %}
#include <vector>
 
// no need to specify size at initialization
std::vector<int> array; 
std::vector<int> array2 = { 9, 7, 5, 3, 1 }; // use initializer list to initialize array
std::vector<int> array3 { 9, 7, 5, 3, 1 }; // use uniform initialization to initialize array (C++11 onward)
{% endhighlight %}

Note that in both the uninitialized and initialized case, you do not need to include the array size at compile time. This is because std::vector will dynamically allocate memory for its contents as needed.

### Assignment

As of C++11, you can also assign values to a std::vector using an initializer-list:

{% highlight CPP linenos %}
array = { 0, 1, 2, 3, 4 }; // okay, array size is now 5
array = { 9, 8, 7 }; // okay, array size is now 3
{% endhighlight %}

### Access element

Just like std::array, accessing array elements can be done via the [] operator (which does no bounds checking) or the at() function (which does bounds checking):

{% highlight CPP linenos %}
array[2] = 2;
array.at(3) = 3;
{% endhighlight %}

### Self-cleanup prevents memory leaks

When a vector variable goes out of scope, it automatically deallocates the memory it controls (if necessary). This is not only handy (as you don’t have to do it yourself), it also helps prevent memory leaks.

### Vectors remember their size

Unlike built-in dynamic arrays, which don’t know the size of the array they are pointing to, std::vector keeps track of its size. We can ask for the vector’s size via the size() function:

{% highlight CPP linenos %}
#include <vector>
#include <iostream>
 
int main()
{
    std::vector<int> array { 9, 7, 5, 3, 1 };
    std::cout << "The size is: " << array.size() << '\n';   // print 5
 
    return 0;
}
{% endhighlight %}

### Resizing an array

Resizing a built-in dynamically allocated array is complicated. Resizing a std::vector is as simple as calling the resize() function:

{% highlight CPP linenos %}
#include <vector>
#include <iostream>
 
int main()
{
    std::vector<int> array { 0, 1, 2 };
    array.resize(5); // set size to 5, zero-pad the new elements
 
    std::cout << "The size is: " << array.size() << '\n'; // print 5
 
    for (auto const &element: array)
        std::cout << element << ' ';
 
    return 0;
};
{% endhighlight %}

Note:

1. When we resized the array, the existing element values were preserved
2. New elements are initialized to the default value for the type (which is 0 for integers)
3. Vectors may be resized to be smaller

### Compacting bools

std::vector has another cool trick up its sleeves. There is a special implementation for std::vector of type bool that will compact 8 booleans into a byte! This happens behind the scenes, and is largely transparent to you as a programmer.
