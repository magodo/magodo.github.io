---
layout: post
title: "C++ Functions"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Pass arguments

There are 3 primary methods of passing arguments to functions: pass by value, pass by reference, and pass by address.

## pass by value

Advantages of passing by value:

* Arguments passed by value can be variables (e.g. x), literals (e.g. 6), expressions (e.g. x+1), structs & classes, and enumerators.
* Arguments are never changed by the function being called, which prevents side effects.

Disadvantages of passing by value:

* Copying structs and classes can incur a significant performance penalty, especially if the function is called many times.

When to use pass by value:

* When passing fundamental data type and enumerators.

When not to use pass by value:

* When passing arrays, structs, or classes.

In most cases, pass by value is the best way to pass arguments to functions -- it is flexible and safe.

## pass by reference

Advantages of passing by reference:

* It allows a function to change the value of the argument, which is sometimes useful. Otherwise, __const references__ can be used to guarantee the function won’t change the argument.
* Because a copy of the argument is not made, it is fast, even when used with large structs or classes.
* We can return multiple values from a function.
* References must be initialized, so there’s no worry about null values.

Disadvantages of passing by reference:

* Because a non-const reference cannot be made to an rvalue (e.g. a literal or an expression), reference arguments must be normal variables.
* It can be hard to tell whether a parameter passed by non-const reference is meant to be input, output, or both. Judicious use of const and a naming suffix for out variables can help.
* It’s impossible to tell from the function call whether the argument may change. An argument passed by value and passed by reference looks the same. We can only tell whether an argument is passed by value or reference by looking at the function declaration. This can lead to situations where the programmer does not realize a function will change the value of the argument.

When to use pass by reference:

* When passing structs or classes (use const if read-only).
* When you need the function to modify an argument.

When not to use pass by reference:

* When passing fundamental types (use pass by value).

## pass by pointer

Advantages of passing by address:

* It allows a function to change the value of the argument, which is sometimes useful. Otherwise, `const` can be used to guarantee the function won’t change the argument.
* Because a copy of the argument is not made, it is fast, even when used with large structs or classes.
* We can return multiple values from a function.

Disadvantages of passing by address:

* Because literals and expressions do not have addresses, pointer arguments must be normal variables.
* All values must be checked to see whether they are null. Trying to dereference a null value will result in a crash. It is easy to forget to do this.
* Because dereferencing a pointer is slower than accessing a value directly, accessing arguments passed by address is slower than accessing arguments passed by value.

When to use pass by address:

* When passing pointer values.
* When passing built-in arrays (if you’re okay with the fact that they’ll decay into a pointer).

When not to use pass by address:

* When passing structs or classes (use pass by reference).
* When passing fundamental types (use pass by value).

As you can see, pass by address and pass by reference have almost identical advantages and disadvantages. Because pass by reference is generally safer than pass by address, pass by reference should be preferred in most cases.

Rule: Prefer pass by reference to pass by address whenever applicable.

## There is only "pass by value"

References are typically implemented by the compiler as pointers. This means that behind the scenes, pass by reference is essentially just a pass by address (with access to the reference doing an implicit dereference).

Also, pass by address is actually just passing an address by value!

Therefore, we can conclude that C++ really __passes everything by value__! The value of pass by address/reference comes solely from the fact that we can dereference the passed address to change the argument, which we can not do with a normal value parameter!

# inline function

One major downside of functions is that every time a function is called, there is a certain amount of performance overhead that occurs. This is because the CPU must store the address of the current instruction it is executing (so it knows where to return to later) along with other registers, all the function parameters must be created and assigned values, and the program has to branch to a new location. 

C++ offers a way to combine the advantages of functions with the speed of code written in-place: __inline functions__. The `inline` keyword is used to request that the compiler treat your function as an inline function. When the compiler compiles your code, all inline functions are expanded in-place -- that is, the function call is replaced with a copy of the contents of the function itself, which removes the function call overhead! 

The downside is that because the inline function is expanded in-place for every function call, this can make your compiled code quite a bit larger, especially if the inline function is long and/or there are many calls to the inline function.

Note that the `inline` keyword is only a __recommendation__ -- the compiler is free to ignore your request to inline a function. This is likely to be the result if you try to inline a lengthy function!

Finally, modern compilers have gotten really good at inlining functions automatically -- better than humans in most cases. Even if you don’t mark a function as inline, the compiler will inline functions that it believes will result in performance increases. Thus, in most cases, there isn’t a specific need to use the inline keyword. Let the compiler handle inlining functions for you.

## preprocessor macro vs inline function

Preprocessor macros are just substitution patterns applied to your code. They can be used almost anywhere in your code because they are replaced with their expansions before any compilation starts.

Inline functions are actual functions whose body is directly injected into their call site. They can only be used where a function call is appropriate.

Now, as far as using macros vs. inline functions in a function-like context, be advised that:

* Macros are not type safe, and can be expanded regardless of whether they are syntatically correct - the compile phase will report errors resulting from macro expansion problems.
* Macros can be used in context where you don't expect, resulting in problems
* Macros are more flexible, in that they can expand other macros - whereas inline functions don't necessarily do this.
* Macros can result in side effects because of their expansion, since the input expressions are copied wherever they appear in the pattern.
* Inline function are not always guaranteed to be inlined - some compilers only do this in release builds, or when they are specifically configured to do so. Also, in some cases inlining may not be possible.
* Inline functions can provide scope for variables (particularly static ones), preprocessor macros can only do this in code blocks {...}, and static variables will not behave exactly the same way.

# Function overload

__Function overloading__ is a feature of C++ that allows us to create multiple functions with the same name, so long as they have different _parameters_.

## Function return types are not considered for uniqueness

Note that the function’s return type is NOT considered when overloading functions.

## How function calls are matched with overloaded functions

Making a call to an overloaded function results in one of three possible outcomes:

1. A match is found. The call is resolved to a particular overloaded function.
2. No match is found. The arguments can not be matched to any overloaded function.
3. An ambiguous match is found. The arguments matched more than one overloaded function.

When an overloaded function is called, C++ goes through the following process to determine which version of the function will be called:

* First, C++ tries to find an exact match. This is the case where the actual argument exactly matches the parameter type of one of the overloaded functions. For example:

{% highlight CPP linenos %}
void print(char *value);
void print(int value);
 
print(0); // exact match with print(int)
{% endhighlight %}

Although 0 could technically match print(char*) (as a null pointer), it exactly matches print(int). Thus print(int) is the best match available.

* If no exact match is found, C++ tries to find a match through __promotion__. As well known, types can be automatically promoted via internal type conversion to other types. To summarize,

    * Char, unsigned char, and short is promoted to an int.
    * Unsigned short can be promoted to int or unsigned int, depending on the size of an int
    * Float is promoted to double
    * Enum is promoted to int

For example:

{% highlight CPP linenos %}
void print(char *value);
void print(int value);
 
print('a'); // promoted to match print(int)
{% endhighlight %}

In this case, because there is no print(char), the char ‘a’ is promoted to an integer, which then matches print(int).

* If no promotion is found, C++ tries to find a match through __standard conversion__. Standard conversions include:

    * Any numeric type will match any other numeric type, including unsigned (eg. int to float)(__might introduce ambiguous__)
    * Enum will match the formal type of a numeric type (eg. enum to float)(__might introduce ambiguous__)
    * Zero will match a pointer type and numeric type (eg. 0 to char*, or 0 to float)(__might introduce ambiguous__)
    * A pointer will match a void pointer

For example:

{% highlight CPP linenos %}
struct Employee; // defined somewhere else
void print(float value);
void print(Employee value);
 
print('a'); // 'a' converted to match print(float)
{% endhighlight %}

In this case, because there is no print(char), and no print(int), the ‘a’ is converted to a float and matched with print(float).

Note that all standard conversions are considered equal. No standard conversion is considered better than any of the others.

* Finally, C++ tries to find a match through __user-defined conversion__. Classes can define conversions to other types that can be implicitly applied to objects of that class. 

For example, we might define a class X and a user-defined conversion to int.

{% highlight CPP linenos %}
class X; // with user-defined conversion to int
 
void print(float value);
void print(int value);
 
X value; // declare a variable named cValue of type class X
print(value); // value will be converted to an int and matched to print(int)
{% endhighlight %}

Although value is of type class X, because this particular class has a user-defined conversion to int, the function call print(value) will resolve to the Print(int) version of the function.

## Ambiguous matches

Ambiguous match will be resulted because of:

* all standard conversions are considered equal
* all user-defined conversions are considered equal
* functions are of same function parameters but different return type

For example:

{% highlight CPP linenos %}
void print(unsigned int value);
void print(float value);
 
print('a');
print(0);
print(3.14159);
{% endhighlight %}

In the case of `print('a')`, C++ can not find an exact match. It tries promoting ‘a’ to an int, but there is no print(int) either. Using a standard conversion, it can convert ‘a’ to both an unsigned int and a floating point value. Because all standard conversions are considered equal, this is an ambiguous match.

`print(`0) is similar. 0 is an int, and there is no print(int). It matches both calls via standard conversion.

`print(3.14159)` might be a little surprising, as most programmers would assume it matches print(float). But remember that all literal floating point values are doubles unless they have the ‘f’ suffix. 3.14159 is a double, and there is no print(double). Consequently, it matches both calls via standard conversion.

Ambiguous matches are considered a _compile-time error_. Consequently, an ambiguous match needs to be disambiguated before your program will compile. There are two ways to resolve ambiguous matches:

1. Often, the best way is simply to define a new overloaded function that takes parameters of exactly the type you are trying to call the function with. Then C++ will be able to find an exact match for the function call.

2. Alternatively, explicitly cast the ambiguous parameter(s) to the type of the function you want to call. For example, to have print(0) call the print(unsigned int), you would do this:

{% highlight CPP linenos %}
print(static_cast<unsigned int>(0)); // will call print(unsigned int)
{% endhighlight %}

## Matching for functions with multiple arguments

If there are multiple arguments, C++ applies the matching rules to each argument in turn. The function chosen is the one for which each argument matches at least as well as all the other functions, with at least one argument matching better than all the other functions. In other words, the function chosen must provide a better match than all the other candidate functions for at least one parameter, and no worse for all of the other parameters.
(每个参数至少和其他的函数匹配的一样好，并且至少有一个参数比其他的函数匹配的更好)

In the case that such a function is found, it is clearly and unambiguously the best choice. If no such function can be found, the call will be considered ambiguous (or a non-match).


# Default parameters

C++ support default parameter in:

1. forward declaration THEN definition: function declaration 
2. definition only: function definition

For example:

{% highlight CPP linenos %}
void printValue(int x = 10, int y = 20, int z = 30)
{
    std::count << "Values: " << x << " " << y << " " << z << '\n';
}
{% endhighlight %}

Note that it is impossible to supply a user-defined value for `z` without also supplying a value for `x` and `y`. This is because C++ does not support a function call syntax such as `printValue(,,3)`. This has two major consequences:

* All default parameters must be the rightmost parameters. The following is not allowed:
{% highlight CPP %}
void printValue(int x=10, int y);  // not allowed
{% endhighlight %}
* If more than one default parameter exists, the leftmost default parameter should be the one most likely to be explicitly set by the user.

## Default parameter and function overloading

It is important to note that default parameters do NOT count towards the parameters that make the function unique. Consequently, following is not allowed:
{% highlight CPP linenos %}
void printValue(int x);
void printValue(int x, int y = 20);
{% endhighlight %}
If the caller were to call `printValue(10)`, the compiler would not be able to disambiguate whether the user wanted `printValue(int, 20)` or `printValue(int)`.

# Function Pointers

A pointer to a function could be defined as:

{% highlight CPP %}
void (*foo)(int);
{% endhighlight %}

Remember the right-left rule.

## Calling a function using a funtion pointer

There are two ways to do this:

1. Explicitly dereference: `(*foo)()`
2. Implicitly dereference: `foo()`

NOTE: Default arguments are resolved at __compile-time__ (that is, if you don't supply an argument for a defaulted parameter, the compiler substitutes one in for you when the code is compiled). However, function pointers are resolved at __run-time__. Consequently, default parameters can NOT be resolved when making a function call with a function pointer. You'll explicitly have to pass in values for any defaulted parameters in this case.

## Providing default functions

It is possible to set a default callback function parameter in a function call. For example:

{% highlight CPP %}
bool ascending(int, int);
void selectionSort(int *array, int size, bool (*comp)(int, int) = ascending);
{% endhighlight %}

## Making function pointers prettier with typedef

`typedef` can be used to make pointers to function look more like regular variables:

{% highlight CPP %}
typedef bool (*validateFcn)(int, int);
{% endhighlight %}

This defines a type called `validateFcn` that is a pointer to a function that takes two ints and returns a bool.

Now instead of doing this:

{% highlight CPP %}
void selectionSort(int *array, int size, bool (*comp)(int, int));
{% endhighlight %}

You can do this:

{% highlight CPP linenos %}
void selectionSort(int *array, int size, validateFcn comp);
{% endhighlight %}

One final __NOTE__: C++ doesn't allow the conversion of __function pointer__ to __void pointer__ (or vice-versa).

# std::vector capacity and stack behavior

`std::vector` contains two separate attributes: __size__ and __cap__acity. 

In the context of a `std::vector`, 

* size is how many elements are being used in the array
* capacity is how many elements were allocated.

`std::vector` will reallocate its memory if needed, but it would prefer not to, because resizing an array is computationally expensive. Consider the following:

{% highlight CPP linenos %}
#include <vector>
 
int main()
{
std::vector<int> array;
array = { 0, 1, 2, 3, 4 }; // okay, array length = 5
std::cout << "size: " << array.size() << "  capacity: " << array.capacity() << '\n';
 
array = { 9, 8, 7 }; // okay, array length is now 3!
std::cout << "size: " << array.size() << "  capacity: " << array.capacity() << '\n';
 
return 0;
}
{% endhighlight %}

This produces the following:

---
size: 5  capacity: 5

size: 3  capacity: 5

---

Array `subscripts` and `at()` are based on __size__, not capacity. If you try to access 4th element in example above, it will fail.

## Stack behavior with std::vector

We can make use of 3 functions provided by `std::vector` to simulate stack behavior:

* `push_back()`: pushes an element on stack
* `back()`: return value of the top element on the stack
* `pop_back()`: pop an element off the stack

Let's see an example:

{% highlight CPP linenos %}

void printStack(const std::vector<int> &stack)
{
    for (const auto &element : stack)
        std::cout << element << ' ';
    std::cout << "(cap " << stack.capacity() << " size " << stack.size() << ")\n";
}
 
int main()
{
    std::vector<int> stack;
 
    printStack(stack);
 
    stack.push_back(5); // push_back() pushes an element on the stack
    printStack(stack);
 
    stack.push_back(3);
    printStack(stack);
 
    stack.push_back(2);
    printStack(stack);
 
    std::cout << "top: " << stack.back() << '\n'; // back() returns the last element
 
    stack.pop_back(); // pop_back() back pops an element off the stack
    printStack(stack);
 
    stack.pop_back();
    printStack(stack);
 
    stack.pop_back();
    printStack(stack);
 
    return 0;
}

{% endhighlight %}

This prints:

---
(cap 0 size 0)

5 (cap 1 size 1)

5 3 (cap 2 size 2)

5 3 2 (cap 3 size 3)

top: 2

5 3 (cap 3 size 2)

5 (cap 3 size 1)

(cap 3 size 0)

---

Unlike array subscripts or at(), the stack-based functions will resize the std::vector if necessary. In the example above, the vector gets resized 3 times (from a capacity of 0 to 1, 1 to 2, and 2 to 3).

Because resizing the vector is expensive, we can tell the vector to allocate a certain amount of capacity up front using the `reserve()` function.

# Handling Errors

## Assert

An __assert statement__ is a preprocessor macro that evaluates a conditional expression. 

* If the conditional expression is true, the assert statement does nothing;
* Otherwise, an error message is displayed and the program is terminated. 

  This error message contains the conditional expression that failed, along with the name of the code file and the line number of the assert.

Assert functionality lives in `<cassert>` header.

Rule: Favor `asset` statement liberally throughout your code where you want your code to quit if error occurs.

## NDEBUG and other considerations

The `assert` function comes with a small performance cost that is incurred each time the assert condition is checked. Furthermore, `asserts` should (ideally) never be encountered in production code (because your code should already be thoroughly tested). Consequently, many developers prefer that asserts are only active in debug builds. C++ comes with a way to turn off asserts in production code: `#define NDEBUG`:

{% highlight CPP linenos %}
#define NDEBUG

// all assert() calls will now be ignored to the end of the file
{% endhighlight%}

## Exception

C++ provides one more method for detecting and handling errors known as exception handling. The basic idea is that when an error occurs, the error is “thrown”. If the current function does not “catch” the error, the caller of the function has a chance to catch the error. If the caller does not catch the error, the caller’s caller has a chance to catch the error. The error progressively moves up the stack until it is either caught and handled, or until main() fails to handle the error. If nobody handles the error, the program typically terminates with an exception error.

# Ellipsis

C++ provides a special specifier known as __ellipsis__(aka "..."), that allows us to define functions which take a _variable_ number of parameters.

Function that uses ellipsis take the form:

{% highlight CPP %}
return_type function_name(argument_list, ...)
{% endhighlight%}

Note that functions that use ellipsis must have at least one non-ellipsis parameter.

