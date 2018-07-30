---
layout: "post"
title: "C++ 输入和输出(I/O)"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. 输入输出流(I/O stream)

输入输出虽然不是C++核心语言的一部分，但是它由C++标准库提供。因此在`std`这个命名空间内。

## 1.1 iostream库

当你加入*iostream*这个头文件时，C++会提供你一套I/O相关的类结构（class hierarchy）:

![iostream class hierarchy](/assets/img/cpp/iostream_class_hierarchy.png)

**stream**

抽象层面上，一个流代表一串可以被顺序访问的字符。随着时间的推移，一个流可以产生(produce)数据，也会消耗(consume)数据.

通常，我们会处理两类流：输入流 和 输出流

输入流用于保从文件，网络，设备（例如键盘）等data producer端传来的数据。这些数据存于输入流中，直到程序来读取。

输出流用于将程序中的数据向文件，网络，设备（例如打印机）等data consumer端传输。这些数据会存于输出流中，直到data consumer准备完毕之后，才会真正被传出。

程序员只需要学会如何与这些输入流或者输出流交互，就可以与各种不同的设备进行数据传输。而其中不同设备的实现细节，则交由操作系统去处理。

## 1.2 C++中的I/O

**istream** 是用于处理输入流的类。它主要提供了一个extraction operator(>>)，用于将数据从输入流中取出。其中的数据是由其他设备产生的。

**ostream** 是用于处理输出流的类。它主要提供了一个insertion operator(<<)，用于将数据存入输出流中。其中的数据将被其他设备消耗。

**iostream** 既可用于输入也可用于输出。

**_withassign** 的类多了一个assingment operator，允许用户将一个流赋值给另一个流。大部分情况下，不会用到这些类。

## 1.3 C++中的标准流

C++中有4种预定义的标准流：

* **cin** -- istream_withassign 类，绑定到标准输入（通常为键盘）
* **cout** -- ostream_withassign 类，绑定到标准输出（通常为显示器）
* **cerr** -- ostream_withassign 类，绑定到标准错误（通常为显示器）。提供非缓冲输出。
* **clog** -- ostream_withassign 类，绑定到标准错误（通常为显示器）。提供缓冲输出。

# 2. 使用istream输入

## 2.1 抽取操作符(>>)

`>>`可以用于从输入流中读取数据。在读取数据的时候，有个普遍的问题是，如何保证输入流中的数据不会溢出你的接收buffer。例如：

{%highlight CPP linenos%}
char buf[5];
std::cin >> buf;
{%endhighlight%}

如果用户通过键盘向标准输入流中输入超过5个字符(不包括换行符或EOF)，那么以上的代码就会溢出buffer。通常，我们不应该对输入流中数据的长度做任何假设。

有一种解决方法是使用**manipulator**. **manipulator**是一种与`<<`或`>>`一起使用时修改stream的对象。例如，`std::endl`就是一个**manipulator**，它既输出一个换行符，又flush任何被缓存的数据。

C++提供另一个manipulator -- **setw**(头文件：*iomanip*) -- 可被用于限制从流中读入字符的数量。使用setw，仅需传入限制个数，并且插入在流和buffer之间：

{%highlight CPP linenos%}
std::cin >> std::setw(5) >> buf;
{%endhighlight%}

这样，如果用户输入了6个字符，那么第一次只会读出其中的前4个字符留一位存EOF。第二次再调用`>>`会继续读入剩余的2位:

{%highlight CPP linenos%}
#include <iostream>
#include <iomanip>

int main()
{
    char buf[5];

    std::cin >> std::setw(5) >> buf;
    std::cout << "*\n" << buf << "\n*\n";
    std::cin >> std::setw(5) >> buf;
    std::cout << "*\n" << buf << "\n*\n";
}
{%endhighlight%}

运行：

    stream > ./a.out 
    123456
    *
    1234
    *
    *
    56
    *

## 2.2 cin的其他成员函数

1. `>>` 遇到字符串中的空格（blanks, tab, newline）就停止读取，并且不会保存输入这个空格；
2. `cin.get(char&)` 用于读取一个字符存于char变量中
3. `cin.get(char *buf, int limit)` 用于读取最多*limit-1*个字符存于buf中，然后在buf中补上EOF。
   不过，buf不会读取输入流中的换行符，并且在遇到换行符的时候就停止输入，返回。这会导致后续的对该输入流的`get`读取立即返回，因为第一个字符就是换行符。例如：

        #include <iostream>

        int main()
        {
            char buf[10];
            char ch;
            std::cin.get(buf, 10);
            std::cout << buf;

            std::cin.get(buf, 10);
            std::cout << buf;
        }

   运行:

        stream > ./a.out 
        hello
        hello02_istream > 

   可见，第二个`get`立即返回了。

4. `cin.getline(char *buf, int limit)` 和`get`的唯一区别就是会读取换行符，但是不会将其存入buf中。例如：

        #include <iostream>
        #include <string.h>

        int main()
        {
            char buf[10];
            char ch;
            std::cin.getline(buf, 10);
            std::cout << buf;
            std::cout << "Last read: " << std::cin.gcount() << " characters."  << '\n';
            std::cout << "Length of buf: " << strlen(buf) << '\n';

            std::cin.getline(buf, 10);
            std::cout << buf;
            std::cout << "Last read: " << std::cin.gcount() << " characters."  << '\n';
            std::cout << "Length of buf: " << strlen(buf) << '\n';
        }

   运行：

        02_istream > ./a.out                                                                             
        hello
        helloLast read: 6 characters.
        Length of buf: 5
        hi
        hiLast read: 3 characters.
        Length of buf: 2

5. `cin.gcount()` 返回上次被读取的字符个数。

6. `cin.ignore()` 丢弃stream中的第一个字符

7. `cin.ignore(int nCount)` 丢弃stream中的前nCount个字符

8. `cin.peek()` 允许从stream中读取一个字符并且依然将该字符保存于流中

9. `cin.unget()` 将上一个被读取出stream的字符重新返回stream中，从而会被下一个读取操作读取

10. `cin.putback(char ch)` 允许将任意字符放入stream中，从而会被下一个读取操作读取

# 3. 使用ostream输出

插入操作符`<<`用于将数据传入一个输出流。

## 3.1 格式化输出

输出数据支持格式化输出(formatting)。有两种方式可以改变输出的格式：**flag** 和 **manipulators**

### 3.1.1 Flag

Flag可以被认为是一个布尔类型变量，可以被打开或关闭。

* 打开某个选项，使用`setf()`函数。打开多个选项，可以使用`|`；
* 关闭某个选项，使用`unsetf()`函数

例如：

{%highlight CPP linenos%}
#include <iostream>

int main()
{
    std::cout.setf(std::ios::showpos); // 显示"+"，对于positive number
    std::cout << 27 << '\n';
}
{%endhighlight%}

输出： `+27`

有一个需要注意的是：有些flag属于某些groups，称为**format group**. 一个**format group**中的flag通常提供类似的格式化功能，但往往是互斥的。例如，有一个**format group**称为*basefield*，它包含*oct*,*dec*,*hex*三个flag，用于控制整数的进制。默认情况下"dec"选项是打开的。按理说，如果我们想改变进制为16进制，可能会写成：

{%highlight CPP linenos%}
#include <iostream>
using namespace std;
int main()
{
    cout.setf(ios::hex);
    cout << 27 << '\n';
}
{%endhighlight%}

可是输出仍然为：`27`

原因是，`setf()`函数仅仅将某个flag打开 -- 它还没聪明到能够将互斥的开关关闭。因此，上例中ios::hex与ios::dec实际上都是打开状态的，而ios::dec显然有更高的优先级。

有两种方式可以解决这个问题：

1. 手动关闭ios::dec

        cout.unsetf(ios::dec);
        cout.setf(ios::hex);
        cout << 27 << endl;

2. 使用`setf()`函数的另一个形式，输入两个参数：第一个参数要被打开的flag，第二个参数是flag所处的formatting group。当使用这种形式的`setf()`时，所有属于该group的flag都被关闭，除了被传入的flag被打开。例如上例，我们可以：

        cout.setf(ios::hex, ios::basefield);
        cout << 27 << endl;

### 3.1.2 Manipulators

相对与FLag而言，Manipulator会好用很多。它是在stream输入输出语句中的一种对象，用于改变输入或输出的方式。

Manipulaotr的一个好处是，它会自动打开和关闭某些flag，例如上面改变整数进制的情况，使用Manipulator:

{%highlight CPP linenos%}
#include <iostream>

int main()
{
    using namespace std;
    cout << oct << 27 << endl;
    cout << 27 << endl;   // 依然是oct进制
    cout << dec << 27 << endl;
    cout << hex << 27 << endl;
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
33
33
27
1b
{%endhighlight%}

## 3.2 一些使用的formater(包括flag, manipulator, member function)

这里列出一些比较常用的flags, manipulators 以及 member functions。其中，

* flags属于`ios`类;
* manipulators属于`std` namespace;
* member function属于`ostream`类。

### 3.2.1 布尔显示

|---
| Group | Flag | Meaning |
|:-|:-|:-|
|| boolalpha | 如果打开，bool值输出为"true" or "false";否则，输出为 0 or 1
|===

|---
|Manipulator|Meaning
|:-|:-|
|boolalpha|布尔值输出为"true" or "false"
|noboolalpha|布尔值输出为 0 or 1(default)
|===

### 3.2.2 正数"+"号显示


|---
| Group | Flag | Meaning |
|:-|:-|:-|
|| showpos | 如果打开，则在正数前加上"+"
|===

|---
|Manipulator|Meaning
|:-|:-|
|showpos|在正数前加"+"
|noshowpos|不在正数前加"+"
|===

### 3.2.3 大小写

|---
| Group | Flag | Meaning |
|:-|:-|:-|
|| uppercase | 如果打开，则显示大写的字母(例如：1.23456E+007)
|===


|---
|Manipulator|Meaning
|:-|:-|
|uppercase|显示大写的字母
|nouppercase|显示小写字母
|===

### 3.2.4 整数的进制


|---
| Group | Flag | Meaning |
|:-|:-|:-|
|basefield| dec | 十进制显示整数(default)
|basefield| hex | 十六进制显示整数
|basefield| oct | 八进制显示整数
|basefield| (none) | 按照整数开头的字符来决定(0,none,0x)
|===


|---
|Manipulator|Meaning
|:-|:-|
|dec|十进制显示整数
|hex|十六进制显示整数
|oct|八进制显示整数
|===

### 3.2.5 浮点数的精度，记号法(notation)，小数点

|---
| Group | Flag | Meaning |
|:-|:-|:-|
|floatfield|fixed|使用decimal的方式显示浮点数
|floatfield|scientific|使用科学技术发显示浮点数
|floatfield|(none)|对于小数短的情况，使用fixed;否则，使用科学技术法
|floatfield|showpoint|总是显示小数点和后面的0

|---
|Manipulator|Meaning
|:-|:-|
|fixed|使用decimal的方式显示浮点数
|scientific|使用科学技术发显示浮点数
|showpoint|总是显示小数点和后面的0
|noshowpoint|只显示有效数，如果小数点后没有0则不显示小数点
|setprecision(int)|设置精度(定义于：iomanip)
|===

|---
|Member function|Meaning
|:-|:-|
|precision()|返回当前的精度
|precision(int)|设置精度并且返回之前的精度
|===

如果设置了fixed或者scientific，precision会决定小数点后的个数。如果precision少于实际的有效位数，显示的数字会被rounded.

如果既没有设置fixed，又没有设置scientific，precision会决定有多少有效数会被显示（包括整数部分）。同样，如果precision少于实际的有效位数，显示的数字会被rounded.

### 3.2.6 宽度，填充字符，对齐

当我们输出的时候，默认情况下只会输出要输出的内容。有时候，我们希望输出的内容是向左对齐或者向右对齐的。这时，首先需要定义一个_field width_, 它定义了输出内容的总宽度。如果，实际的输出内容小于这个宽度，那么我们可以设置其左右对齐；否则，输出内容也不会被截断，仅仅是无视这个设置的_field width_而已。

|---
|Group|Flag|Meaning|
|:-|:-|:-|
|adjustfield|internal|数字：符号左对齐；值右对齐|
|adjustfield|left|左对齐|
|adjustfield|right|右对齐(default)|
|===

|---
|Manipulator|Meaning
|:-|:-
|internal|数字：符号左对齐；值右对齐|
|left|左对齐|
|right|右对齐|
|setfill(char)|设置输入的参数作为填充字符(定义于：iomanip)
|setw(int)|设置输入和输出的宽度(定义于：iomanip)
|| - 输出：如果实际长度更长，不会被截断
|| - 输入：如果实际长度更长，则剩余的内容会在下一次读取
|===

|---
|Member function|Meaning
|:-|:-
|fill()|返回当前的填充字符
|fill(char)|设置填充字符并返回之前的字符
|width()|返回当前宽度
|width(int)|设置宽度并返回之前的宽度
|===

要是用这里的formater，必须记得先调用`setw(int)`或者`width(int)`设置_field width_. 由于`setw(int)`或者`width(int)`只影响下一个输出语句，因此需要反复的调用。

# 4. 面向string的流

之前提到，都是往cout写或者从cin读。此外，还有被称为**string stream**的一组类可以允许你使用`<<`和`>>`来操作string. 

与istream和ostream一样，string stream也提供了buffer来保存流中的数据。不同的是，stream string并不是与I/O通道（例如，键盘，显示器等）相连的。stream string的其中一个主要作用是用来缓存将要被显示的数据，或者逐行地处理输入。

C++中有6个不同的string stream 类：

1. istringstream (继承自istream)
2. ostringstream（继承自ostream）
3. stringstream（继承自iostream）
4. wistringstream 
5. wostringstream 
6. wstringstream

前三个类适用于读写正常宽度的string；后三个用于读写宽字符的string. 它们都在<sstream>头文件中声明。

## 4.1 读取和写入

有两种方式将数据写入string stream中:

1. 使用`<<`操作符：

        stringstream ss;
        ss << "Hello";

2. 使用`str(std::string)`成员函数啦设置buffer中的值：

        stringstream ss;
        ss.str("Hello");

类似地，有两种方式读取string stream中的值：

1. 使用`>>`操作符：


        stringstream ss;
        string str;

        ss << "Hello World!";
        ss >> str;
        cout << str << endl;
        ss >> str;
        cout << str << endl;

   输出：

        Hello
        World!


2. 使用`str()`成员函数:

        stringstream ss;
        ss << "hello";
        cout << ss.str();


注意，`>>`被调用后，值会从流中被读走。而`str()`则仅仅是返回当前buffer中的值。

## 4.2 string和数字间的转换

由于`>>`和`<<`内部针对各种基本数据类型都有相应的处理，因此我们可以利用他们来对string和数字进行互相转换。

1. 数字转string:

        #include <iostream>
        #include <sstream>

        int main()
        {
            using namespace std;
            stringstream ss;
            int nValue = 12345;
            double dValue = 67.89;
            ss << nValue << " " << dValue;

            string strValue1, strValue2;
            ss >> strValue1 >> strValue2;
            cout << strValue1 << " " << strValue2 << endl;
        }

   输出：`12345 67.89`

2. string转为数字：


        #include <iostream>
        #include <sstream>

        int main()
        {
            using namespace std;
            stringstream ss;
            ss << "12345  67.89";
            int nValue;
            double dValue;

            ss >> nValue >> dValue;
            cout << nValue << " " << dValue << endl;
        }
  
   输出：`12345 67.89`

## 4.3 清空string stream buffer

有多种方式可以清空string stream buffer:

1. `ss.str("")`
2. `ss.str(std::string())`
3. `ss.clear()` 这个成员函数还会将stream的state设为OK, 清空error flags.



