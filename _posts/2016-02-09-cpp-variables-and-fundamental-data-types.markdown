---
layout: post
title: "C++ Variables and Fundamental Data Types"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Fundamental variable definition, initialization and assignment

## RULE: Favor implicit initialization over explicit initialization

When a variable is defined, you can immediately give that variable a value. This is called __initialization__. 

When we assign values to a defined variable using assignment operator, it's called an __explicit initialization__:

    int nValue = 5; // explicit init

When a variable is defined, you can also assign a value using an __implicit initialization__:

    int nValue(5); // implicit initialization

implicit initialization can perform better than explicit initialization for some data types, and comes with some other benefits regarding to classes. It also helps differentiate initialization from assignment. Consequently, we recommend its use over explicit initialization.

## RULE: If you're using a C++11 compatible compiler, favor uniform initialization

In an attempt to provide a single initialization mechanism that will work with all data types, C++11 adds a new form of initialization called __uniform initialization__:

    int value{5};

Uniform initialization has the added benefit of disallowing “narrowing” type conversions. This means that if you try to use uniform initialization to initialize a variable with a value it can not safely hold, the compiler will throw an warning or error. For example:

    int value{4.5}; // error

## RULE: Avoid defining multiple variables on a single line if initialize any of them

Because defining multiple variables on a single line AND initializing them is a recipe for mistakes, we recommend that you only define multiple variables on a line if you’re not initializing any of them:

    int a, b = 5; // a is uninitialized!

    int c, d(5);  // also not recommended

    int e, f;
    e = 1;
    f = 2;

    int g(3);
    int h(4);

## RULE: Define variables as close to their first use as possible

C++ compilers (neither later C) do not require all variables to be defined at the top of a function. The proper C++ style is to define variables as close to the first use of that variable as possible:

    int main()
    {
        using namespace std;

        cout << "Enter a number: ";
        int x;
        cint >> x;

        cout << "Enter another number: ";
        int y;
        cin >> y;

        /* Do something */

        return 0;
    }

This has quite a few advantages.

1. variables that are defined only when needed are given context by the statements around them. If x were defined at the top of the function, we would have no idea what it was used for until we scanned the function and found where it was used. Defining x amongst a bunch of input/output statements helps make it obvious that this variable is being used for input and/or output.
2. defining a variable only where it is needed tells us that this variable does not affect anything above it, making our program easier to understand and requiring less scrolling.
3. it reduces the likelihood of inadvertently leaving a variable uninitialized, because we can define and then immediately initialize it with the value we want it to have.

# Integers

## RULE: Favor signed integers over unsigned integers

All integer variables except _char_ are signed by default. _char_ can be either signed or unsigned by defualt(but is usually signed for conformity).

Generally, the _signed_ keyword is not used, except on _chars_.

Best practice is to avoid use of __unsigned__ integers unless you have a specific need for them(e.g. bitwise flag), as __unsigned__ integers are more prone to unexpected bugs and behaviors than signed integers.

## RULE: Do not depend on the resuts of overflow in your program

Overflow results in information being lost, which is almost never desirable. If there is any suspicion that a variable might need to store a value that falls outside its range, __use a larger variable!__

Also note that the results of overflow are only predictable for __unsigned integers__. Overflowing __signed integers__ or __non-integers__ (e.g. floating point numbers) may result in different results on different systems.

# Fixed width integers

To help with cross-platform portability, C99 defined a set of fixed-width integers (in the `stdint.h` header) that are guaranteed to have the same size on any architecture. These are defined as follows:

|---
|Name|Type|
|:-|:-|
|int8_t|1 byte signed|
|---
|uint_8_t| 1 byte unsigned|
|---
|int16_t| 2 byte signed|
|---
|uint_16_t| 2 byte unsigned|
|---
|int32_t| 4 byte signed|
|---
|uint32_t| 4 byte unsigned|
|---
|int64_t| 8 byte signed|
|---
|uint64_t| 8 byte unsigned|
|===

C++ officially adopted these fixed-width integers as part of C++11. They can be accessed by including the cstdint heade: `#include <cstdint>` (note: no ".h" here)

# Floating point numbers

## Float point literal

By default, floating point literals default to type double. An __f__ suffix is used to denote a literal of type float.

## RULE: Favor double over float unless space is at a premium, as the lack of precision in a float will often lead to challenges

In scientific notation: The digits in the significand (the part before the E) are called the __significant digits__. The number of significant digits defines a number’s __precision__. The more digits in the significand, the more precise a number is

Floating point numbers can only store a certain number of significant digits, and the rest are lost. The __precision__ of a floating point number defines how many significant digits it can represent _without information loss._

When outputting floating point numbers, `cout` has a default __precision__ of __6__ -- that is, it assumes all variables are only significant to 6 digits, and hence it will truncate anything after that. For example:

    #include <iostream>

    int main(){   
        using namespace std;
        double a(1234567.0); // This will output: `1.23457e+06`, only 6 precision
              
        cout << a << endl;
    }         

However, we can override the default precision that __cout__ shows by using the `std::setprecision()` function that is defined in a header file called `iomanip`:

    #include <iostream>
    #include <iomanip> // for std::setprecision()
    int main()
    {
        std::cout << std::setprecision(16); // show 16 digits
        float f = 3.33333333333333333333333333333333333333f;
        std::cout << f << std::endl;
        double d = 3.3333333333333333333333333333333333333;
        std::cout << d << std::endl;
        return 0;
    }

Outputs:

    3.333333253860474
    3.333333333333334

Because we set the precision to 16 digits, each of the above numbers has 16 digits. But, as you can see, the numbers certainly aren’t precise to 16 digits!

The number of digits of precision a value has depends on both the size (floats have less precision than doubles) and the particular value being stored (some values have more precision than others). 

* Float values have between 6 and 9 digits of precision, with most float values having __at least 7__ significant digits (which is why everything after that many digits in our answer above is junk)
* Double values have between 15 and 18 digits of precision, with most double values
having __at least 16__ significant digits
* Long double has a minimum precision of 15, 18, or 33 significant digits depending on how many bytes it occupies.


## Rounding errors

One of the reasons floating point numbers can be tricky is due to non-obvious differences between binary (how data is stored) and decimal (how we think) numbers. Consider the fraction 1/10. In decimal, this is easily represented as 0.1, and we are used to thinking of 0.1 as an easily representable number. However, in binary, 0.1 is represented by the infinite sequence: 0.00011001100110011… Because of this, when we assign 0.1 to a floating point number, we’ll run into precision
problems. (Here we can see: _floating point numbers often have small rounding errors, even when the number has fewer significant digits than the precision._) 

For example:

    #include <iostream>       
    #include <iomanip>        
                              
    int main(){               
        using namespace std;  
        float a(0.1f);        
                              
        cout << a << endl;    
        cout << setprecision(9);
        cout << a << endl;    
    }           

Output:

    0.1
    0.100000001

On the bottom line, where we have cout show us 9 digits of precision, we see that d is actually not quite 0.1! This is because the float had to truncate the approximation due to its limited memory, which resulted in a number that is not exactly 0.1. This is called a __rounding error__

Take note that mathematical operations (such as addition and multiplication) tend to make rounding errors grow. So even though 0.1 has a rounding error in the __9th__ significant digit, when we add 0.1 ten times, the rounding error has crept into the __8th__ significant digit.

# Chars

## Printing chars

When using cout to print a char, cout outputs the char variable as an ASCII character instead of a number.

Note: The fixed width integer __int8_t__ is _usually_ treated the same as a signed char in C++, so it will _generally_ print as a char instead of an integer.

If we want to output a char as a number instead of a character, we have to tell cout to print the char as if it were an integer. To convert between fundamental data types (for example, from a char to an int, or vice versa), we use a type cast called a __static cast__: `static_cast<new_type>(expression)`

## Newline(\n) vs. std::endl

When std::cout is used for output, the output may be buffered. Both ‘\n’ and std::endl will move the cursor to the next line. _In addition, std::endl will also ensure that any queued output is actually output before continuing_.

So when should you use ‘\n’ vs std::endl? The short answer is:

* Use std::endl when you need to ensure your output is output immediately (e.g. when writing a record to a file, or when updating a progress bar). Note that this may have a performance cost, particularly if writing to the output device is slow (e.g. when writing a file to a disk).
* Use ‘\n’ in other cases.

## Other char types, wchar_t, char16_t, and char32_t

* __wchar_t__ should be avoided in almost all cases (except when interfacing with the Windows API). Its size is implementation defined, and is not reliable. It has largely been deprecated.
* **char16_t** and **char32_t** were added to C++11 to provide explicit support for 16-bit and 32-bit Unicode characters (8-bit Unicode characters are already supported by the normal char type). You won’t need to use **char16_t** or **char32_t** unless you’re planning on making your program Unicode compatible and you are using 16-bit or 32-bit Unicode characters. 


# Constant

C++ has two kinds of constants: literal, and symbolic.

* Literal constants (usually just called “literals”) are literal numbers inserted into the code.  Like: `bool isOn = true;` or `int x = 5;`
* Symbolic constants contains two favors: 

    * object-like macros
    * `const`/`constexpr` decorated variables

## `const`/`constexpr` symbolic constant

Const variables _must_ be initialized when defined. Defining a const variable without initializing it will also cause a compile error.

C++ actually has two different kinds of constants:

* Runtime constants are those whose initialization values can only be resolved at runtime. E.g.
`const int userAge(age); // where age is a variable defined above`
* Compile-time constants are those whose initialization values can be resolved at compile-time. E.g. `const int userAge(25);`

However, there are a few odd cases where C++ requires a compile-time constant instead of a run-time constant (such as when defining the length of a fixed-size array). Because a const value could be either runtime or compile-time, the compiler has to keep track of which kind of constant it is.

To help disambiguate this situation, C++11 introduced new keyword __constexpr__, which ensures that the constant must be a compile-time constant.

## Avoid using object-like macro to create symbolic constant, use `const` variables instead

There are several reasons:

* First, because macros are resolved by the preprocessor, which replaces the symbolic name with the defined value, `#defined` symbolic constants do not show up in the debugger (which shows you your actual code). So although the compiler would compile `int max_students = numClassrooms * 30;`, in the debugger you’d see `int max_students = numClassrooms * MAX_STUDENTS_PER_CLASS;`. You’d have to go find the definition of `MAX_STUDENTS_PER_CLASS` in order to know what the actual value was. This can make your programs harder to debug
* Second, `#defined` values always have global scope . This means a value `#defined` in one piece of code may have a naming conflict with a value `#defined` with the same name in another piece of code.

## Using symbolic constant variables throughout a program

The easiest way is:

1. Create a header file to hold these constants declarations, and a source file to hold these constants definitions;
2. Inside this header file, declare a namespace with variables of `extern const`. Inside this source file, define the namespace with variables of `extern const`;
3. Add all your constants inside the namespace (make sure they're constant);
4. `#include` the header file wherever you need it

E.g. 

constants.cpp:

{% highlight CPP linenos %}
namespace Constants
{
    // actual global variables
    extern const double pi(3.14159);
    extern const double avogadro(6.0221413e23);
    extern const double my_gravity(9.2); // m/s^2 -- gravity is light on this planet
}
{% endhighlight%}

constants.h

{% highlight CPP linenos %}
#ifndef CONSTANTS_H
#define CONSTANTS_H
 
namespace Constants
{
    // forward declarations only
    extern const double pi;
    extern const double avogadro;
    extern const double my_gravity;
}
 
#endif
{% endhighlight%}

Use the scope resolution operator (::) to access your constants in .cpp files:

{% highlight CPP linenos %}
#include "constants.h"
double circumference = 2 * radius * Constants::pi;
{% endhighlight%}

Now the symbolic constants will get instantiated only once (in constants.cpp), instead of once every time constants.h is #included, and the other uses will simply reference the version in constants.cpp.




