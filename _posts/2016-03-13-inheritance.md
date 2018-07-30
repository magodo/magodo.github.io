---
layout: "post"
title: "C++ 继承(inheritance)"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. 基础

继承发生在类和类之间，被继承的类拥有其父类(或：基类，base class)的所有成员变量和成员函数，成为被继承类的一部分。

与组合（composition）加入了"has-a"的关系相比，继承类之间是"is-a"的关系。

# 2. 继承类的构造函数

## 2.1 构造函数调用顺序

如果是一条直线的继承，例如：

![linear inheritance](/assets/img/cpp/linear-inherit.png)

则构造函数的顺序是： A->B->C. 

这是符合逻辑的，因为基类对子类的信息是不可知的，而子类却可以使用基类的信息去影响自己的构造过程。另一方面，子类是基于基类产生的。因此，基类的构造函数在子类之前被调用。

## 2.2 在子类的构造函数中初始化基类中的成员变量

假设基类和子类的声明如下：

{%highlight CPP linenos%}
class Base
{
public:
    int m_nValue;
 
    Base(int nValue=0)
        : m_nValue(nValue)
    {
    }
};
 
class Derived: public Base
{
public:
    double m_dValue;
 
    Derived(int nValue=0, double dValue=0.0);
};
{%endhighlight%}

由于子类继承了基类中所有的成员变量和成员函数，因此想要初始化基类中的成员变量(`m_nValue`)，可能有以下几种错误的做法：

1. 在子类的初始化列表(initialization list)中初始基类的成员变量:

        Derived::Derived(int nValue, double dValue)
                        :m_nValue(nValue)
                        ,m_dValue(dValue)
        {}

    这种做法在C++中是会报错的。C++不允许在初始化列表中初始化继承的成员变量。换句话说，C++的初始化列表中只允许初始化属于自己类的成员变量。

    C++之所以有这样子的设定是因为：当基类中的成员变量(`nValue`)是`const`或引用的时候，如果上述操作是允许的，那么就意味着子类有机会在其构造函数的初始化列表中去改变它的值！

    加上了这个限制之后，C++保证所有的成员变量只被初始化一次。

2. 在子类的构造函数的函数体内改变基类的值：

        Derived::Derived(int nValue, double dValue)
                        ,m_dValue(dValue)
        {
            m_nValue = nValue;
        }

    这样子的做法有两点缺点：

    * 当基类的成员变量是`const`或引用的话，这种方法不可行
    * 基类的成员变量被初始化了两次

正确的做法是：在子类的构造函数的初始化列表中显示地调用基类的构造函数

    Derived::Derived(int nValue, double dValue)
                    :Base(nValue)
                    ,m_dValue(dValue)
    {}

现在，假设实例化对象的时候，例如：`Derived object(1, 1.0);`. 实际的过程如下：

1. 为对象分配空间；
2. 调用`Derived`的构造函数；
3. 编译器检查我们有没有显示地去调用基类的构造函数：
    * 如果没有，则隐式地调用`Base`的默认构造函数
    * 如果有（上例），则调用之
4. `Base`的初始化列表被执行；
5. `Base`的构造函数函数体被执行；
6. `Derived`的初始化列表被执行；
7. `Derived`的构造函数的函数体被执行

Ju一个栗子:

{% highlight CPP linenos %}
#include <iostream> 
#include <string>

class Person
{
    public:
        std::string m_strName;
        int m_nAge;
        bool m_bIsMale;
        
        Person(std::string name = "", int age = 0, bool is_male = true);
        ~Person();
        std::string GetName();
        int GetAge();
        bool IsMale();
};

class FootballPlayer: public Person
{
    public:
        double m_dAverageGoal;
        double m_dAveragePassSuccessRatio;

        FootballPlayer(std::string name = "", int age = 0, bool is_male = true, double ave_goal = 0.0, double ave_pass_success_ratio = 0.0);
        ~FootballPlayer();
        double GetAverageGoal();
        double GetAveragePassSuccessRatio();
        void ShowStatData();
};

/* Member functions of Person */
Person::Person(std::string name, int age, bool is_male)
    :m_strName(name)
    ,m_nAge(age)
    ,m_bIsMale(is_male)
{
    std::cout << "Construction of Person\n";
}

Person::~Person()
{
    std::cout << "Deconstruction of Person\n";
}

std::string Person::GetName()
{
    return m_strName;
}

int Person::GetAge()
{
    return m_nAge;
}

bool Person::IsMale()
{
    return m_bIsMale;
}

/* Member functions of FootballPlayer */
FootballPlayer::FootballPlayer(std::string name, int age, bool is_male, double ave_goal, double ave_pass_success_ratio)
    :Person(name, age, is_male)
    ,m_dAverageGoal(ave_goal)
    ,m_dAveragePassSuccessRatio(ave_pass_success_ratio)
{
    std::cout << "Construction of FootballPlayer\n";
}

FootballPlayer::~FootballPlayer()
{
    std::cout << "Deconstruction of FootballPlayer\n";
}

double FootballPlayer::GetAverageGoal()
{
    return m_dAverageGoal;
}

double FootballPlayer::GetAveragePassSuccessRatio()
{
    return m_dAveragePassSuccessRatio;
}

void FootballPlayer::ShowStatData()
{
    using std::cout;
    cout << "Name                         : " << m_strName << '\n';
    cout << "Age                          : " << m_nAge << '\n';
    cout << "Gender                       : " << ((m_bIsMale)? "M":"F") << '\n';
    cout << "Average Goal                 : " << m_dAverageGoal << '\n';
    cout << "Average passing success ratio: " << m_dAveragePassSuccessRatio*100 << "%\n";
}

/* MAIN */
int main()
{
    FootballPlayer player1("magodo", 27, true, 0.5, 0.8);
    player1.ShowStatData();
}
{% endhighlight %}

输出：

{% highlight CPP linenos %}
➜  basic ./a.out 
Construction of Person
Construction of FootballPlayer
Name                         : magodo
Age                          : 27
Gender                       : M
Average Goal                 : 0.5
Average passing success ratio: 80%
Deconstruction of FootballPlayer
Deconstruction of Person
{% endhighlight %}

需要注意的是，子类只能调用其直接继承自的基类的构造函数，而不能调用基类的基类的构造函数。

# 3. 访问修饰符（public, private, protected）

访问修饰符，可以修饰类中的成员变量和成员函数；也可以修饰子类继承基类时的继承关系。

## 3.1 修饰类中的成员变量和函数

被访问修饰符修饰的类中的成员变量或函数的可访问情况如下表所示：

|---
||类本身|类的实例|类的子类|
|:-:|:-:|:-:|:-:|
|public|o|o|o|
|private|o|x|x|
|protected|o|x|o|
|===

<br>
其中： 

* o: 代表可以访问
* x: 代表不可易访问


例如：

{%highlight CPP linenos%}
class Base
{
    public:
        int m_nPublic;           // 能被 Base, Derived(public/protected继承), Base的对象访问
    protected:
        int m_nProtected;        // 能被 Base, Derieved(public/protected继承) 访问
    private:
        int m_nPrivate;          // 能被 Base 访问
};

class Derived: public Base
{
    public:
        Derived()
        {
            m_nPublic = 1;       // OK
            m_nProtected = 2;    // OK
            m_nPrivate = 3;      // NOK!
        }
}

int main()
{
    Base obj;

    obj.m_nPublic  = 1;          // OK
    obj.m_nProtected = 2;        // NOK!
    obj.m_nPrivate = 3;          // NOK!
}
{%endhighlight%}

## 3.2 修饰子类对于基类的继承关系

在继承基类的时候，可以选择是`public`, `private` 或者 `protected`的继承方式。如果不指定，C++默认是设为`private`的（和类中成员不指定的动作一样）。

* `public`: 基类中成员的修饰符在子类中原样保持；
* `protected`: 基类中成员的修饰符在子类中，`public`改为`protected`，其他原样保持；
* `private`: 基类中成员的修饰符在子类中都是`private`

假设有如下的继承关系：

A->B->C


则对B， B的对象，以及B的子类(C)的影响如下：

|---
|:-:|:-:|:-:|
||**B对A中变量的访问**|**B的实例对继承自A的变量的访问**|**C对B继承自A的变量的访问**|
|**public**|无影响|无影响|无影响|
|**protected**|无影响|可访问A中的public变量->不能访问|无影响|
|**private**|无影响|可访问A中的public变量->不能访问|可访问A中的public/protected变量->不能访问|
|===

实际上，对于一个特定的类，只要记住三点：

1. 类对于自己定义的成员，总是具有访问权限；
2. 子类只能访问它的直属基类，访问其`public`与`protected`的成员；
3. 类的对象只能访问该类的`public`的成员。如果该成员继承自基类，则要依据继承关系确定该成员在这个类中是否为`public`。

# 4. 在子类中加入，重写，隐藏/暴露 成员函数或变量

## 4.1 加入

## 4.2 重写

当类调用某个函数时，先搜索这个类中是否有该函数的实现。如果没有，才会在继承链上寻找基类中该函数的实现。因此，子类可以重写基类中的函数。

在子类改写了的函数中，如果要调用基类里的函数，使用`基类::函数名(...)`(scope resolution operator)来调用。如果，子类中直接调用同名函数，而不加`::`，那它将调用自己。

注意：在子类中重写某个基类中的函数时，基类中的访问修饰符对于子类新定义的函数是没有关联的。也就是说，基类中本来是`public`的函数，在子类中可以被定义为`private`;反过来也一样。

以上这些，对于成员变量也是一样的道理。

## 4.3 隐藏/暴露

C++中有两种方式在子类中隐藏基类中`public`的函数为`private`/`protected`，或者暴露基类中`protected`的函数为`public`：

1. 正如4.2中提到的，子类在重写基类中的函数时可以任意设置该函数的访问修饰符。因此，我们可以定义一个直接调用基类中函数的同名函数，给它设置我们想要的访问修饰符；

2. 子类直接对基类中的函数设置访问修饰符，例如：

    {%highlight CPP linenos%}

    #include <iostream>

    using namespace std;

    class Base
    {
        protected:
            void Echo()
            {
                cout << "Hello\n";
            }
    };

    class Derived: public Base
    {
        public:
            // 设置基类的private函数为public
            // 注意，这里既没有函数返回值，也没有`()`符号。
            Base::Echo;
    };

    int main()
    {
        Derived obj;
        obj.Echo();
    }
    {%endhighlight%}

此外，对于隐藏`public`成员，我们也可以通过在继承的时候使用`protected`/`private`的方式继承。

最后要注意的是，在子类中可以改变访问修饰符的成员一定是基类中子类可以访问的成员，即`protected`/`public`的成员变量。也就是说，子类无法改变基类中`private`的成员的访问修饰符。

以上这些，对于成员变量也是一样的道理。

# 5. 多重继承

## 5.1 多个基类中有相同签名的函数

如果子类调用的函数是继承在它的基类中的，并且它所继承的多个基类中不止一处定义了该函数，则编译器会报错。

   例如：

        #include <iostream>

        class A
        {
            public:
                void Echo()
                {
                    std::cout << "This is A\n";
                }
        };
        class B
        {
            public:
                void Echo()
                {
                    std::cout << "This is B\n";
                }
        };
        class C: public A, public B
        {
        };

        int main()
        {
            C obj;

            obj.Echo();
        }

   会编译报错：

        a.cpp: In function ‘int main()’:
        a.cpp:34:9: error: request for member ‘Echo’ is ambiguous
        a.cpp:21:14: error: candidates are: void B::Echo()
        a.cpp:13:14: error:                 void A::Echo()

   一种workaround是在调用的时候使用`scope resolver`。但是，这种方式会使代码变得难以维护。

## 5.2 菱形继承(diamond inheritance)

如下图所示是所谓的菱形继承关系：

![diamond](/assets/img/cpp/diamond-inheritance.png)

这种菱形继承会引发很多问题，例如： 由于B和C都继承自A，因此他们都保存了一份A的数据。当D调用定义在A中的成员变量或函数时就会产生歧义而报错。形成如下图所示的情况：

![diamond](/assets/img/cpp/diamond-multi-copy.png)

{%highlight CPP linenos%}
#include <iostream>

class A
{
    public:
        void Echo()
        {
            std::cout << "In A\n";
        }
};

class B: public A
{
};
class C: public A
{
};
class D: public B, public C
{
};

int main()
{
    D obj;
    obj.Echo();
}
{%endhighlight%}

编译器报错：

{%highlight CPP linenos%}
diamond.cpp: In function ‘int main()’:
diamond.cpp:32:9: error: request for member ‘Echo’ is ambiguous
diamond.cpp:13:14: error: candidates are: void A::Echo()
diamond.cpp:13:14: error:                 void A::Echo()
{%endhighlight%}


事实上，很多实际问题都可以通过single inheritance就可易解决。很多OOP语言（例如 Smalltalk, PHP）都不支持多重继承。其他的现代语言例如JAVA和C#对于正常的类也只支持single inheritance, 只对接口类允许多重继承。

不过，在有的场合下不可避免的要使用到多重继承。这时候，可以通过virtual继承的方式规避上述的菱形继承问题。

# 6. 虚拟基类(virtual base class)

C++中解决上面提到的“菱形继承”问题的一种方式就是通过“虚拟基类”，使得上例中的D在被创建之后，只存在一份A的数据。

先看一个“菱形继承”的例子：

{%highlight CPP linenos%}

#include <string>
#include <iostream>
#include <math.h>

class Machine
{
    public:
        Machine()
        {
            std::cout << "I'm a machine!\n";
        }
};

class Calculator: public Machine
{
    private:
        double m_ips;
    public:
        Calculator(double ips)
            :Machine()
            ,m_ips(ips)
        {
            std::cout << ips << " \"addition\" per second\n";
        }
};

class StorageDevice: public Machine
{
    private:
        int m_space;
    public:
        StorageDevice(int space)
            :Machine()
            ,m_space(space)
        {
            std::cout << "Storage space: " << space << "GB" << "\n";
        }
};

class PersonalComputer: public Calculator, public StorageDevice
{
    private:
        std::string m_os;
    public:
        PersonalComputer(double ips, int space, std::string os)
            :Calculator(ips)
            ,StorageDevice(space)
            ,m_os(os)
    {
        std::cout << "This is a " << os << "PC\n";
    }
};

int main()
{
    /* Four core 2GHz one adder machine supporting SIMD */
    PersonalComputer pc(pow(20, 9)*4*4, 240, "Linux");
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
➜  virtual_base_class ./a.out 
I'm a machine!
8.192e+12 "addition" per second
I'm a machine!
Storage space: 240GB
This is a LinuxPC
{%endhighlight%}

可见，"Machine"被初始化了两次。

为了使"Machine"的数据只保留一份，需要在"Calculator"和"StorageDevice"继承的时候使用“虚拟基类”的继承方式。只需要在继承类型（例如：`public`）前加上`virtual`即可：

{%highlight CPP linenos%}
class Calculator: virtual public Machine
{
...
}
{%endhighlight%}

这里存在一个问题，就是`PersonalComputer`在实例化的时候，怎样调用`Machine`的构造函数？是通过`Calculator`还是`StorageDevice`？答案是：`PersonalComputer`本身负责对于`Machine`的构造函数的调用。这是一个罕见的调用基类的基类的函数的例子：

{%highlight CPP linenos%}

#include <string>
#include <iostream>
#include <math.h>

class Machine
{
    public:
        Machine()
        {
            std::cout << "I'm a machine!\n";
        }
};

class Calculator: virtual public Machine
{
    private:
        double m_ips;
    public:
        Calculator(double ips)
            :Machine()
            ,m_ips(ips)
        {
            std::cout << ips << " \"addition\" per second\n";
        }
};

class StorageDevice: virtual public Machine
{
    private:
        int m_space;
    public:
        StorageDevice(int space)
            :Machine()
            ,m_space(space)
        {
            std::cout << "Storage space: " << space << "GB" << "\n";
        }
};

class PersonalComputer: public Calculator, public StorageDevice
{
    private:
        std::string m_os;
    public:
        PersonalComputer(double ips, int space, std::string os)
            :Calculator(ips)
            ,StorageDevice(space)
            ,m_os(os)
            ,Machine()    /* 调用基类的虚拟基类的构造函数（少有的跨基类调用函数的情形） 
                           * 这里虽然Machine的构造函数看起来是最后调用的，但是实际上它是被最先调用的，以保证它是在其他非虚拟类之前被构造完成
                           */
    {
        std::cout << "This is a " << os << " PC\n";
    }
};

int main()
{
    /* Four core 2GHz one adder machine supporting SIMD */
    PersonalComputer pc(pow(20, 9)*4*4, 240, "Linux");
}
{%endhighlight%}

输出：

{%highlight CPP linenos%}
➜  virtual_base_class ./a.out        
I'm a machine!
8.192e+12 "addition" per second
Storage space: 240GB
This is a Linux PC
{%endhighlight%}

这里有几个注意点：

1. 虚拟基类的构造函数会先于继承它的非虚拟类的构造（即使它在初始化列表中处在后面，如上例）；
2. 继承虚拟基类的类（例如`Calculator`）在其构造函数中也应该调用虚拟基类的构造函数：
  1. 当创建`Calculator`对象的时候，`virtual`关键字被忽略，会先调用`Machine`的构造函数，再调用`Calculator`的初始化列表和构造函数函数体；
  2. 当创建`PersonalComputer`对象的时候，`Calculator`和`StorageDevice`中对`Machine`的构造函数的调用被忽略，而应该由`PersonalComputer`类负责对`Machine`构造函数的调用；
3. 如果一个类在其依赖链上有多个继承自虚拟基类的基类，那么对于虚拟基类的构造函数的调用，都由最下面的子类来负责。在这个例子中就是`PersonalComputer`。
