---
layout: "post"
title: "C++11 auto && decltype"
categories: blog
tags: ['c++']
published: true
comments: true 
script: [post.js]
---

* TOC
{:toc}

# 1. auto

auto的引入可以让编译器自动推断某个变量的类型，减少程序员的输入。例如以下代码：

    vector<int> v;

    for (vector<int>::iterator it = v.begin(); it != v.end(); ++it)
    { *it >> cout; }

有了`auto`后可以写成:

    vector<int> v;

    for (auto it = v.begin(); it != v.end(); ++it)
    { *it >> cout; }

因为，编译器是可以根据`v.begin()`来推断出`it`的类型，而不需要用户手动指定。

## 1.1 auto 的推断过程

考虑以下例子：

    int x = int{};
    const int& crx = x;

    auto something = crx;

`something`的类型并不是`const int&`, 而是`int`. 因为`auto`通过以下步骤推断一个表达式的类型：

1. 如果表达式是一个引用，引用被去掉；
2. 如果在第1步以后，有**top-level**的 `const` and/or `volatile`, 也被去掉。

注意第二步里的**top-level**, 这往往是从右往作读(top->low)，例如：

    int i{};
    const int *ci1 = &i; // has low-level const, don't ignore
    int *const ci2 = &i; // has top-level const

    auto something1 = ci1; // const int*
    auto something2 = ci2; // int*

另外，以上的这个推断的过程和模板函数对输入参数的推断过程是一样的，除了：`auto`可以额外推断`std::initializer_list`类型，而模板函数不可以。

所以，同样地情况在函数推断中：

    template<class T>
    void foo(T arg);

    foo(crx);

`crx`的类型也被推导为`int`. 

注意，以上两条是对于没有修饰符时`auto`的推断过程。

## 1.2 有修饰符的auto的另类推断过程

在上面的模板函数的例子中，如果我们希望实际的输入函数的类型被推导为`const int&`. 有以下两种写法：

    // 在调用的时候显示指出
    foo<const int&>(crx);

    // 或者在声明模板函数时指出
    template<class T>
    void foo(const T& arg);

后者同样可以用于`auto`，通过加入修饰符来指定推断出的类型是`const T&`:

    const auto& something = crx;

于是，`something`这次被推断为`const auto&`.

接下来需要注意，加入修饰符之后的`auto`的推断过程有两点需要注意:

1. `auto&`，表达式为`const`

    例如：

        const int c = 0;
        auto& ac = c;

    这个例子中的`ac`的类型是`const int&`。这和之前说的第二步中的去掉const不同，如果这里把const丢掉的话，`ac`就可以修改一个`const`变量了，因此语言设计时这种情况中`const`被保留了。

2. `auto&&`，表达式为lvalue or rvalue

    例如：

        int i = 123;
        auto&& ri_1 = i;    // lvalue
        auto&& ri_2 = 123;  // (p)rvalue

    这种情况下的推断过程如下：

    * 如果表达式是lvalue, 首先进行正常的推断过程(去掉reference和const/volatile)，然后在最终的类型上加上一个引用(&);
    * 如果表达式是rvalue，仅进行正常的推导过程。
    
    因此，对于以上两个例子：

    1. auto&& ri_1 = i;

        首先，进行无修饰符的推导，得到的结果是`int`，然后加上一个引用，算上指定的rvalue reference，我们得到了: `auto& &&`. 根据*reference collapsing rules*, `& &&` -> `&`. 所以，结果是`int&`.

    2. auto&& ri_2 = 123;

        首先，进行无修饰符推导，得到`auto`的结果`int`. 因此，最终的结果是`int&&`. 如果想知道这里这么推导的原因，可以查询*universal referene*或者*forwarding reference*.

## 1.3 设计理念

`auto`之所以丢掉`&`，是因为即使有一个引用来推断变量的类型，不代表我们实际要的类型是引用（大概率并不是），因此不要`&`; `top-level const`也是同理。

# 2. decltype

`auto`可以在编译器能够推断出某个变量的类型的时候用于作为类型来声明那个变量，并且仅此而已。而实际上，很多非变量声明的情况下，编译器在也可以推断出某个希望的类型，例如下面的例子中，希望定义一个由输入参数类型决定的类型:

    template<typename T, typename S>
    void foo(T lhs, S rhs)
    {
        using product_type = ???(lhs * rhs);
    }

在C++11以前，某些非编译器提供了一些非标准的方法实现了以上的`???`（例如：`typeof`）.

为了提供一个统一的，标准的`typeof`,C++11实现了`decltype`。

另一个`decltype`的适用场景为，希望编译器自动推断出模板函数的返回值，用法如下：

    tempalte<typename T, typename S>
    auto multiply(T lhs, S rhs) -> decltype(lhs * rhs)
    { return lhs * rhs; }

可能有人会错误地写成如下形式：

    template<typename T，typename S>
    decltype(lhs*rhs) multiply(T lhs, S rhs)
    { return lhs * rhs; }

这是不能通过编译的，原因是`lhs` 和 `rhs`在函数名声明之前是不存在的。

至此，很多人可能会有如下认识：

> `decltype` 和 `auto` 一样，差别仅在于前者适用的范围更广。

这实际上是错误的!

## 2.1 decltype 推断过程：情景1 (simple expression)

> 当`decltype(expr)`中的`expr`是一个不带括号的变量，函数参数，或者类成员变量，那么`decltype(expr)`是那个变量，函数参数，或者类成员变量在源代码中声明的变量。

以下举了几个例子：

    struct S
    {
        S(): m_x{42} {}
        int m_x;
    };

    int x;
    const in cx = 42;
    const int& crx = x;
    const S* p = new S();

    typedef decltype(x) x_type; // x_type = int
    auto a = x;                 // a is int

    typedef decltype(cx) cx_type;   // cx_type = const int
    auto b = cx;                    // b is int

    typedef decltype(crx) crx_type; // crx_type = const int&
    auto c = crx_type;              // c is int

    typedef decltype(p->m_x) m_x_type; /* m_x_type = int
                                        * 虽然，p是low-level const指针，但是decltype推导的是
                                        * 成员变量声明的类型，因此就是int */
    auto d = p->m_x;    // d is int

## 2.2 decltype 推断过程: 情景2（complex expression）

情景2适用于情景1中提到的三种情况外的的所有情况。不过，要知道它的类型推断过程，需要首先了解`lvalue`, `xvalue`, `prvalue`:

* `lvalue`: 不可移动的对象，其对立面为`rvalue`(即可移动的对象)
* `xvalue`: `rvalue`中的一种，含有 *identity* （有名右值引用）。包括：
    
    * 返回值声明为`rvalue reference`的函数表达式；
    * 类型转换为`rvalue reference` (e.g. `static_cast<A&&>(a)`)
    * 一个`xvalue`对象的成员变量 (e.g. `(static_cast<A&&>(a)).m_x`)

* `prvalue`: `rvalue`中除了`xvalue`的。

接下来是`decltype`对于complex expression的推导过程：

> 如果 expr 的类型是 T. 当 expr 是 lvalue, decltype(expr) = "T&"; 当 expr 是 xvalue, decltype(expr) = "T&&"; 当 expr 是 prvalue, decltype(expr) = "T".

以下给出一些例子：

    /* Parenthesed Expressions */

    struct S
    {
        S():m_x{42} {}
        int m_x;
    };

    int x;
    const int cx = 42;
    const int& crx = x;
    const S* p = new S();

    using x_with_parens_type = decltype((x)); // int&

    using cx_with_parens_type = decltype((cx)); // const int&

    using crx_with_parens_type = delctype((crx)); /* const int& + &
                                                   * according to C++11 reference collapsing rules,
                                                   * this makes no difference. Hence, const int& */

    using m_x_with_parens_type = decltype((p->m_x)); //  ??? 这个怎么是 const int&

    /* Function Expressions */

    const S foo();
    using foo_type = decltype(foo()); /* since foo() is a prvalue, decltype doesn't add
                                       * a reference. Hence, foo_type is "const S" */

    const int& foobar();
    using foobar_type = decltype(foobar()); /* since foobar() is a lvalue, decltype adds a reference.
                                             * according to C++11 reference collapsing rules,
                                             * this makes no difference. Hence, const int& */

    std::vector<int> v{42,43};
    using iterator_type = decltype(v.begin()); /* since v.begin() is a prvalue, no reference is added.
                                                * Hence std::vector<int>::iterator. */
    using ele_type = decltype(v[0]); /* since v[0] is a lvalue(int&), add one reference. 
                                      * according to c++11 reference collapsing rules, makes no difference.
                                      * hence int& */

    /* Binary or Ternary Expressions */

    int x{0};
    int y{0};
    const int cx{42};
    const int cy{43};
    double d1{3.14};
    double d2{2.72};

    using prod_xy_type = decltype(x*y); /* since x * y is a prvalue, no reference is added. Hence int. */
    using prod_cxcy_type = decltype(cx*cy); /* since cx*cy is prvalue(int, not const int!),
                                             * no reference is added. Hence int. */
    using cond_type = decltype(d1 < d2? d1:d2); /* since the expression is lvalue(double), decltype
                                                 * will add a reference. Hence double&. */
    using cond_type_promotion = decltype(x < d1? x:d1); /* since the expression is evaluated to be
                                                         * (double)(x), which means x is promoted to double,
                                                         * in which case a temporary will be created. This
                                                         * temporary is a prvalue. So decltype adds no ref.
                                                         * Hence double. */

# 引用

[1] [thbecker: "auto and decltype"](http://thbecker.net/articles/auto_and_decltype/)

