---
layout: "post"
title: "kotlin读书笔记"
categories: "blog"
tags: ['kotlin']
published: true
comments: true
script: [post.js]
excerpted: |
---

* TOC
{:toc}

# class, objects and interfaces

## Basic

### property

- `val`：声明一个read-only的property
- `var`：声明一个mutable的preperty

默认情况下，对于一个property，编译器会做以下几件事情：

- 创建一个字段来存property的值
- 定义getter和setter来返回/更新它的值 (用户可以重写getter/setter)

### interface - methods with default implementations

Kotlin的interface可以包含method的实现，抽象的property声明，但是不能包含状态。基本的定义如下：

{%highlight kotlin%}
interface Clickable {
	val label: String
    fun click()
    fun showOff() = println("I'm clickable")
}
{%endhighlight%}

继承了interface的非抽象类，需要实现interface中未实现的method；对于interface中已经实现了的method，你可以选择重写（override）或者沿用（什么都不用做）。

对于声明的抽象property，意味着继承了interface的类需要提供一种方式可以访问/修改这个property，在interface中并不会分配field来存这个property，因此也就意味着不会有state。

{%highlight kotlin%}
class Button(override val label: String): Clickable {
    override fun click() = println("I was clicked")
}
{%endhighlight%}

当一个类继承了多个interface，并且这些interface都实现了某个同名的method，那么这个类在定义的时候就必须重写这个method。重写的时候，如果你想访问某个interface中的那个实现，可以通过如下的方式：

{%highlight kotlin%}
...
override fun showOff() {
    super<Clickable>.showOff()
}
...
{%endhighlight%}

### open vs final(*)

|---
|Modifier|应用到class|应用到member|
|-|-|-|
|`final`(默认)|不能被继承|不能被子类重写
|`open`|可以被继承|可以被子类重写
|===

例如：

{%highlight kotlin%}
open class RichButton: Clickable {
    override fun click() {} // open (because of overriden)
    fun disable() {} // final
    open fun animate() {} // open explicitly
    ...
}
{%endhighlight%}

如果你重写了parent class/interface的一个member(open)，那么这个新的member也是open的。如果，你希望重写之后这个member为final，那么需要显示地在`override`之前指定`final`:

{%highlight kotlin%}
open class RichButton: Clickable {
    final override fun click() {}
{%endhighlight%}

### abstract class

abstract class有以下几个特点：

1. 不能被实例化
2. 通常会包含abstract members，需要由子类实现，并且这些member默认是`open` (而那些有实现的member则是`final`)

例如：

{%highlight kotlin%}
abstract class Animated {
    abstract fun animate()
}
{%endhighlight%}

### visibility modifiers: public(*), protected, private, internal

|---
|Modifier|class member|top-level declaration(class/function/.etc)
|-|-|-|
|public(default)|visible everywhere|visible everywhere|
|protected|visible in subclass|N/A|
|private|visible in current class|visible in current file|
|internal|visible in current module|visible in current module|
|===

Kotlin要求class或member定义时使用的其他对象的visibility大于等于自身，例如：

{%highlight kotlin%}
internal open class TalkativeButton: Focusable {
    ...
}

fun TalkativeButton.giveSpeech() {
    ...
}
{%endhighlight%}

会报错：`Error: 'public' member exposes its 'internal' receiver type TalkativeButton`。修改的方式可以将`TalkativeButton`定义为`public`，或者将`giveSpeech`定义为`internal`.

值得注意的是，一个class的extension function不能访问其`private`或者`protected`的成员。

### inner class vs nested classe(*)

Kotlin允许在一个类的内部定义类，但是这个内部的类默认情况下不可以访问外部类的成员(`nested`)，除非显示地指定(`inner`)。

|---
|Modifier|Meaning|
|-|-|
|(default)|nested class, doesn't store a reference to outer class
|inner|inner class, sotre a reference to outer class|
|===

示意图如下：

![nest_vs_inner](/assets/img/kotlin/nest_vs_inner.png)

从inner class访问外部类的语法为：`this@Outer`，例如：

{%highlight kotlin%}
class Outer {
    inner class Inner {
        fun getOuterReference(): Outer = this@Outer
    }
}
{%endhighlight%}

### sealed class - defining restricted class hierarchies

举个例子：

{%highlight kotlin%}
interface Expr
class Num(val value: Int): Expr
class Sum(val left: Expr, val right: Expr): Expr

fun eval(e: Expr): Int =
when (e) {
    is Num -> e.value
    is Sum -> eval(e.right) + eval(e.left)
    else -> throw IllegalArgumentException("unknown expression")
}
{%endhighlight%}

上面的例子有两个地方不太智能：

1. 编译器强行要求我们检查`when(e)`的默认分支，但是这个分支如果走到了，我们其实也干不了什么事情，只能抛出一个异常结束
2. 当有一个新的类实现了`Expr`，你必须手动的往`when(e)`中添加分支（前提是你记得这么做）

Kotlin提供了一种叫做`sealed`class的类。当你对某个类指定了`sealed`，那么你就限制了这个类所有可能的子类，这些子类必定是定义在这个类的内部的nested class. 

注意，`sealed`自带`open`属性。

上面的例子可以改成下面这样：

{%highlight kotlin%}
sealed class Expr {
    class Num(val value: Int): Expr()
    class Sum(val left: Expr, val right: Expr): Expr()
}

fun eval(e: Expr): Int = 
when (e) {
    is Expr.Num -> e.value
    is Expr.Sum -> eval(e.right) + eval(e.left)
}
{%endhighlight%}

这样子，当有新的子类定义之后，并且没有在`when(e)`中加入响应的分支，那么编译器就会报错，提醒你需要加入该分支。同时，编译器也不会要求你处理默认分支。示意图如下：

![sealed](/assets/img/kotlin/sealed.png)

`sealed`实际上是一个语法糖，它会为被修饰的类定义一个`private`的构造函数，于是只有定义在`sealed`类内的子类才可以被成功构造（成功调用父类的构造函数）。

注意，当前版本的`sealed`有很多限制，例如：

1. 所有子类都必须是nested class
2. 子类不可以是data class

## Constructor

Kotlin 中存在两种构造函数：

- primary constructor: the main, concise way to initialize a class and is declared outside of the class body
- secondary constructor (one or more): declared in the class body

其中，primary c'tor 至多有一个，secondary c'tor 可以有多个。

### primary c'tor

以下定义了一个类`User`，并且定义了其primary constructor:

{%highlight kotlin%}
class User(val nickname: String)
{%endhighlight%}

其中括号括起来的部分就是primary constructor，注意，它作为类声明header的一部分。它有两个作用：

1. 指定了构造函数的参数
2. 使用构造函数的参数初始化properties

如果使用完整（繁琐）的方式来改写这个定义，如下所示：

{%highlight kotlin%}
class User constructor(_nickname: String) {
    val nickname: String

    init {
        nickname = _nickname
    }
}
{%endhighlight%}

上面的关键字：

- `constructor`: 开启primary或者secondary constructor 的声明
- `init`： 开启一段`initializer block`，这段代码仅在这个类通过实例化的时候被调用（作为primary c'tor的一部分来执行）。可以多次定义，其执行顺序与定义顺序相同。

    这个例子里面，由于初始化过程比较直接，仅仅是赋值，那么你不需要使用`initializer block`，而是直接使用`_nickname`初始化`nickname`即可；如果你不需要改变默认的modifier/annotation(默认为：public)，甚至也可以忽略`constructor`这个关键字：

{%highlight kotlin%}
class User(_nickname: String) {
    val nickname =_nickname
}
{%endhighlight%}

进一步，可以简化为：

{%highlight kotlin%}
class User(val nickname: String)
{%endhighlight%}

构造函数的参数也可以指定默认值：

{%highlight kotlin%}
class User(val nickname: String,
           val isSubscribed: Boolean = true)
{%endhighlight%}

如果某个类继承自一个父类，那么这个类的primary constructor在定义的时候也需要调用调用父类的构造函数:

{%highlight kotlin%}
open class User(val nickname: String) {...}

class TwitterUser(nickname: String): User(nickname) {...}
{%endhighlight%}

如果你希望定义一个不能被外部代码实例化的类（例如想实现一个singleton，当然更好的方式是使用object declaration），需要指定构造函数为private：

{%highlight kotlin%}
class Secretive private constructor() {}
{%endhighlight%}

### secondary c'tor

语法如下：

{%highlight kotlin%}
open class View {
    constructor(ctx: Context) {
    }
}
{%endhighlight%}

如果一个类既有secondary c'tor，又有primary c'tor，那么secondary c'tor必须delegate to primary c'tor（可以使用`this`来引用自身的primary c'tor）:

{%highlight kotlin%}
class Person(val name: String) {
    constructor(name: String, parent: Person) : this(name) {
        parent.children.add(this)
    }
}
{%endhighlight%}

相反地，如果一个类只有secondary c'tor而没有primary c'tor，但是它有父类。那么**每一个**secondary c'tor都必须初始化父类（或者delegate给某一个初始化父类的secondary c'tor）。同样地情况，如果这个类还有`initializer block`，那么在实例化的时候`initializer block`会先于secondary c'tor的执行，例如：

{%highlight kotlin%}
class Constructors {
    init {
        println("Init block")
    }

    constructor(i: Int) {
        println("Constructor")
    }
}
{%endhighlight%}

输出：

	Init block
	Constructor

### no c'tor

如果一个类既没有primary c'tor，又没有secondary c'tor，那么编译器会为其自动生成一个空的primary c'tor. 
