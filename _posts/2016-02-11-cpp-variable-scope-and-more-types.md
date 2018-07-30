---
layout: post
title: "C++ Variable Scopre and More Types"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Local variable, scope and duration

Rule: Define variables in the smallest scope that they are used.

Rule: Avoid using nested variables with the same names as variables in an outer block.

# Global variables

## Rule: Avoid to use a same-named local variable as global variable

Local variables with the same name as a global variable hide the global variable inside the block that the local variable is declared in. However, the global scope operator `::` can be used to tell the compiler you mean the global version instead of the local version.

By convention, many developers prefix global variable names with “**g_**” to indicate that they are global. This both helps identify global variables as well as avoids naming conflicts with local variables.


## Internal and external variables

### Define internal/external linkage global variables

* Internal: `static`, `static const`, `const`
* External: No decoration, `extern const` (don't use `extern` soley to define external global variables because C++ will complain)


### Forward declaration for external linkage variables

Use `extern` keyword

## RULE: Avoid use of non-const global variables at all if possible!

There is few good reasons to use non-const global variables. Because otherwise, you will never know why a global variable is changed only until you searched all occurrences of that variables among files reference it. At some cases, `static` variables inside block(local) could get some benefit of global variables, see next section.

# Static duration

`static` has some different meanings depending on where it is used:

1. When applied to a variable outside of a block, it defines that variable is with internal linkage. It means this variable is only visible in the file where it is defined;
2. When applied to a variable inside a block, it allocates the memory in _.data_ section at the first time and keep it even if block exits. It means, the following calls to this **same** block, it will refer to the same memory allocated in the first call. (NOTE: if a same-named variable is defined in different blocks, then they are different memories!)

# Namespace

## Rule: Don’t use the “using” keyword in the global scope, including header files!

The using keyword follows normal scope rules (just like variables) -- if declared inside a block, it has block scope (and is only valid within that block). If declared outside a block, it has global scope, and affects the whole file from that point forward.

Using the using keyword judiciously can make your code neater and easier to read. Although the using keyword can be used outside of a function to help resolve every identifier in the file, this is highly discouraged, as it increases the chance of conflicting identifiers from multiple namespaces (and the global scope), which somewhat defeats the point of namespaces in the first place.

Note that putting a using statement outside of a block in a header means that it will end up in the global scope of every file that includes that header. This should definitely be avoided.

## Multiple namespace blocks with the same name allowed

It’s legal to declare namespace blocks in multiple locations (even multiple times in the same file, if you can find a good reason for doing so). All declarations within the namespace block are considered part of the namespace.

## Nested namespaces and namespace aliases

Namespaces can be nested inside other namespaces.

Because typing the fully qualified name of a variable or function inside a nested namespace can be painful, C++ allows you to create namespace aliases:

{% highlight CPP linenos %}
namespace Foo
{
    namespace Goo
    {
        const int g_x = 5;
    }
}
 
namespace Boo = Foo::Goo; // Boo now refers to Foo::Goo
 
int main()
{
    std::cout << Boo::g_x; // This is really Foo::Goo::g_x
    return 0;
}
{% endhighlight%}

It’s worth noting that namespaces in C++ were not designed as a way to implement an information hierarchy -- they were designed primarily as a mechanism for preventing naming collisions. As evidence of this, note that the entirety of the standard template library lives under the singular namespace std::. Some newer languages (such as C#) differ from C++ in this regard.

In general, you should avoid nesting namespaces if possible, and there’s few good reasons to nest them more than 2 levels deep.

# Type conversion

There are two basic types of type conversion: 

1. implicit type conversion, where the compiler automatically transforms one fundamental data type into another;
2. explicit type conversions, where the developer uses a casting operator to direct the conversion.

## Implicit type conversion

### Numeric promotion

Whenever a value from one type is converted into a value of a larger similar data type, this is called a __numeric promotion__.  Inlcudes:

* __Integral promotion__ involves the conversion of integer types narrower than int (which includes bool, char, unsigned char, signed char, unsigned short, signed short) to an integer (if possible) or an unsigned int.
* __Floating point promotion__ involves the conversion of a float to a double.

The important thing to remember about promotions is that they are always safe, and no data loss will result.

### Numeric conversion

When we convert a value from a larger type to a similar smaller type, or between different types, this is called a __numeric conversion__.

Unlike promotions, which are always safe, conversions may or may not result in a loss of data. Because of this, any code that does an implicit conversion will often cause the compiler to issue a warning.

### Evaluating arithmetic expressions

When evaluating expressions, the compiler breaks each expression down into individual subexpressions. The arithmetic operators require their operands to be of the same type. If operands of mixed types are used, the compiler will implicitly convert one operand to agree with the other using a process called __usual arithmetic conversion__. To do this, it uses the following rules:

* If the operand is an integer, it undergoes integral promotion (as described above).
* If the operands still do not match, then the compiler finds the highest priority operand and converts the other operand to match.

The priority of operands is as follows:

* long double (highest)
* double
* float
* unsigned long long
* long long
* unsigned long
* long
* unsigned int
* int (lowest)

## Explicit type conversion (casting)

In C++, there are 5 different types of casts: __C-style casts__, __static casts__, __const casts__, __dynamic casts__, and __reinterpret casts__.

__const casts__ and __reinterpret casts__ should generally be avoided because they are only useful in rare cases and can be harmful if used incorrectly.

### C-style cast

In standard C programming, casts are done via the () operator, with the name of the type to cast to inside. For example:

{% highlight CPP linenos %}
int i1 = 10;
int i2 = 3;
float f = (float)i1 / i2;
{% endhighlight%}

C++ will also let you use a C-style cast with a more function-call like syntax:

{% highlight CPP linenos %}
int i1 = 10;
int i2 = 3;
float f = float(i1) / i2;
{% endhighlight%}

Because C-style casts are not checked by the compiler at compile time, C-style casts can be inherently misused, because they will let you do things that may not make sense, such as _getting rid of a const_ or _changing a data type without changing the underlying representation_ (leading to garbage results).

Consequently, C-style casts should generally **be avoided**.

### static_cast

Static_cast is best used to convert one fundamental type into another.

{% highlight CPP linenos %}
int i1 = 10;
int i2 = 3;
float f = static_cast<float>(i1) / i2;
{% endhighlight%}

The main advantage of static\_cast is that it provides compile-time type checking, making it harder to make an inadvertent error. Static\_cast is also (intentionally) less powerful than C-style casts, so you can’t inadvertently remove const or do other things you may not have intended to do.

# std::string

Because strings are commonly used in programs, most modern languages include a built-in string data type. C++ includes one, not as part of the core language, but as part of the standard library.

## String input and output

String input should use `std::getline()` against `std::cin >>` since operator `>>` only returns characters up to the first whitespace it encounters.

Don't mix `std::getline()` and `std::cin >>` together, it will cause problems:

{% highlight CPP linenos %}
#include <iostream>    
#include <string>      
int main()
{
    using namespace std;
        
    string i;          
    cin >> i;          
    string s;
    getline(cin, s);   
    cout << "i = " << i << endl;
    cout << "s = " << s << endl;
}
{% endhighlight %}

You will get:

    > ./a.out
    abc
    i = abc
    s = 

This is because `>>` get "abc\n", it will extract "abc" to the variable "i" and leave the "\n" in the input stream. Then when `getline` is called, it sees "\n" is already in the input stream and figures we must have entered empty string.

A good rule of thumb is that after reading a string with std::cin, remove the newline from the stream. This can be done using the following:

    std::cin.ignore(32767, '\n');

# Enumerated Type

## Naming enums

* Enum identifiers named starting with a capital letter;
* Enumerators (enum member) are named using all caps.

cause enumerators are placed into the same namespace as the enumeration, an enumerator name can’t be used in multiple enumerations within the same namespace.

Consequently, it’s common to prefix enumerators with a standard prefix.

## Scopred Enums

In classic enumerated type, it is allowed to compare two different enumerators defined in different enumerations since they are placed into the same namespace as the enumeration. It will sometimes cause no sense comparision since they belongs to different enumerations.

C++11 defines a new concept, the __scoped enumeration__, which makes enumerations both strongly typed and strongly scoped. To make an enum class, we use the keyword __class__ after the enum keyword. Here’s an example:

{% highlight CPP linenos %}

#include <iostream>
int main()
{
    enum class Color // "enum class" defines this as an scoped enumeration instead of a standard enumeration
    {
        RED, // RED is inside the scope of Color
        BLUE
    };
 
    enum class Fruit
    {
        BANANA, // BANANA is inside the scope of Fruit
        APPLE
    };
 
    Color color = Color::RED; // note: RED is not directly accessible any more, we have to use Color::RED
    Fruit fruit = Fruit::BANANA; // note: BANANA is not directly accessible any more, we have to use Fruit::BANANA
    
    if (color == fruit) // compile error here, as the compiler doesn't know how to compare different types Color and Fruit
        std::cout << "color and fruit are equal\n";
    else
        std::cout << "color and fruit are not equal\n";
 
    return 0;
}
{% endhighlight %}

NOTE:

1. The strong typing rules means that each enum class is considered a unique type. This means that the compiler will not implicitly compare enumerators from __different__ enumerations. If you try to do so, the compiler will throw an error, as shown in the example above

2. You can still compare enumerators from within the __same__ enum class (since they are of the same type)

3. With enum classes, the compiler will no longer implicitly convert enumerator values to integers (if you try to print it, compiler will raise errors). If you have to,  you can explicitly convert an enum class enumerator to an integer by using a `static_cast` to int


# Structure

## initializing structres

Initializing structs by assigning values member by member is a little cumbersome, so C++ supports a faster way to initialize structs using an __initializer list__. This allows you to initialize some or all the members of a struct at declaration time:

{% highlight CPP linenos %}
struct Employee
{
    short id;
    int age;
    double wage;
};
 
Employee joe = { 1, 32, 60000.0 }; // joe.id = 1, joe.age = 32, joe.wage = 60000.0
Employee frank = { 2, 28 }; // frank.id = 2, frank.age = 28, frank.wage = 0.0 (default initialization)
{% endhighlight %}

In C++11, we can also use __uniform initialization__:

{% highlight CPP linenos %}
Employee joe { 1, 32, 60000.0 }; // joe.id = 1, joe.age = 32, joe.wage = 60000.0
Employee frank { 2, 28 }; // frank.id = 2, frank.age = 28, frank.wage = 0.0 (default initialization)
{% endhighlight %}

If the initializer list does not contain an initializer for some elements, those elements are initialized to a default value (that generally corresponds to the zero state for that type). In the above example, we see that frank.wage gets default initialized to 0.0 because we did not specify an explicit initialization value for it.

Starting with C++11, it’s possible to give non-static struct members a default value:

{% highlight CPP linenos %}
struct Triangle
{
    double length = 1.0;
    double width = 1.0;
};
 
int main()
{
    Triangle x; // length = 1.0, width = 1.0
    x.length = 2.0;
 
    return 0;
}
{% endhighlight %}

Unfortunately, in C++11, the non-static member initialization syntax is incompatible with the __initializer list__ and __uniform initialization__ syntax, so you’ll have to decide which one to use. For structs, we recommend the uniform initialization syntax.

In C++14, this restriction was lifted and both can be used. If both are provided, the __initializer list__/__uniform initialization__ syntax takes precedence.

# Auto Keyword

## Prior to C++11

Prior to C++11, the auto keyword was probably the least used keyword in C++, it was a way to explicitly specify that a variable should have automatic duration. However, since all variables in modern C++ default to automatic duration unless otherwise specified, the auto keyword was entirely obsolete.

## Automatic type deduction in C++11

Starting with C++11, when initializing a variable, the auto keyword can be used in place of the variable type to tell the compiler to infer the variable’s type from the assignment’s type. This is called __automatic type deduction__.

For example:

{% highlight CPP linenos %}
auto d = 5.0; // 5.0 is a double literal, so d will be type double
auto i = 1 + 2; // 1 + 2 evaluates to an integer, so i will be type int
{% endhighlight %}

Note that this only works when initializing a variable upon creation. Variables created without initialization values can not use this feature (as C++ has no context from which to deduce the type).

## Automatic type deduction for functions in C++14

In C++14, the auto keyword was extended to be able to auto-deduce a function’s return type. Consider:

{% highlight CPP linenos %}
auto add(int x, int y)
{
    return x + y;
}
{% endhighlight %}

Since x + y evaluates to an integer, the compiler will deduce this function should have a return type of int.

While this may seem neat, we recommend that this syntax be avoided for functions that return a fixed type. The return type of a function is of great use in helping to document a function. When a specific type isn’t specified, the user may not know what is expected.

## Trailing return type syntax in C++11

C++11 also added the ability to use a __trailing return syntax__, where the _return type is specified after the rest of the function prototype_.

Consider the following function declaration:

{% highlight CPP linenos %}
int add(int x, int y);
{% endhighlight %}

In C++11, this could be equivalently written as:

{% highlight CPP linenos %}
auto add(int x, int y) -> int;
{% endhighlight %}

In this case, auto does not perform automatic type deduction -- it is just part of the syntax to use a trailing return type.

### Why would you want to use this?

One nice thing is that it makes all of your function names line up:

{% highlight CPP linenos %}
auto add(int x, int y) -> int;
auto divide(double x, double y) -> double;
void printSomething();
auto calculateThis(int x, double d) -> std::string;
{% endhighlight %}

But it is of more use when combined with some advanced C++ features, such as classes and the decltype keyword.


