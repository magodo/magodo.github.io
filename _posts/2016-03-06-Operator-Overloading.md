---
layout: "post"
title: "C++: 操作符重载"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

C++的操作符重载有以下几个注意点：

1. 至少有一个操作数是用户自定义的类型（类）；
2. 只能重载C++中已经存在的操作符（例如不能重载`**`）；
3. 所有被重载的操作符保持它们之前的**优先级**和**结合方向**

C++的操作符重载一般可以通过两种方法实现：

* 通过`friend function`实现
* 通过`class member function`实现

# 1. 使用 Friend Function进行操作符重载

使用`friend function`进行操作符重载一般用于那些不会对该类型内部变量进行修改的情况。

(虽然只要在定义该函数的时候，使用"non-const"引用来声明参数也是可以修改内部变量。可是，这样子的做法违反了封装原则，即类型的私有变量会被外界函数修改)

## 1.1 重载算术操作符（+，-，*，/）

由于算术操作符不会改变操作数，因此只需要使用`friend function`来重载即可。

{% highlight CPP linenos %}

#include <iostream>

using namespace std;

class Playground
{
    private:
        int m_length;
    public:
        Playground(int len);
        int getLength();

        // 操作数都为自定义类型
        friend Playground operator+(const Playground &obj1, const Playground &obj2);

        // 操作数中其中一个为自定义类型，每个操作需要定义两个
        friend Playground operator+(const Playground &obj, int len);
        friend Playground operator+(int len, const Playground &obj);
};

Playground::Playground(int len)
{
    m_length = len;
}

int Playground::getLength()
{
    return m_length;
}

Playground operator+(const Playground &obj1, const Playground &obj2)
{
    // 由于是friend function, 因此可以访问该类的对象的private成员
    return Playground(obj1.m_length + obj2.m_length);
}

Playground operator+(const Playground &obj, int len)
{
    // 由于是friend function, 因此可以访问该类的对象的private成员
    return Playground(obj.m_length + len);
}

Playground operator+(int len, const Playground &obj)
{
    // 这里直接使用了上面重加载的函数，可以使代码紧凑和简洁
    return obj + len;
}

int main()
{
    Playground obj1(1), obj2(2);
    Playground sum1 = obj1 + obj2;
    Playground sum2 = obj1 + 10;
    Playground sum3 = 20 + obj1;
    cout << sum1.getLength() << endl;
    cout << sum2.getLength() << endl;
    cout << sum3.getLength() << endl;
}

{% endhighlight %}

几个注意点：

1. 由于重载函数是`friend function`, 因此可以直接在函数内访问/修改（如果输入参数为non-const）该类的对象的private成员；
2. 当操作符不会改变操作数时（例如所有算术操作符都不会），则把`friend function`中的操作数都定义为`const`；
3. 当只有一个操作数是该类型，需要定义两个`friend function`来处理不同的参数顺序。其中一个函数的实现可以借助于调用另一个。

## 1.2 重载I/O操作符

有时候，自定义的类中有很多状态，想要一次性输出所有状态到输出流中需要很多trival的操作。这时候，可以通过重载自定义类型的`<<`操作符来达到你想要定义的输出格式。

重载`<<`时需要注意的是，`<<`的输入参数分别为`ostream`与自定义类（如：`std::cout << MyClass`）

又有时候，自定义类的输入方式也想改变，这时候可以通过重载其`>>`操作符。

重载`>>`时需要注意的是，`>>`的输入参数分别为`istream`与自定义类（如：`std::cin >> MyClass`）；还需要注意的是，`>>`一般会需要改变对象的private成员哦，这样子的话，`friend function`中的第二个参数必须为`non-const`!

此外，这两个操作符返回的都是`istream`/`ostream`的引用，这样可以支持chainable operation!


{% highlight cpp linenos %}

#include <iostream>

class MyData
{
    private:
        int m_year;
        int m_month;
        int m_day;

    public:
        MyData(int year = 2016, int month = 3, int day = 6);

        /* 注意返回值是ostream/istream的引用！！！ 
           返回ostream/istream是为了支持"chain"操作；
           返回对其的引用可别忘了。。
        */
        friend std::ostream& operator<<(std::ostream&, const MyData&);
        friend std::istream& operator>>(std::istream&, MyData&);
};

MyData::MyData(int year, int month, int day)
{
    m_year = year;
    m_month = month;
    m_day = day;
}

std::ostream& operator<<(std::ostream &out, const MyData &obj)
{
    out << "-------\nDate Info\n-------\n"
        << "Year: " << obj.m_year << "\n"
        << "Month: " << obj.m_month << "\n"
        << "Day: " << obj.m_day << "\n";
    return out;
}

/* 注意， >> 的第二个参数不再是const的了，因为要对其成员进行修改！ */
std::istream& operator>>(std::istream &in, MyData &newday)
{
    std::cout << "Enter new day(year month day):\n";
    in >> newday.m_year;
    in >> newday.m_month;
    in >> newday.m_day;
    return in;
}

int main()
{
    MyData today, newday;

    std::cin >> newday;
    /* chainable 操作 */
    std::cout << newday << today;
}
{% endhighlight %}

以上代码输出：

{%highlight CPP linenos%}

➜  io_operator ./a.out  
Enter new day(year month day):
2016
10
2
-------
Date Info
-------
Year: 2016
Month: 10
Day: 2
-------
Date Info
-------
Year: 2016
Month: 3
Day: 6

{%endhighlight%}

注意, 这里对于`<<`的重载的第二个输入参数是类对象的引用。这样子造成一个问题是：如果当要输出的对象只是个anonymous object, 也就意味着该对象只是一个rvalue（例如`(i+j)`返回的对象）, 是没有引用的. 这样就不能直接用于这种重载后的`<<`操作符了。

于是有的人（我）可能想将该参数作为类的对象的值进行定义，可是这会引发更大的问题：当对象中有成员变量是动态分配的内存的指针时，会容易发生crash！见下面的例子：

{%highlight CPP linenos%}

#include <iostream>
#include <string.h>
#include <stdlib.h>

class MyString
{
    private:
        char *m_string;
        int m_length;
    public:
        MyString(const char *string = "");
        ~MyString();
        /* 这里使用类的对象实体作为输入参数会触发该类的拷贝构造函数
         * 如果没有定义拷贝构造函数，则会使用C++默认的，浅拷贝
         * 如果该类成员变量有指向动态内存的指针，并且在析构函数中被释放，则会导致double free的crash
         */
        friend std::ostream& operator<<(std::ostream&, const MyString);
        /* 正确的做法是传入该对象的引用 */
        //friend std::ostream& operator<<(std::ostream&, const MyString&);
};

MyString::MyString(const char *string)
{
    if (string != NULL)
    {
        m_length = strlen(string) + 1;
        m_string = new char[m_length];
        if (m_string != NULL)
        {
            strncpy(m_string, string, m_length);
            m_string[m_length-1] = '\0';
        }
    }
}

MyString::~MyString()
{
    delete[] m_string;
    m_string = NULL;
}

//std::ostream& operator<<(std::ostream &out, const MyString &obj)
std::ostream& operator<<(std::ostream &out, const MyString obj)
{
    out << obj.m_string;
    return out;
}

int main()
{
    MyString obj("hello");
    std::cout << obj << '\n';
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
➜  io_operator ./a.out 
hello
*** glibc detected *** ./a.out: double free or corruption (fasttop): 0x08d45a10 ***
{%endhighlight%}

因此，结论还是使用引用比较安全。

## 1.3 重载比较操作符（==, !=, <, <=, >, >=）

比较操作符的重载和算术操作符类似，因为它们都不会修改操作数。所以，我们依然使用`friend function`，传入`const`参数。

{%highlight CPP linenos%}

#include <iostream>

class Score
{
    private:
        /* 假设成绩的比较有优先级：
           编程 > 数学
         */
        int m_programming;
        int m_math;

    public:
        Score(int prog = 0, int math = 0);
        friend bool operator==(const Score&, const Score&);
        friend bool operator!=(const Score&, const Score&);
        friend bool operator>(const Score&, const Score&);
        friend bool operator>=(const Score&, const Score&);
        friend bool operator<(const Score&, const Score&);
        friend bool operator<=(const Score&, const Score&);
};

Score::Score(int prog, int math)
{
    m_programming = prog;
    m_math = math;
}

bool operator==(const Score &score1, const Score &score2)
{
    return (score1.m_math == score2.m_math) && (score1.m_programming == score2.m_programming);
}

bool operator!=(const Score &score1, const Score &score2)
{
    /* 利用 == */
    return !(score1 == score2);
}

bool operator>(const Score &score1, const Score &score2)
{
    if (score1.m_programming == score2.m_programming)
        return (score1.m_math > score2.m_math);
    else
        return (score1.m_programming > score2.m_programming);
}

bool operator<=(const Score &score1, const Score &score2)
{
    /* 利用 > */
    return !(score1 > score2);
}

bool operator<(const Score &score1, const Score &score2)
{
    if (score1.m_programming == score2.m_programming)
        return (score1.m_math < score2.m_math);
    else
        return (score1.m_programming < score2.m_programming);
}
bool operator>=(const Score &score1, const Score &score2)
{
    /* 利用 < */
    return !(score1 < score2);
}

int main()
{
    using namespace std;

    Score kinokoScore(100, 100), magodoScore(100, 0);
    if (kinokoScore != magodoScore)
    {
        cout << "Not equal!\n";
        if (kinokoScore > magodoScore)
            cout << "Magodo is stupid > ~ <\n";
        else
            cout << "Kinoko is stupid > ~ <\n";
    }
    else
        cout << "Equal!\n";
}
{%endhighlight%}

以上代码输出：

{%highlight CPP linenos%}
➜  comparison_operator ./a.out 
Not equal!
Magodo is stupid > ~ <
{%endhighlight%}

## 1.4 重载一元操作符（+, -, !）

首先，一元操作符和之前所说的算术操作符和比较操作符一样，不会改变操作数，所以，我们依然使用`friend function`，传入`const`参数。

其次，一元操作符只有一个输入参数！

{%highlight CPP linenos%}
#include <iostream>

class Point
{
    private:
        int m_x;
        int m_y;

    public:
        Point(int x = 0, int y = 0);
        friend Point operator-(const Point &point);
        friend bool operator!(const Point &point);
        friend std::ostream& operator<<(std::ostream &out, Point &point);
};

Point::Point(int x, int y)
{
    m_x = x;
    m_y = y;
}

Point operator-(const Point &point)
{
    return Point(-(point.m_x), -(point.m_y));
}

bool operator!(const Point &point)
{
    return (point.m_x == 0 && point.m_y == 0);
}

std::ostream& operator<<(std::ostream &out, Point &point)
{
    out << " (" << point.m_x << ", " << point.m_y << ") ";
    return out;
}

int main()
{
    Point origin_point, some_point(3, 3), OP = -some_point;

    if (!origin_point)
    {
        std::cout << origin_point << "is origin point!\n";
    }
    if (!some_point)
    {
        std::cout << some_point << "is origin point!\n";
    }

    std::cout << "Oposite " << some_point << " -> " << OP << '\n';
}
{%endhighlight%}

以上代码输出：

{%highlight CPP linenos%}
➜  unary_operator ./a.out  
 (0, 0) is origin point!
 Oposite  (3, 3)  ->  (-3, -3)
{%endhighlight%}

# 2. 使用 成员函数 进行重载

使用成员函数进行操作符重载一般用于那些会修改类的内部变量的操作符。

这种操作符的重载方式有以下限制：

* 最左边的操作数必须是该类的一个对象；
* 最左边的操作数是该成员函数隐式传入的`this`指针，其他的操作数则是该函数的参数

大部分的操作符既可以通过该方式被重载，也可以通过`friend function`来重载 ，除了以下的例外：

* 以下操作符需要通过`friend function`进行重载：
  * `operator+(int, YourClass)`
  * `oeprator<<(ostream&, YourClass)`
  * `oeprator>>(istream&, YourClass)`
  * 等等

  因为，这些操作符最左边的操作数不是该类的一个对象。

* 以下操作符需要通过成员函数进行重载：
  * assignment (=, +=, -=, *=, /=, ++, --, ...)
  * subscript ([])
  * call (())
  * member selection (->)

## 2.1 使用 成员函数 重载 Friend Function 可以重载的操作符


{%highlight CPP linenos%}
#include <iostream>

class Point
{
    private:
        int m_x;
        int m_y;

    public:
        Point(int x = 0, int y = 0);

        /* 成员函数 （注意，第一个参数是隐式传入的this指针） */
        Point operator+(const Point&);

        /* 成员函数 （注意，第一个参数是隐式传入的this指针） */
        Point operator-();
        
        /* 只能使用friend function */
        friend std::ostream& operator<<(std::ostream&, const Point);
};

Point::Point(int x, int y)
{
    m_x = x;
    m_y = y;
}

Point Point::operator+(const Point &point)
{
    return Point(m_x+point.m_x, m_y+point.m_y);
}

Point Point::operator-()
{
    return Point(-m_x, -m_y);
}

std::ostream& operator<<(std::ostream &cout, const Point point)
{
    cout << "(" << point.m_x << ", " << point.m_y << ") ";
}

int main()
{
    Point point(3, 3);

    std::cout << -point << std::endl;
    std::cout << point + point << std::endl;

}
{%endhighlight%}
 
输出：

{%highlight CPP linenos%}
➜  intro ./a.out 
(-3, -3) 
(6, 6) 
{%endhighlight%}

注意，C++看到成员函数的原型（例如：`Point Point::operator-()`）时，编译器会将其转化为帶`this`指针的函数（例如：`Point operator-(const Point *this)`）。

## 2.2 重载 ++ 和 --

别忘了`++`和`--`都有两个版本哦！由于它们会改变类对象内部的成员变量，因此最好使用成员函数的方式来重载。


{%highlight CPP linenos%}
#include <iostream>

class Month
{
    private:
        int m_month;

    public:
        Month(int m = 1);
        Month& operator++();
        Month& operator--();
        Month operator++(int);
        Month operator--(int);
        friend std::ostream& operator<<(std::ostream&, const Month);
};

Month::Month(int m)
{
    m_month = m;
}

/* ++month */
Month& Month::operator++()
{
    if (m_month == 12)
    {
        m_month = 1;
    }
    return *this;
}

/* --month */
Month& Month::operator--()
{
    if (m_month == 1)
    {
        m_month = 12;
    }
    return *this;
}

/* month++ */
Month Month::operator++(int)
{
    Month temp(m_month);
    ++(*this);
    return temp;
}

/* month-- */
Month Month::operator--(int)
{
    Month temp(m_month);
    --(*this);
    return temp;
}

std::ostream& operator<<(std::ostream &out, const Month month)
{
    out << month.m_month;
    return out;
}

int main()
{
    Month month(12);
    std::cout << ++month << '\n';
    std::cout << --month << '\n';

    std::cout << month++ << '\n';
    std::cout << month << '\n';
    std::cout << month-- << '\n';
    std::cout << month << '\n';
}
{%endhighlight%}

输出：


{%highlight CPP linenos%}
➜  inc_dec ./a.out       
1
12
12
1
1
12
{%endhighlight%}

注意：

1. 由于C++在函数重载的过程中是通过函数的参数的数量和类型来区分重载的方式的，对于`postfix`形式的`++`/`--`，函数参数的数量和类型与`prefix`形式的是相同的。C++提供一种叫做`dummy variable`或者`dummy argument`的方式来进行区分，利用一个假的(fake)整形(`int`)参数来作为`postfix`与`prefix`的区分（我们甚至在定义函数的时候都不许要给这个int参数一个名字！）；
2. `prefix`形式，我们返回的是该对象自己的一个引用；`postfix`形式，我们返回的是一个新的object，其内容为调用对象在调用前的内容；
3. 我们在定义`postfix`形式的时候，直接使用之前已经定义过的`prefix`形式的代码。减少代码量并且降低出错的几率。

## 2.3 重载 下标索引 subscriptor []


{%highlight CPP linenos%}
#include <iostream>
#include <cassert>

class MyString
{
    private:
        char m_string[10];

    public:
        MyString();

        /* 注意，返回值为引用哦 */
        char& operator[](int);
};

MyString::MyString()
{
    for (int i = 0; i < 10; i++)
        m_string[i] = 'a';
}

char& MyString::operator[](int subscript)
{
    /* 这里可以进行边界检查 */
    assert(subscript < 10 && subscript >= 0);
    return m_string[subscript];
}

int main()
{
    MyString mystring;

    mystring[0] = 'b';
    std::cout << mystring[0] << '\n';
    std::cout << mystring[1] << '\n';
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
➜  subscript ./a.out  
b
a
{%endhighlight%}

注意：

1. 由于下标操作符的返回值应该是个lvalue(存在内存空间)，而指针和引用都保证了这一点，因此，在重载subscript的时候需要将返回值声明为引用。这样，返回的m_string元素是真正的m_string元素，也就支持直接对其就行修改了；
2. 重载下标操作符可以使程序员进行边界检查，这是很容易出错的一个点，因为编译器不会报错，只有运行时才会暴露。

## 2.4 重载圆括号(parenthesis) ()

圆括号操作符的特点是它允许各种操作数的**数目**和**类型**。

注意：

1. 圆括号操作符只能通过成员函数的方式进行重载；
2. 圆括号在C中一般用于函数调用或者是作为子表达式来获得更高的优先级。而在这里，圆括号操作符只是一个操作符，和上述两者都无关（例如，可以用作对二位或高维数组的索引。因为[]操作符的操作数的数目是被限定只有一个）


{%highlight CPP linenos%}
#include <iostream>
#include <cassert>

class Matrix
{
    private:
        int m_point[4][4];

    public:
        Matrix();
        int& operator()(const int, const int);
        friend std::ostream& operator<<(std::ostream&, const Matrix);
};

Matrix::Matrix()
{
    for (int i = 0; i < 4; i++)
        for (int j = 0; j < 4; j++)
            m_point[i][j] = 0;
}

/* 重载 输入参数为2个时 的一种()操作符 */
int& Matrix::operator()(const int x, const int y)
{
    assert( x >= 0 && x < 4 && y >= 0 && y < 4);
    return m_point[x][y];
}

std::ostream& operator<<(std::ostream &out, const Matrix obj)
{
    for (int i = 0; i < 4; i++)
    {
        for (int j = 0; j < 4; j++)
        {
            out << obj.m_point[i][j] << " ";
        }
        out << '\n';
    }
    return out;
}

int main()
{
    Matrix mat;

    mat(0,0) = mat(1,1) = mat(2,2) = mat(3,3) = 1;
    std::cout << mat;
}
{%endhighlight%}

输出：


{%highlight CPP linenos%}
➜  parenthesis ./a.out 
1 0 0 0 
0 1 0 0 
0 0 1 0 
0 0 0 1 
{%endhighlight%}

注意：虽然()操作符非常灵活，但是请不要过多使用。因为，我们不能通过()给使用者太多有用的信息。它一般用于取多为数组的索引方法，其他的方法一般建议使用正常的成员函数来实现，这样更具有描述性。

## 2.5 重载类型转换操作符

虽然C++知道如何在内部的类型之间进行隐式转化，可是C++不知到用户自定义的类如何转化。

假设我们有如下这个类：

{%highlight CPP linenos%}
#include <iostream>

class Meter
{
    private:
        int m_meter;

    public:
        Meter(int m)
        {
            m_meter = m;
        }

        int getMeter()
        {
            return m_meter;
        }
        /* 重载 int 类型转换符
         * 注意：
         *      1. 该重载函数的operator 与 int() 之间有个空格！
         *      2. 该重载函数没有返回类型（C++默认你的类型是正确的, i.e. int）
         */
        operator int()
        {
            return m_meter;
        }

};

class KiloMeter
{
    private:
        int m_km;

    public:
        KiloMeter(int km)
        {
            m_km = km;
        }

        /* 重载自己定义的类的转换符 */
        operator Meter()
        {
            return Meter(m_km * 1000);
        }



};

void printLength(int m)
{
    std::cout << m << '\n';
}

void printMeter(Meter m)
{
    /* 重载类型转换符后，可以使用强制转换 */
    std::cout << static_cast<int>(m) << '\n';
}

int main()
{
    Meter meter(100);

    printLength(meter.getMeter());
    /* 更简洁 */
    printLength(meter);

    KiloMeter km(100);
    printMeter(km);

}
{%endhighlight%}

注意：

1. 重载函数的`operator` 与 类型(e.g. `int()`) 之间有个空格！
2. 重载函数没有返回类型（C++默认你的类型是正确的, e.g. int）
3. 重载类型转换符后，可以使用`static_cast<>`进行类型转换
4. 类新转换也可以转换成自定义的类（例如这里的`KiloMeter`）

## 2.6  重载赋值操作符 = 

要明白对于赋值的重载，需要知道赋值操作符和Copy构造函数的关系，浅拷贝和深度拷贝的关系。

### 2.6.1 Assignment Operator VS Copy Constructor

**Assignment Operator** 

用于将值从某个对象拷贝到另一个**已经存在**的对象；

**Copy Constructor**

用于将值从某个对象拷贝到一个**新创建**的对象。一般有以下三种情况：

1. 实例化一个对象的同时初始化之：

        MyClass inst1(1);
        MyClass inst2 = inst1;  // 调用copy constructor

2.        通过值传递的某个对象。例如：
  * 参数类型为该类的函数的调用，则会触发被传入的对象的`Copy Constructor`(因为它要创建一个临时的对象在这个函数里使用)
  * 某个函数的返回类型为该类，则会触发被返回的对象的`Copy Constructor`

从代码的逻辑上来看，这两者应该是基本一样的，它们只是在不同的时刻被调用了而已。如下面的例子：

{%highlight CPP linenos%}
#include <iostream>

class Point
{
    private:
        int m_x;
        int m_y;
        /* Copy Constructor 与 Assignment Overload 都会去调用的函数*/
        void copyValue(const Point&);

    public:
        Point(int, int);

        /* Copy Constructor */
        Point(const Point&);

        /* Assignment Overload */
        Point& operator=(const Point&);

        friend std::ostream& operator<<(std::ostream&, Point);
};

Point::Point(int x, int y)
{
    m_x = x;
    m_y = y;
}

void Point::copyValue(const Point &srcObj)
{
    m_x = srcObj.m_x;
    m_y = srcObj.m_y;
}

/* Copy Constructor 
 * 注意：输入参数为Point的引用，这是因为如果不是引用而是值传递的话，
 *       在该函数（Copy Constructor）被调用的时候，会触发传入的对象
 *       的Copy Constructor(根据Copy Constructor的触发场合)，这又会
 *       再次触发自己的Copy Constructor……
 * 注意：这个函数没有返回值
 */
Point::Point(const Point &srcObj)
{
    copyValue(srcObj);
}

/* Assignment Overload */
Point& Point::operator=(const Point &srcObj)
{
    /* 由于C++中自己赋值给自己是允许的，这种情况下应该直接返回自己，
     * 否则，在该对象有动态分配的内存的情况下会引起问题
     */
    if (this != &srcObj)
    {
        m_x = srcObj.m_x;
        m_y = srcObj.m_y;
    }
    return *this;
}



std::ostream& operator<<(std::ostream &out, Point point)
{
    out << "(" << point.m_x << ", " << point.m_y << ") ";
    return out;
}

int main()
{
    Point point(2,2);
    Point p(1,1);
    std::cout << p << '\n';
    p = point;
    std::cout << p << '\n';

}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
(1, 1) 
(2, 2) 
{%endhighlight%}

注意：

1. 由于`Copy Constructor`与`Assignment Overload`的代码有大部分是重复的，因此将这部分代码放在一个private的成员函数中；
2. `Copy Constructor`与其他构造函数类似，没有返回值。注意：其输入参数为该类的引用；
3. `Assignment Overload`的返回值为调用对象自身的引用（为了chaining），其输入参数可以是该类的对象实体或者是引用（如果是实体，则在传入参数的时候会调用传入对象的`Copy Constructor`）。注意，由于C++允许对象赋值给自己，这种情况下`Assignment Overload`函数应该直接返回对象自身。因为，一旦该对象中有动态分配的内存，赋值给自己是很危险的(这个判断在`Copy Constructor`中是不需要的)。

C++既提供一个默认的`Copy Constructor`, 也提供一个默认的`Assignment Operator`. 然而，由于C++编译器不知到你定义的类的细节，因此这两个默认的函数做的Copy动作比较简单，仅仅使用member-wise copy(浅拷贝)。

### 2.6.2  Shallow Copy VS Deep Copy

#### 2.6.2.1 Shallow Copy

浅拷贝是把某个类的对象的成员变量按值赋值给另一个同类的对象。

当类成员中没有指向动态分配的内存的情况下，浅拷贝的行为模式是没问题的。然而，如果与动态内存有关系，则会导致很大的问题，因为浅拷贝仅仅拷贝了内存的地址（指针值传递）。

{%highlight CPP linenos%}
#include <iostream>
#include <string.h>
#include <stdlib.h>

class MyString
{
    private:
        int m_length;
        char *m_string;

    public:
        MyString(const char *string = "");
        ~MyString();

        char* getString();
        int getLength();
};

MyString::MyString(const char *string)
{
    m_length = strlen(string) + 1;
    m_string = new char[m_length];
    strncpy(m_string, string, m_length);
    m_string[m_length-1] = '\0'; // Make sure m_string is terminated by '\0'
}

MyString::~MyString()
{
    delete[] m_string;
    m_string = NULL;
}

int MyString::getLength()
{
    return m_length;
}

char* MyString::getString()
{
    return m_string;
}

int main()
{
    MyString hello("Hello");
    MyString insidious = hello;
    std::cout << hello.getString() << '\n';
    // Crash!!! Double free or corruption!
    // 因为两个对象的m_string 指向同一个内存地址，在析构的时候被重复free
}
{%endhighlight%}

上面的代码会在运行时报如下的错误：

{%highlight CPP linenos%}
➜  shallow_deep_copy ./a.out  
Hello
*** glibc detected *** ./a.out: double free or corruption (fasttop): 0x08454a10 ***
{%endhighlight%}

这是因为两个对象的`m_string`指向了同一个内存地址，在析构的时候被重复free。其根本原因是由于初始化`insidious`的时候调用的是C++默认的(浅拷贝) `copy constructor`，其中的指针是值传递的。

#### 2.6.2.2 Deep Copy

深度拷贝指被赋值的对象拥有其独立的成员变量和其成员变量指向的内存空间。

深度拷贝需要程序员重载C++默认的`Copy Consturctor`和`Assignment Operator`.

{%highlight CPP linenos%}
#include <iostream>
#include <string.h>
#include <stdlib.h>

class MyString
{
    private:
        int m_length;
        char *m_string;
        /* 共用copy函数 */
        void deepCopy(const MyString&);

    public:
        MyString(const char *string = "");
        ~MyString();

        /* copy constructor*/
        MyString(const MyString&);

        /* assignment operator */
        MyString& operator=(const MyString&);

        char* getString();
        int getLength();
};

MyString::MyString(const char *string)
{
    m_length = strlen(string) + 1;
    m_string = new char[m_length];
    strncpy(m_string, string, m_length);
    m_string[m_length-1] = '\0'; // Make sure m_string is terminated by '\0'
}

MyString::~MyString()
{
    delete[] m_string;
    m_string = NULL;
}

int MyString::getLength()
{
    return m_length;
}

char* MyString::getString()
{
    return m_string;
}

void MyString::deepCopy(const MyString &obj)
{
    m_length = obj.m_length;

    if (obj.m_string)
    {
        m_string = new char[m_length];
        strncpy(m_string, obj.m_string, m_length);
    }
    else
    {
        m_string = 0;
    }
}

/* copy constructor*/
MyString::MyString(const MyString &obj)
{
    deepCopy(obj);
}

/* assignment operator */
MyString& MyString::operator=(const MyString &obj)
{
    /* 注意：这里对赋值两边对象是否相同需要判断 */
    if (this != &obj)
    {
        /* 需要先清除已分配的内存
         * 否则，每当我们进行赋值的时候，就会产生一定的内存泄漏！
         * （new和delete要成对出现！）
         */
        delete[] m_string;
        deepCopy(obj);
    }
    return *this;
}

int main()
{
    MyString hello("Hello");
    MyString insidious = hello;
    std::cout << hello.getString() << '\n';
}
{%endhighlight%}

注意，`Copy Constructor`与`Assignment Operator`的实现有以下几个细微却重要的区别：

1. 后者在入口处有判断是否赋值操作符的两边的对象是同一个。这是为了避免之后的操作中由memory alias引起的错误。例如：
  * `strncpy`的source和dest是同一个内存可能导致问题；
  * `delete[] m_string`会将assigning对象的m_string指向的内存空间也释放掉！
2. 后者先释放已分配的内存。如果不这么做会导致每一次的赋值操作都有内存泄漏；
3. 返回`*this`的引用，为了chainning.

### 2.6.3 Disable Copy

如果想disable C++默认的`Copy Constructor`与`Assignment Operator`，只需要：重载它们作为`private`成员函数。例如：

{%highlight CPP linenos%}
class MyString
{
    private:
        int m_length;
        char *m_string;

        /* copy constructor*/
        MyString(const MyString&);
        /* assignment operator */
        MyString& operator=(const MyString&);

    public:
        // reset of code
};
{%endhighlight%}
