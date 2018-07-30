---
layout: "post"
title: "C++ 组合(composition)"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. 组合
## 1.1 概念

组合（composition）可以使一些比较复杂的类中加入某些现有的，简单的类的实例作为成员变量。

例如，我们可以定义如下复杂类：

{%highlight CPP linenos%}
#include "RAM.h"
#include "CPU.h"
#include "Motherboard.h"

class PersonalComputer
{
    private:
        RAM           m_ram;
        CPU           m_cpu;
        Motherboard   m_motherboard;
};
{%endhighlight%}

## 1.2 初始化

利用“初始化列表（initializer list）”对其组合类的成员变量进行初始化。以上面的类为例：

{%highlight CPP linenos%}
PersonalComputer::PersonalComputer(int ram_size, int cpu_speed, char* motherboard_type)
                                    :m_ram(ram_size)
                                    ,m_cpu(cpu_speed)
                                    ,m_motherboard(motherboard_type)
{
}
{%endhighlight%}

需要注意的是，成员变量中的实例的所有权（ownership）是属于这个类的实例：

* 当这个类的实例被创建时，成员变量的实例被创建；
* 当这个类的实例被销毁时，成员变量的实例被销毁

## 1.3 一个实际的例子

以下的例子里创建了一个生物(`Creature`)，它**有一个**坐标(`Location`)。因此，构建了一个`Location`的类来处理坐标的显示和坐标的改变。同时，将`Location`的对象作为`Creature`的一个成员变量：

location.h:

{%highlight CPP linenos%}

#ifndef LOCATION_H
#define LOCATION_H

#include <iostream>

class Location
{
    private:
        int m_x;
        int m_y;
    public:
        Location(int x = 0, int y = 0);
        friend std::ostream& operator<<(std::ostream&, Location&);
        void SetLocation(int, int);
        int GetX();
        int GetY();
};

#endif
{%endhighlight%}

location.cc:

{%highlight CPP linenos%}

#include "location.h"

Location::Location(int x, int y)
            :m_x(x)
            ,m_y(y)
{
}

void Location::SetLocation(int x, int y)
{
    m_x = x;
    m_y = y;
}

int Location::GetX()
{
    return m_x;
}

int Location::GetY()
{
    return m_y;
}

std::ostream& operator<<(std::ostream &out, Location &obj)
{
    out << '(' << obj.m_x << "," << obj.m_y << ')';
    return out;
}

#ifdef TEST_LOCATION

int main()
{
    Location loc(2,2);
    std::cout << loc << '\n';
    loc.SetLocation(3,3);
    std::cout << loc << '\n';
}

#endif
{%endhighlight%}

creature.cc:

{%highlight CPP linenos%}

#include "location.h"
#include <string>

class Creature
{
    private:
        std::string m_name;
        Location m_loc;
    public:
        Creature(std::string name="", int x = 0, int y = 0);
        ~Creature();
        void MoveTo(int x, int y, bool no_print = true);
        void ShowCurrentLocation();
};

Creature::Creature(std::string name, int x, int y)
    :m_name(name)
    ,m_loc(x, y)
{
}

Creature::~Creature()
{
}

void Creature::MoveTo(int x, int y, bool no_print)
{
    m_loc.SetLocation(x, y);
    if (!no_print)
    {
        std::cout << m_name << " move to " << m_loc << '\n';
    }
}

void Creature::ShowCurrentLocation()
{
    std::cout << m_name << " is at " << m_loc << '\n';
}

int main()
{
    std::cout << "Enter the name of your creature:" << std::endl;
    std::string name;
    std::cin >> name;
    Creature obj(name);
    obj.ShowCurrentLocation();

    int x, y;
    while(1)
    {
        std::cout << "Where to move? ( <0 to quit)" << std::endl;
        std::cin >> x >> y;
        if (x >= 0 && y >= 0)
        {
            obj.MoveTo(x, y, false);
        }
        else
        {
            break;
        }
    }
    obj.ShowCurrentLocation();
}
{%endhighlight%}

以下为程序的执行片段：

{%highlight CPP linenos%}
➜  creature ./a.out 
Enter the name of your creature:
magodo
magodo is at (0,0)
Where to move? ( <0 to quit)
1 1
magodo move to (1,1)
Where to move? ( <0 to quit)
2 2
magodo move to (2,2)
Where to move? ( <0 to quit)
-1 -1
magodo is at (2,2)
{%endhighlight%}

# 2. 聚类 (Aggregation)

聚类是一种特殊的组合，它不会使成员变量中被聚类的类的实例的所有权交由聚类类的实例。也就是说，当一个聚类的类的实例被销毁的时候，该实例中的被聚类的类的实例不会被销毁。

在组合(composition)中，被组合的类的实例是通过在成员变量中定义：

* 那些类的对象
* 指向那些类的对象的指针（创建和销毁这些类是由该组合类负责）

在聚类（aggregation）中，被聚类的类的实例是通过在成员变量中定义：

* 指向那些类的对象的指针（创建和销毁不由聚类负责）
* 对于那些类的对象的引用（创建和销毁不由聚类负责）

因此，聚类中的成员变量中的对象来自于以下几种情况之一：

* 在聚类的构造函数中作为参数传入；
* 通过聚类的一些存取函数(accessor and mutator)来传入

值得注意的是，组合和聚类不是互斥的。一个类里既可以有组合的成员变量，也可以有聚类的成员变量。就好比学校里的一个部门，部门的名字，职能是组合的成员变量；而部门里的老师是聚类的成员变量。因为，当部门关闭的时候，这些老师还是存在的。

虽然聚类是非常有用的，但是使用过程中也要有所注意。尤其是因为聚类的类不会销毁被聚类的对象。如果，当聚类的对象被销毁后没有其他指针或引用指向这些被聚类的对象，则会造成内存泄漏！

## 2.1 一个栗子


{%highlight CPP linenos%}

#include <iostream>
#include <string>
#include <vector>

class Teacher
{
    private:
        std::string m_name;
    public:
        Teacher(std::string name);
        std::string GetName();
};

class SchoolDepartment
{
    private:
        std::string m_name;
        std::vector<Teacher*> m_teachers;

    public:
        SchoolDepartment(std::string name);
        std::string GetName();
        void AddTeacher(Teacher *teacher);
        void ShowAllTeacher();
};

/* Define member functions for Teacher */
Teacher::Teacher(std::string name)
    :m_name(name)
{
}

std::string Teacher::GetName()
{
    return m_name;
}

/* Define member functions for SchoolDepartment */
SchoolDepartment::SchoolDepartment(std::string name)
    :m_name(name)
{
}

void SchoolDepartment::AddTeacher(Teacher *teacher)
{
    m_teachers.push_back(teacher);
}

void SchoolDepartment::ShowAllTeacher()
{
    for (auto p_teacher: m_teachers)
    {
        std::cout << p_teacher->GetName() << std::endl;
    }
}

std::string SchoolDepartment::GetName()
{
    return m_name;
}

/* MAIN */
int main()
{
    Teacher teacher1("magdodo"), teacher2("kinoko");
    SchoolDepartment *math_dep = new SchoolDepartment("Math");

    std::cout << "There is a department in school: "  << math_dep->GetName() << "\n";
    std::cout << "Currently has no teacher! So hiring smart employees..." << '\n';
    math_dep->AddTeacher(&teacher1);
    math_dep->AddTeacher(&teacher2);
    std::cout << "Now " << math_dep->GetName() << " department has got following teachers:\n"  << '\n';
    math_dep->ShowAllTeacher();
    std::cout << "\nBecause of teachers are not that good, " << math_dep->GetName() << " department is closing..." << '\n';
    delete math_dep;
    std::cout << "\nHowever, the two damn teachers are still there, they are:\n\n";
    std::cout << teacher1.GetName() << '\n';
    std::cout << teacher2.GetName() << '\n';
}

{%endhighlight%}

输出：

{%highlight CPP linenos%}
There is a department in school: Math
Currently has no teacher! So hiring smart employees...
Now Math department has got following teachers:

magdodo
kinoko

Because of teachers are not that good, Math department is closing...

However, the two damn teachers are still there, they are:

magdodo
kinoko
{%endhighlight%}

# 3. 容器类 （Container Class）

容器类是一种持有和管理某个其他类的实例的一种特殊的类。

容器类一般都会至少提供以下这些功能：

1. 创建一个空的容器（通过构造函数）
2. 插入新的对象到容器
3. 从容器中删除对象
4. 记录当前容器内对象的数量
5. 清空容器
6. 访问存储的对象
7. 对对象排序（可选）

容器类一般有两种实现方式：

* **Value Containers** 以组合（composition）的方式存储所持有的对象（因此，负责创建和销毁这些对象）；

* **Reference Containers** 以聚类的方式存储持有对象的指针或引用

C++的容器类通常只持有一种类（型）的对象。如果需要一个既持有整形对象，又持有double对象的容器，你就必须写两个容器，或者使用模板。

## 3.1 一个数组容器类

array.h

{% highlight CPP linenos %}
#include <iostream>

class IntArray
{
    private:
        int m_length;
        int *m_elements;
        void Allocate(int length);                      // common function for reuse

    public:
        IntArray();                                     // default constructor
        IntArray(int length);                           // construct "length" array
        ~IntArray();
        int GetLength();                                 
        void Erase();
        void Reallocate(int length);                    // Reallocate a fixed length array and erase all old data
        void Resize(int length);                        // Resize current array and keep the data
        void Insert(int value, int index);              // Insert a new value at index, all value after index(include the one at index) will shitf one bit afterwards if any
        void Remove(int index);                         // Delete the value at index
        int& operator[](int index);                     // Overload operator[] for reference the object in array
};
{% endhighlight %}

array.cpp

{% highlight CPP linenos %}

#include <cassert>
#include "array.h"

IntArray::IntArray()
    :m_length(0)
    ,m_elements(NULL)
{
}

IntArray::IntArray(int length)
{
    Allocate(length);
}

IntArray::~IntArray()
{
    delete[] m_elements;
}

void IntArray::Allocate(int length)
{
    assert(length >= 0);
    int *newElements = new int[length];
    if (newElements != NULL)
    {
        m_length = length;
        m_elements = newElements;
    }
}

int IntArray::GetLength()
{
    return m_length;
}

void IntArray::Erase()
{
    delete[] m_elements;
    m_elements = NULL;
    m_length = 0;
}

void IntArray::Reallocate(int length)
{
    Erase();
    Allocate(length);
}

void IntArray::Resize(int length)
{
    assert(length >= 0);
    int *newElements = new int[length];
    if (newElements != NULL)
    {
        int copyLength = (length > m_length)? m_length:length;
        --copyLength;
        for (;copyLength >= 0; --copyLength)
        {
            newElements[copyLength] = m_elements[copyLength];
        }
        Erase();
        m_elements = newElements;
        m_length = length;
    }
}

void IntArray::Insert(int value, int index)
{
    assert(index >= 0 && index <= m_length);
    int *newElements = new int[m_length + 1];
    if (newElements != NULL)
    {
        int i;
        for (i = 0; i <= index-1; i++)
        {
            newElements[i] = m_elements[i];
        }
        newElements[index] = value;
        for (i = index+1; i <= m_length; i++)
        {
            newElements[i] = m_elements[i-1];
        }
        delete[] m_elements;
        m_length++;
        m_elements = newElements;
    }
}

void IntArray::Remove(int index)
{
    assert(index >= 0 && index < m_length);
    int *newElements = new int[m_length-1];
    int i;
    for (i = 0; i < index; i++)
    {
        newElements[i] = m_elements[i];
    }
    for (i = index+1; i < m_length; i++)
    {
        newElements[i-1] = m_elements[i];
    }
    delete[] m_elements;
    --m_length;
    m_elements = newElements;
}

int& IntArray::operator[](int index)
{
    assert(index >= 0 && index < m_length);
    return m_elements[index];
}

/* MAIN */
using namespace std;
 
int main()
{
    // Declare an array with 10 elements
    IntArray cArray(10);
 
    // Fill the array with numbers 1 through 10
    for (int i=0; i<10; i++)
        cArray[i] = i+1;
 
    // Resize the array to 8 elements
    cArray.Resize(8);
 
    // Insert the number 20 before the 5th element
    cArray.Insert(20, 5);
 
    // Remove the 3rd element
    cArray.Remove(3);
 
    // Print out all the numbers
    for (int j=0; j<cArray.GetLength(); j++)
        cout << cArray[j] << " ";
    cout << '\n';
 
    return 0;
}
{% endhighlight %}

输出：

{% highlight CPP %}
➜  container ./a.out 
1 2 3 5 20 6 7 8 
{% endhighlight %}
