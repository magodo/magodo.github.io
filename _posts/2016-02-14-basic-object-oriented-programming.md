---
layout: post
title: "C++ Basic object-oriented programming"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* Do not remove this line (it will not be displayed)
{: toc}

# Class and class members

## Instantiate

{% highlight CPP linenos %}
class DateClass
{
public:
    int m_month;
    int m_day;
    int m_year;
};

DateClass today { 10, 14, 2020 }; // declare a variable of class DateClass
{% endhighlight %}

## class/member naming conventions

* Name your classes starting with a Capital letter
* Using the “m_” prefix for member variables helps distinguish member variables from function parameters or local variables inside member functions.

## struct could also hold functions, but don't be tempted to use it!

In C, structs can only hold data, and do not have associated member functions. In C++, after designing classes (using the class keyword), Bjarne Stroustrup spent some amount of time considering whether structs (which were inherited from C) should be granted the same capabilities. Upon consideration, he determined that they should, in part to have a unified ruleset for both. So although we wrote the above programs using the class keyword, we could have used the struct keyword instead.

Many developers (including myself) feel this was the incorrect decision to be made, as it can lead to dangerous assumptions: For example, it’s fair to assume a class will clean up after itself, but it’s not safe to assume a struct will. Consequently, we recommend __using the struct keyword for data-only structures, and the class keyword for defining objects that that require both data and functions to be bundled together__.

## public vs private 

* __public member__ member of a struct or class that can be accessed from outside of the struct or class
* __private member__ member of a class that can only be accessed by other member of the class

By default, 

* all members of a __class__ are __private__
* all members of a __struct__ are __public__

Rule: Make member variables private, and member functions public, unless you have a good reason not to.

# Encapsulation

In object-oriented programming, Encapsulation (also called information hiding) is the process of keeping the details about how an object is implemented hidden away from users of the object. In C++, we implement encapsulation via access specifiers. Typically, all member variables of the class are made private (hiding the implementation details), and most member functions are made public (exposing an interface for the user).

__Benefit__

* encapsulated classes are easier to use and reduce the complexity of your programs
* encapsulated classes help protect your data and prevent misuse
* encapsulated classes are easier to change
* encapsulated classes are easier to debug

# Constructor

When all members of a class (or struct) are public, we can initialize the class (or struct) directly using an initialization list or uniform initialization (in C++11).

However, as soon as we make any member variables private, we’re no longer able to initialize classes in this way. It does make sense: if you can’t directly access a variable (because it’s private), you shouldn’t be able to directly initialize it. In this case, we need to use __constructor__.

A __constructor__ is a special kind of class member function that is automatically called when an object of that class is instantiated. It is typically used to initialize member variables of the class to appropriate default or user-provided values, or to do any setup steps necessary for the class to be used 

Unlike normal member functions, constructors have specific rules for how they must be named:

1. Constructors should always have the same name as the class (with the same capitalization)
2. Constructors have no return type (not even void)

## Default constructor

A constructor that takes no parameters (or has parameters that all have default values) is called a __default constructor__. The default constructor is called if no user-provided initialization values are provided.

## Constructors with parameters

While the default constructor is great for ensuring our classes are initialized with reasonable default values, often times we want instances of our class to have specific values. Fortunately, constructors can also be declared with parameters:

{% highlight CPP linenos %}
#include <cassert>
 
class Fraction
{
private:
    int m_numerator;
    int m_denominator;
 
public:
    Fraction() // default constructor
    {
         m_numerator = 0;
         m_denominator = 1;
    }
 
    // Constructor with parameters
    Fraction(int numerator, int denominator=1)
    {
        assert(denominator != 0);
        m_numerator = numerator;
        m_denominator = denominator;
    }
 
    int getNumerator() { return m_nNumerator; }
    int getDenominator() { return m_nDenominator; }
    double getValue() { return static_cast<double>(m_numerator) / m_denominator; }
};
{% endhighlight %}

NOTE:

1. These two constructors can coexist peacefully in the same class due to function overloading. In fact, you can define as many constructors as you want, so long as each has a unique __signature__ (number and type of parameters, return type).
2. We can give each parameter a __default__ value at _parameter list_ in method definition

### Method to use constructor

1. _Implicit initialization_

{% highlight CPP linenos %}
Fraction fiveThirds(5, 3); // calls Fraction(int, int) constructor
{% endhighlight %}

2. In C++11, we can also use _uniform initialization_

{% highlight CPP linenos %}
Fraction fiveThirds {5, 3}; // calls Fraction(int, int) constructor
{% endhighlight %}

Note that we have given the second parameter of the constructor with parameters a default value, so the following is also legal:

{% highlight CPP linenos %}
Fraction six(6); // calls Fraction(int, int) constructor
{% endhighlight %}

## Provide at least one constructor for your class, even if it’s an empty default constructor

If your class has no other constructors, C++ will automatically create an _empty_ default constructor for you. All member variables hold garbage.

However, if you do have other non-default constructors in your class, but no default constructor, C++ will not create an empty default constructor for you. In this case, the class will not be instantiatiable without parameters.

Generally speaking, it’s a good idea to always provide at least one constructor in your class. This will prevent C++ from creating an empty default constructor, ensuring that users don’t instantiate objects of your class that have uninitialized members.

## Constructor member initializer list

### why use this?

1. Some types of data(e.g. _const_ and _reference_ variables) must be initialized on the line they are declared:

{% highlight CPP linenos %}
class Something
{
private:
    const int m_value;
 
public:
    Something()
    {
        m_value = 5; // error: const vars can not be assigned to
    } 
};
{% endhighlight %}

This produces code similar to the following:

{% highlight CPP linenos %}
const int m_value; // error: const vars must be initialized with a value
m_value = 5; //  error: const vars can not be assigned to
{% endhighlight %}

2. Member initialization lists are required when doing composition and inheritance

### Introduction

Below code is using initialization list:

{% highlight CPP linenos %}
class Something
{
private:
    int m_value1;
    double m_value2;
    int *m_value3;
 
public:
    Something() : m_value1(0), m_value2(0.0), m_value3(0) // Implicitly initialize our member variables
    {
    // No need for assignment here
    }
};
{% endhighlight %}

The member initializer list is inserted after the constructor parameters. It begins with a colon (:), and then lists each variable to initialize along with the value for that variable(__implicit initialization__) separated by a comma. Note that we no longer need to do the assignments in the constructor body, since the initializer list replaces that functionality. Also note that the initializer list does not end in a semicolon.

Since C++11, instead of __implicit initialization__, you could use __uniform initialization__:

{% highlight CPP linenos %}
class Something
{
private:
    const int m_value;
 
public:
    Something(): m_value { 5 } // Uniformly initialize our member variables
    {
    } 
};
{% endhighlight %}

Rule: Favor uniform initialization over implicit initialization if you compiler is C++11 compatible.

### Initializing array members with member initialization list

Consider a class with an array member.

Prior to C++11, you can only zero an array member via a member initialization list:

{% highlight CPP linenos %}
class Something
{
private:
    const int m_array[5];
 
public:
    Something(): m_array {} // zero the member array
    {
        // If we want the array to have values, we'll have to use assignment here
    }
 
};
{% endhighlight %}

However, in C++11, you can fully initialize a member array using uniform initialization:

{% highlight CPP linenos %}
class Something
{
private:
    const int m_array[5];
 
public:
    Something(): m_array { 1, 2, 3, 4, 5 } // use uniform initialization to initialize our member array
    {
    }
 
};
{% endhighlight %}

## Non-static member initialization

__BIG NOTE__: This functionality is not implemented in gcc 4.6.4...

Non-static member initialization provides default values for your member variables that your constructors will use if the constructors do not provide an initialization values for the members themselves (via the member initialization list).

{% highlight CPP linenos %}
class Square
{
private:
    double m_length = 1.0; // m_length has a default value of 1.0
    double m_width = 1.0; // m_width has a default value of 1.0
 
public:
    Square()
    {
    // This constructor will use the default values above since they aren't overridden here
    }
 
    void print()
    {
        std::cout << "length: " << m_length << ", width: " << m_width << '\n';
    }
};
 
int main()
{
    Square x; // x.m_length = 1.0, x.m_width = 1.0
    x.print();
 
    return 0;
}
{% endhighlight %}

Note, if a default initialization value is provided and the constructor initializes the member via the member initializer list, the member initializer list will take precedence.

## overlapping and delegating constructor

When you instantiate a new object, the object’s constructor is called implicitly by the C++ compiler. It’s not uncommon to have a class with multiple constructors that have overlapping functionality. Consider the following class:

{% highlight CPP linenos %}
class Foo
{
public:
    Foo()
    {
        // code to do A
    }
 
    Foo(int value)
    {
        // code to do A
        // code to do B
    }
};
{% endhighlight %}

This class has two constructors: a default constructor, and a constructor that takes an integer. Because the “code to do A” portion of the constructor is required by both constructors, the code is duplicated in each constructor.

### Prior to C++11

The obvious solution would be to have the Foo(int) constructor call the Foo() constructor to do the A portion. However, with a pre-C++11 compiler, if you try to have one constructor call another constructor, it will often compile, but it will not work as you expect, and you will likely spend a long time trying to figure out why, even with a debugger. __It turns out, this actually creates a temporary object and then discards it__.

However, constructors are allowed to call non-constructor functions in the class. Just be careful that any members the non-constructor function uses have already been initialized:

{% highlight CPP linenos %}
class Foo
{
private:
    void DoA()
    {
        // code to do A
    }
 
public:
    Foo()
    {
        DoA();
    }
 
    Foo(int nValue)
    {
        DoA();
        // code to do B
    }
 
};
{% endhighlight %}

### Since C++11

__BIG NOTE__: This functionality is not implemented in gcc 4.6.4...

Starting with C++11, constructors are now allowed to call other constructors. This process is called __delegating constructors__ (or __constructor chaining__).

To have one constructor call another, simply call the constructor in the member initializer list. This is the one case where calling another constructor directly is acceptable. Applied to our example above:

{% highlight CPP linenos %}
class Foo
{
private:
 
public:
    Foo()
    {
        // code to do A
    }
 
    Foo(int value): Foo() // use Foo() default constructor to do A
    {
        // code to do B
    }
 
};
{% endhighlight %}

Here’s another example of using delegating constructor to reduce redundant code:

{% highlight CPP linenos %}
#include <string>
 
class Employee
{
private:
    int m_id;
    std::string m_name;
 
public:
    Employee(int id, std::string name):
        m_id(id), m_name(name)
    {
    }
 
    // All three of the following constructors use delegating constructors to minimize redundant code
    Employee() : Employee(0, "") { }
    Employee(int id) : Employee(id, "") { }
    Employee(std::string name) : Employee(0, name) { }
};
{% endhighlight %}


# Destructor

When an object goes out of scope normally, or a dynamically allocated object is explicitly deleted using the delete keyword, the class destructor is called (if it exists) to do any necessary clean up before the object is removed from memory.

For simple classes (those that just initialize the values of normal member variables), a destructor is not needed because C++ will automatically clean up the memory for you. However, if your class object is holding any resources (e.g. dynamic memory, or a file or database handle), or if you need to do any kind of maintenance before the object is destroyed, the destructor is the perfect place to do so, as it is typically the last thing to happen before the object is destroyed.

Like constructors, destructors have specific naming rules:
1. The destructor must have the same name as the class, preceded by a tilde (~).
2. The destructor can not take arguments.
3. The destructor has no return type.

(note that rule 2 implies that only one destructor may exist per class)

## A warnning about the exit() function

Note that if you use the exit() function, your program will terminate and no destructors will be called. Although the resources(e.g. heap, file) your process held will be released, however, be wary if you’re relying on your destructors to do necessary cleanup work (e.g. write something to a log file or database before exiting).

# "this" pointer

`*this` is just a function parameter, it doesn’t add any memory usage to your class (just to the member function call, since that parameter goes on the stack while the function is executing).

## "this" could be used to "chain" function calls (e.g. overload operator)

It can sometimes be useful to have a class member function return the object it was working with as a return value. The primary reason to do this is to allow a series of functions to be “chained” together, so several functions can be called on the same object! You’ve actually been doing this for a long time. Consider this common example where you’re outputting more than one bit of text using std::cout:

{% highlight CPP linenos %}
std::cout << "Hello, " << userName;
{% endhighlight %}

n this case, std::cout is an object, and operator<< is a member function that operates on that object. The compiler evaluates the above snippet like this:

{% highlight CPP linenos %}
(std::cout << "Hello, ") << userName;
{% endhighlight %}

For instance, we can use `*this` to chain function calls in example below:

{% highlight CPP linenos %}
class Calc
{
private:
    int m_value;
 
public:
    Calc() { m_value = 0; }
 
    Calc& add(int value) { m_value += value; return *this; }      // *this is a reference to the calling object 
    Calc& sub(int value) { m_value -= value; return *this; }
    Calc& mult(int value) { m_value *= value; return *this; }
 
    int getValue() { return m_value; }
};
{% endhighlight %}

Then we can call like this:

{% highlight CPP linenos %}
#include <iostream>
int main()
{
    Calc calc;
    calc.add(5).sub(3).mult(4);
 
    std::cout << calc.getValue() << '\n';
    return 0;
}
{% endhighlight %}

# Class definition and declaration organization

C++ provides a way to separate the declaration portion of the class from the implementation portion. This is done by defining the class member functions outside of the class declaration. To do so, simply define the member functions of the class as if they were normal functions, but prefix the class name to the function using the scope operator (::) (same as for a namespace).

Class declarations can be put in header files in order to facilitate reuse in multiple files or multiple projects. Traditionally, the class declaration is put in a header file of the same name as the class, and the member functions defined outside of the class are put in a .cpp file of the same name as the class.

## Default parameters

Default parameters for member functions should be declared in the class declaration (in the header file), where they can be seen by whomever #includes the header.

# const class objects and member functions

Fundamental data types (int, double, char, etc…) can be made const via the const keyword, and that all const variables must be initialized at time of creation.

In the case of const fundamental data types, initialization can be done through explicit, implicit, or uniform assignment:

{% highlight CPP linenos %}
const int value1 = 5; // explicit initialization
const int value2(7); // implicit initialization
const int value3 { 9 }; // uniform initialization (C++11)
{% endhighlight %}

## const class object

Similarly, instantiated class objects can also be made const by using the const keyword. Initialization is done via class constructors:

{% highlight CPP linenos %}
const Date date1; // initialize using default constructor
const Date date2(2020, 10, 16); // initialize using parameterized constructor
const Date date3 { 2020, 10, 16 }; // initialize using parameterized constructor (C++11)
{% endhighlight %}

Once a const class object has been initialized via constructor, any attempt to modify the member variables of the object is disallowed, as it would violate the __const-ness of the object__. This includes both: 

* changing member variables directly (if they are public)
* calling member functions that set the value of member variables.

## const member function

Along with the code above:

{% highlight CPP linenos %}
std::cout << something.getValue();
{% endhighlight %}

Perhaps surprisingly, this will also cause a compile error, even though getValue() doesn’t do anything to change a member variable! It turns out that __const class objects__ can only explicitly call __const member functions__, and getValue() has not been marked as a const member function. 

A __const member function__ is a member function that guarantees it will not _change any class variables_ or _call any non-const member functions_.

To make getValue() a const member function, we simply append the `const` keyword to the function prototype, after the parameter list, but before the function body:

{% highlight CPP linenos %}
class Something
{
public:
    int m_value;
 
    Something() { m_value= 0; }
 
    void resetValue() { m_value = 0; }
    void setValue(int value) { m_value = value; }
 
    int getValue() const { return m_value; } // note addition of const keyword after parameter list, but before function body
};
{% endhighlight %}

For member functions defined outside of the class declaration, the const keyword must be used on both the function prototype in the class declaration and on the function definition.

It’s worth noting that a const object is not required to initialize its member variables (that is, it’s legal for a const class object to call a constructor that initializes all, some, or none of the member variables)!

Rule: Make any member function that does not modify the state of the class object const

## Const references

Although instantiating const class objects is one way to create const objects, a more common way is by passing an object to a function by const reference.

Most of the time, we don’t need a copy, a reference to the original argument works just fine, and is more performant because it avoids the needless copy. We typically make the reference const in order to ensure the function does not inadvertently change the argument, and to allow the function to work with R-values (e.g. literals), which can be passed as const references, but not non-const references.

Take note, methods called by this const object reference should also be const. 

# static attribute in class

In C, `static` is used to:

1. When applied to a variable outside of a block, it defines that variable is with internal linkage. It means this    variable is only visible in the file where it is defined;
2. When applied to a variable inside a block, it allocates the memory in _.data_ section at the first time and keep   it even if block exits. It means, the following calls to this **same** block, it will refer to the same memory        allocated in the first call. (NOTE: if a same-named variable is defined in different blocks, then they are different  memories!)

C++ introduces two more uses for the static keyword when applied to classes: __static member variables__, and __static member functions__. 

## Static member variables

* Static member variable is shared between all objects of the class. 
* Static members are not associated with class objects: 

  Static members exist even if no objects of the class have been instantiated! Much like global variables, they are created when the program starts, and destroyed when the program ends.

### Defining and initializing static member variables

When we declare a static member variable inside a class, we’re simply telling the class that a static member variable exists (much like a forward declaration). Because static member variables are not part of the individual class objects (they get initialized when the program starts), you must explicitly define the static member outside of the class, in the global scope. E.g:

{% highlight CPP linenos %}
class Something
{
public:
    static int s_value; // declares the static member variable
};
 
int Something::s_value = 1; // defines the static member variable (we'll discuss this line below)
{% endhighlight %}

This line serves two purposes: 

1. declare/define the static member variable (just like a global variable)
2. optionally initializes it. In this case, we’re providing the initialization value 1. If no initializer is provided, C++ initializes the value to 0.

**NOTE** static member definition is not subject to access controls: you can define and initialize the value even if it’s declared as private (or protected) in the class.

If the class is defined in a .h file, the static member definition is usually placed in the associated code file for the class (e.g. Something.cpp). If the class is defined in a .cpp file, the static member definition is usually placed directly underneath the class. Do not put the static member definition in a header file (much like a global variable, if that header file gets included more than once, you’ll end up with multiple definitions, which will cause a compile error).

There is one exception where a static member definition line is not required: when the static member is of type __const integer__ or __const enum__. Those can be initialized directly on the line in which they are declared:

{% highlight CPP linenos %}
class Whatever
{
public:
    static const int s_value = 4; // a static const int can be declared and initialized directly
};
{% endhighlight %}

### merits

There are several senarios where static members variable are useful:

1. Recording times of some events happened on a class
2. No need to copy some data for each class object (e.g. an array used to store a set of pre-calc values)

## Static member functions

Static member functions can be used to work with static member variables in the class. An object of the class is not required to call them.

Note:

* static member functions are not attached to an object, they have no `this` pointer!
* static member functions can only access static member variables.
* when defining static function, don't need `static` decorator. 

## C++ does not support static constructors

f you can initialize normal member variables via a constructor, then by extension it makes sense that you should be able to initialize static member variables via a static constructor. And while some modern languages do support static constructors for precisely this purpose, C++ is unfortunately not one of them.

If your static variable can be directly initialized, no constructor is needed: you can initialize the static member variable at the point of definition (even if it is private). 

If initializing your static member variable requires executing code (e.g. a loop), there are many different, somewhat obtuse ways of doing this. The following code presents one of the better methods. However, it is a little tricky, and you’ll probably never need it:

{% highlight CPP linenos %}
class MyClass
{
private:
    static std::vector<char> s_mychars;
 
public:
 
    class _init // we're defining a nested class named _init
    {
    public:
        _init() // the _init constructor will initialize our static variable
        {
            s_mychars.push_back('a');
            s_mychars.push_back('e');
            s_mychars.push_back('i');
            s_mychars.push_back('o');
            s_mychars.push_back('u');
        }
    } ;
 
private:
    static _init s_initializer; // we'll use this static object to ensure the _init constructor is called
};
 
std::vector<char> MyClass::s_mychars; // define our static member variable
MyClass::_init MyClass::s_initializer; // define our static initializer, which will call the _init constructor, which will initialize s_mychars
{% endhighlight %}

When static member `s_initializer` is defined, the `_init()` default constructor will be called (because s_initializer is of type _init). We can use this constructor to initialize any static member variables. The nice thing about this solution is that all of the initialization code is kept hidden inside the original class with the static member.

# Friend functions and classes

## Friend functions

A __friend function__ of a class is a function that can access the private members of this class as though it were a member of this class. 

A friend function may be either 

* a normal function

  or

* a member function of another class
 
To declare a friend function, simply use the `friend` keyword in front of the prototype of the function you wish to be a friend of the class. It does not matter whether you declare the friend function in the private or public section of the class.

Here’s an example of using a friend function:

{% highlight CPP linenos %}
class Accumulator
{
private:
    int m_value;
public:
    Accumulator() { m_value = 0; } 
    void add(int value) { m_value += value; }
 
    // Make the reset() function a friend of this class
    friend void reset(Accumulator &accumulator);
};
 
// reset() is now a friend of the Accumulator class
void reset(Accumulator &accumulator)
{
    // And can access the private data of Accumulator objects
    accumulator.m_value = 0;
}
{% endhighlight %}

### friend function of multiple classes

A function can be a friend of more than one class at the same time. For example, consider the following example:

{% highlight CPP linenos %}
class Humidity;
 
class Temperature
{
private:
    int m_temp;
public:
    Temperature(int temp) { m_temp = temp; }
 
    setTemperature(int temp) { m_temp = temp; }
 
    friend void printWeather(const Temperature &temperature, const Humidity &humidity);
};
 
class Humidity
{
private:
    int m_humidity;
public:
    Humidity(int humidity) { m_humidity = humidity; }
 
    setHumidity(int humidity) { m_humidity = humidity; }
 
    friend void printWeather(const Temperature &temperature, const Humidity &humidity);
};
 
void printWeather(const Temperature &temperature, const Humidity &humidity)
{
    std::cout << "The temperature is " << temperature.m_temp <<
       " and the humidity is " << humidity.m_humidity << '\n';
}
 
int main()
{
    Humidity hum;
    Temperature temp;
 
    hum.setHumidity(10);
    temp.setTemperature(12);
 
    printWeather(temp, hum);
 
    return 0;
}
{% endhighlight %}

There are two things worth noting about this example. 

1. Because PrintWeather is a friend of both classes, it can access the private data from objects of both classes
2. Note the following line at the top of the example:

<b></b>

    class Humidity;

This is a __class prototype__ that _tells the compiler that we are going to define a class called Humidity in the future_. Without this line, the compiler would tell us it doesn’t know what a Humidity is when parsing the prototype for PrintWeather() inside the Temperature class. Class prototypes serve the same role as function prototypes -- they tell the compiler what something looks like so it can be used now and defined later. However, unlike functions, classes have no return types or parameters, so class prototypes are always simply class ClassName, where ClassName is the name of the class.

## Friend classes

It is also possible to make an entire class a friend of another class. This gives all of the members of the friend class access to the private members of the other class. Here is an example:

{% highlight CPP linenos %}
class Storage
{
private:
    int m_nValue;
    double m_dValue;
public:
    Storage(int nValue, double dValue)
    {
        m_nValue = nValue;
        m_dValue = dValue;
    }
 
    // Make the Display class a friend of Storage
    friend class Display;
};
 
class Display
{
private:
    bool m_displayIntFirst;
 
public:
    Display(bool displayIntFirst) { m_displayIntFirst = displayIntFirst; }
 
    void displayItem(Storage &storage)
    {
        if (m_displayIntFirst)
            std::cout << storage.m_nValue << " " << storage.m_dValue << '\n';
        else // display double first
            std::cout << storage.m_dValue << " " << storage.m_nValue << '\n';
    }
};
 
int main()
{
    Storage storage(5, 6.7);
    Display display(false);
 
    display.displayItem(storage);
 
    return 0;
}
{% endhighlight %}

Note: if class A is a friend of B, and B is a friend of C, that does not mean A is a friend of C.

Be careful when using friend functions and classes, because it allows the friend function or class to violate encapsulation. If the details of the class change, the details of the friend will also be forced to change. Consequently, limit your use of friend functions and classes to a minimum.

## Friend member functions

Instead of making an entire class a friend, you can make a single member function a friend. This is done similarly to making a normal function a friend, except:

1. Using the name of the member function with the `className::` prefix included (e.g. Display::displayItem)
2. Ensure there is the full declaration of the class of the member function before this member function is defined as a friend function in the other classes.

For example:

{% highlight CPP linenos %}
class Storage; // forward declaration for class Storage
 
class Display
{
private:
    bool m_displayIntFirst;
 
public:
    Display(bool displayIntFirst) { m_displayIntFirst = displayIntFirst; }
    
    void displayItem(Storage &storage); // forward declaration above needed for this declaration line
};
 
class Storage
{
private:
    int m_nValue;
    double m_dValue;
public:
    Storage(int nValue, double dValue)
    {
        m_nValue = nValue;
        m_dValue = dValue;
    }
 
    // Make the Display class a friend of Storage (requires seeing the full declaration of class Display, as above)
    friend void Display::displayItem(Storage& storage);
};
 
// Now we can define Display::displayItem, which needs to have seen the full declaration of class Storage
void Display::displayItem(Storage &storage)
{
    if (m_displayIntFirst)
        std::cout << storage.m_nValue << " " << storage.m_dValue << '\n';
    else // display double first
        std::cout << storage.m_dValue << " " << storage.m_nValue << '\n';
}
 
int main()
{
    Storage storage(5, 6.7);
    Display display(false);
 
    display.displayItem(storage);
 
    return 0;
}
{% endhighlight %}

However, a better solution would have been to put each class declaration in a separate header file, with the function bodies in corresponding .cpp files. That way, all of the class declarations would have been visible immediately, and no rearranging of classes or functions would have been necessary.

# Anonymous variables and objects

## anonymous variable

An __anonymous variable__ is a variable that is given no name. Anonymous variables in C++ have __“expression scope”__, meaning they are destroyed at the end of the expression in which they are created. Consequently, they must be used immediately!

For example:

{% highlight CPP linenos %}
int Add(int nX, int nY)
{
    return nX + nY;
}
{% endhighlight %}

When the expression nX + nY is evaluated, the result is placed in an anonymous, unnamed variable. A copy of the anonymous variable is returned to the caller by value.

## anonymous class objects

Although our prior examples have been with built-in data types, it is possible to construct anonymous objects of our own class types as well. This is done by creating objects like normal, but omitting the variable name.

{% highlight CPP linenos %}
Cents cCents(5); // normal variable
Cents(7); // anonymous variable
{% endhighlight %}

A more concret example:

{% highlight CPP linenos %}
class Cents
{
private:
    int m_nCents;
 
public:
    Cents(int nCents) { m_nCents = nCents; }
 
    int GetCents() { return m_nCents; }
};
 
Cents Add(Cents &c1, Cents &c2)
{
    return Cents(c1.GetCents() + c2.GetCents());
}
 
int main()
{
    Cents cCents1(6);
    Cents cCents2(8);
    /* Here "Add() returns a anonymous class object "*/
    std::cout << "I have " << Add(cCents1, cCents2).GetCents() << " cents." << std::endl;
 
    return 0;
}
{% endhighlight %}


