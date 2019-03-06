---
layout: "post"
title: "Dart 之 类型系统"
categories: "blog"
tags: ['dart']
published: true
comments: true
script: [post.js]
---

* TOC
{:toc}

以下内容为阅读官网[dart's type system](https://www.dartlang.org/guides/language/sound-dart)之后的小结。

# 类型推导

Dart是强类型系统的语言，除了提供编译时的类型检查以外，Dart还可以像大部分现代语言一样，做类型推导。其推导范围包括：

- 实例成员（变量/函数）
- 类成员（变量/函数）
- 自动变量
- 泛型的类型

以下分别进行讨论。

## 实例成员（变量/函数）

1. 如果某个类的实例成员继承了父类，那么它**重载**的成员的类型可以被推导:

    ```dart
    class Foo {
        int a;
        String Greet() =>"Hello Foo!";
    }

    class Bar extends Foo{
        // override instance variable 
        var a;

        // override instance method 
        @override
        Greet() => "Hello Bar";

        // new method
        Baz() => "Hello Baz";
    }

    typedef RetString = String Function();
    typedef RetDynamic = String Function();

    void main() {
        var bar = Bar();
        assert(bar.a is int);
        assert(bar.Greet is RetString);
        assert(bar.Baz is! RetString);
        print("${bar.Baz.runtimeType}");
    }
    ```
    输出：

    ```shell
    () => dynamic
    ```

2. 如果某个类的实例的成员有初始化(无论是直接初始化或者是在构造函数中)，那么可以根据该初始化的上下文进行推导:

    ```dart
    class Foo {
        var a;
        var b = "asdf";

        Foo():
            a = 1;
    }

    void main() {
        var f = Foo();
        assert(f.a is int);
        assert(f.b is String);
    }
    ```
## 类成员（变量/函数）

根据初始化表达式来推导。

## 自动变量

根据初始化表达式来推导（不会将初始化以后的操作考虑进去）：

    ```dart
    var x = 3; // x is inferred as an int
    x = 4.0; // error
    ```

## 泛型

泛型类型的构造和泛型函数的执行可以明确指定类型，那么Dart会根据：

>  downward information from the context of occurrence, and upward information from the arguments to the constructor or generic method

进行类型推导：

    ```dart
    var l = [3.0]; // <double>[]
    var ints = l.map((x) => x.toInt()); // Iterable<int>
    ```

> In the last example, x is inferred as double using downward information. The return type of the closure is inferred as int using upward information. Dart uses this return type as upward information when inferring the map() method’s type argument: <int>.

(好吧，我也没看懂...)

# 替换类型

这章节主要设计两个内容：一个是在各种表达式中的类型自动转换规则；另一个是函数重载时候的类型修改规则。

它们都遵循一条规则：

> You can replace a consumer’s type with a supertype and a producer’s type with a subtype.

其中：

>  A consumer absorbs a type and a producer generates a type.

用更简单的方式来说就是：**consumer >= producer**

你可以做如下类比。由于父类比子类所表现的范围更大，类比成父类成本高于子类。而消费者总是愿意接收更贵的东西，生成者则抵制更高的成本。因此，消费者愿意接收supertype，生产者愿意接收subtype. （很生硬🥶）

（事实上，很多OO语言都是这样的，例如：kotlin）

## 自动类型转换

考虑如下类结构：

![class](/assets/img/dart_system/class.png)

那么:

```dart
// 普通赋值
Animal c = Cat(); // OK: 左边是consumer，右边是producer，consumer >= producer
Cat c = Animal(); // NOK: 左边是consumer，右边是producer，consumer < producer

// 泛型赋值
List<Cat> myCats = List<MaineCoon>(); // OK
List<Cat> myCats = List<Animal>(); // NOK (runtime error)

// 函数调用
// 对于输入参数：调用方是producer, 实现方是consumer
// 对于返回参数：调用方是consumer, 实现方是producer
void chase(Animal a) {}
chase(Cat()); // OK
Animal getAnimal() {return Cat();} // OK
```

## 函数重载

从调用者的角度看，输入参数是consumer，返回参数是producer. 对于重载的函数，其输入参数的类型可以比父类的更upward；其返回参数的类型可以比父类的更downward.

```dart
class Animal {
    void chase(Animal a){}
    Animal get parent => Animal();
}

class HoneyBadger extends Animal {
    @override
    void chase(Object a) {} // consumer (input argument) upgrade
    @override
    HoneyBadger get parent => HoneyBadger(); // producer (return type) downgrade
}
```
