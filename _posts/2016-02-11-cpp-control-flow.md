---
layout: post
title: "C++ Control Flow"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Switch Statement

## case label

Case label is declared using the `case` keyword, and followed by a __constant expression__. A __constant expression__ is one that evaluates to a __constant value__ -- in other words, either a _literal_ (such as 5), an _enum_ (such as COLOR\_RED), or a _constant variable_ (such as x, when x has been defined as a const int).

## Multiple statements inside a switch block

One other bit of weirdness about switches is that you can have multiple statements underneath each case without defining a new block.

## Variable declaration and initialization inside case statements

You can declare, but not initialize, variables inside the case statements:

{% highlight CPP linenos %}
switch (x)
{
    case 1:
        int y; // okay, declaration is allowed
        y = 4; // okay, this is an assignment
        break;
 
    case 2:
        y = 5; // okay, y was declared above, so we can use it here too
        break;
 
    case 3:
        int z = 4; // illegal, you can't initialize new variables in the case statements
        break;
 
    default:
        std::cout << "default case" << std::endl;
        break;
}
{% endhighlight %}

Note that although variable y was defined in case 1, it was used in case 2 as well. _All cases are considered part of the same scope_, so a declaration in one case can be used in subsequent cases. However, initialization of variables directly underneath a case label is __disallowed__, and will cause a compile error.

If a case needs to define and/or initialize a new variable, best practice is to do so inside a block underneath the case statement:

{% highlight CPP linenos %}
switch (1)
{
    case 1:
    { // note addition of block here
        int x = 4; // okay, variables can be initialized inside a block inside a case
        std::cout << x;
        break;
    }
    default:
        std::cout << "default case" << std::endl;
        break;
}
{% endhighlight %}

# While Loop

## Rule: Always use signed integers for your loop variables

It is best practice to use signed integers for loop variables. Using unsigned integers can lead to unexpected issues. Consider the following code:

{% highlight CPP linenos %}
#include <iostream>
 
int main()
{
    unsigned int count = 10;
 
    // count from 10 down to 0
    while (count >= 0)
    {
        if (count == 0)
            std::cout << "blastoff!";
        else
            std::cout << count << " ";
        --count;
    }
 
    return 0;
}
{% endhighlight %}

# For Loop

## Scope

In older versions of C++, variables defined as part of the init-statement did not get destroyed at the end of the loop. This meant that you could have something like this:

{% highlight CPP linenos %}
for (int count=0; count < 10; ++count)
    std::cout << count << " ";
 
// count is not destroyed in older compilers
 
std::cout << "\n";
std::cout << "I counted to: " << count << "\n"; // so you can still use it here
{% endhighlight %}

This use has been disallowed, but you may still see it in older code.

Now the equivalant conversion is like:

From:

{% highlight CPP linenos %}
for (init-statement; condition-expression; end-expression)
   statement;
{% endhighlight %}

To (without `continue` statement):

{% highlight CPP linenos %}
{ // note the block here
    init-statement;
    while (condition-expression)
    {
        statement;
        end-expression;
    }
} // variables defined inside the loop go out of scope here
{% endhighlight %}

Here you can see, the `init-statement` is inside the block scope of __for loop__. Any variables declared there will be destroyed outside the __for__.
