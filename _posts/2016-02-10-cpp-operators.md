---
layout: post
title: "C++ Operators"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---
* Do not remove this line (it will not be displayed)
{: toc}

# Arithmetic operators

## Integer division and modulus with negative numbers prior to C++11

Prior to C++11, if either of the operands of integer division are negative, the compiler is free to round up or down! For example, -5 / 2 can evaluate to either -3 or -2, depending on which way the compiler rounds. However, most modern compilers truncate towards 0 (so -5 / 2 would equal -2). The C++11 specification changed this to explicitly define that integer division should always truncate towards 0 (or put more simply, the fractional component is dropped).

Also prior to C++11, if either operand of the modulus operator is negative, the results of the modulus can be either negative or positive! For example, -5 % 2 can evaluate to either 1 or -1. The C++11 specification tightens this up so that `a % b` always resolves to the sign of `a`.

# Increment and decrement oeprators

## Rule: Don’t use a variable that has a side effect applied to it more than once in a given statement.

Take an example:

{% highlight CPP linenos %}
int add(int x, int y)
{
    return x + y;
}

int main()
{
    int x = 5;
    int value = add(x, ++x):
}
{% endhighlight %}

C++ does not define the order in which function arguments are evaluated. If the left argument is evaluated first, this becomes a call to add(5, 6), which equals 11. If the right argument is evaluated first, this becomes a call to add(6, 6), which equals 12! Note that this is only a problem because one of the argument to function add() has a side effect.

There are other cases where C++ does not specify the order in which certain things are evaluated, so different compilers will make different assumptions. Even when C++ does make it clear how things should be evaluated, some compilers implement behaviors involving variables with side-effects incorrectly. These problems can generally all be avoided by ensuring that any any variable that has a side-effect applied is used no more than once in a given statement.


# Conditional operators (?:)

## The conditional operator evaluates as an expression

It's worth noting that the conditional operator evaluates as an _expression_, whereas if/else evaluates as _statement_. This means the conditional operator can be used in some places where if/else can not.

For example, when initializing a _const_ variable:

{% highlight C linenos %}

bool inBigClassroom = false;
const int classSize = inBigClassroom ? 30 : 20;

{% endhighlight %}

There is no satisfactory if/else statement for this, since const variables must be initialized when defined, and theinitializer can't be a statement.

# Ralational operators comparisons

## Comparison of floating point values

**Version 1**

{% highlight C linenos %}
#include <cmath>
bool isAlmostEqual(double a, double b, double epsilon)
{
    return fabs(a-b) <= epsilon;
}
{% endhighlight %}

Limitation: While this works, it’s not great. An epsilon of 0.00001 is good for inputs around 1.0, too big for numbers around 0.0000001, and too small for numbers like 10,000. This means every time we call this function, we have to pick an epsilon that’s appropriate for our inputs. If we know we’re going to have to scale epsilon in proportion to our inputs, we might as well modify the function to do that for us.

**Version 2**

{% highlight C linenos %}
#include <cmath>
bool approximatelyEqual(double a, double b, double epsilon)
{
    return fabs(a-b) <= ( (fabs(a) < fabs(b)? fabs(b) : fabs(a)) * epsilon);
}
{% endhighlight %}

Note that while the approximatelyEqual() function will work for many cases, it is not perfect, especially as the numbers approach zero:

{% highlight C linenos %}
#include <iostream>

int main()
{
    double a = 0.1 + 0.1+ 0.1 + 0.1+ 0.1 + 0.1 + 0.1 + 0.1 + 0.1 + 0.1;

    std::cout << approximatelyEqual(a-1.0, 0.0, 1e-8); // output 0!
}
{% endhighlight %}

**Version 3**

One way to avoid above case is to use both _absolute epsilon_ and _relative epsilon_:

{% highlight C linenos %}
#include <cmath>
bool approximatelyEqual(double a, double b, double absEpsilon, double relEpsilon)
{
    double diff = fabs(a - b);
    if (diff <= absEpsilon)
        return true;
    return diff <= ( (fabs(a) < fabs(b)? fabs(b) : fabs(a)) * relEpsilon);
}
{% endhighlight %}

# Logical Operators

## Logical exclusive or (XOR)

C++ doesn’t provide a logical XOR operator. Unlike logical OR or logical AND, XOR cannot be short circuit evaluated. Because of this, making an XOR operator out of logical OR and logical AND operators is challenging. However, you can easily mimic logical XOR using the not equals operator (!=):

    if (a != b) ... // a XOR b, assuming a and b are booleans

This can be extended to multiple operands as follows:

    if (a != b != c != d) ...

# Bitwise Operators

## Rule: When dealing with bit operators, use unsigned integers

Bit manipulation is one of the few cases where you should unambiguously use __unsigned__ integer data types. This is because C++ does not guarantee how signed integers are stored, nor how some bitwise operators apply to signed variables.

## Bit Flags

### Original way

Given:

    const unsigned int option_4 = 0x08;
    const unsigned int option_5 = 0x10;

Query bit states, use bitwise AND:

    if (myflags & option4 )... // if option4 is set, do something

Turn bits on, use bitwise OR:

    myflags |= option4 | option5; // turn option 4 and 5 on

Turn bits off, we use bitwise AND with an inverse bit pattern:
    
    myflags &= ~option4 & ~option5; // turn option 4 and 5 off

Toggle bit states, use bitwise XOR:

    myflags ^= option4 | option5; // flip option 4 and 5


### C++ way

The C++ standard library comes with functionality called __std::bitset__ that helps us manage bit flags.

To create a std::bitset, you need to include the bitset header, and then define a std::bitset variable indicating how many bits are needed. The number of bits must be a compile time constant.

{% highlight CPP linenos %}
#include <bitset>

std::bitset<8> bits;
{% endhighlight%}

std::bitset provides 4 key functions:

* test() allows us to query whether a bit is a 0 or 1
* set() allows us to turn a bit on (this will do nothing if the bit is already on)
* reset() allows us to turn a bit off (this will do nothing if the bit is already off)
* flip() allows us to flip a bit from a 0 to a 1 or vice versa
