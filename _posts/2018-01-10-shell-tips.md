---
layout: "post"
title: "Shell Tips"
categories: "blog"
tags: ['shell']
published: true
comments: true
script: [post.js]
excerpted: |
    记录Shell编程中的注意点...
---

* TOC
{:toc}

# 解析Argument的顺序

详见：[Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Shell-Expansions.html#Shell-Expansions).

顺序如下：

    brace expansion -> tilde expansion -> parameter/variable expansion -> arithmetic expansion -> command substitution -> word splitting -> filename expansion

在这之后， *syntactic* 的引号被去掉（如果原参数内部有引号，则此时只有 *literal* 的意义）。

* `brace expansion`:
    
        $ echo {a..c}.txt
        a.txt b.txt c.txt

        $ echo th{e,a}n
        then than

* `tilde expansion`:

  一般就是指对`~`的展开（当然也有类似`~+`, `~-`, `~[+/-]N`等用法）。

## filename expansion的注意点

filename expansion有以下几种用法：

|---
|bracket expression|description|
|-|-|
|[XYZ]|match either X, Y or Z|
|[X-Z]|range expression, match all character from X to Z(your current locale defines the order)|
|[[:class:]]|match chars defined by POSIX char class|
|[^...]|negating expression (not portable!)|
|[!...]|same as above|
|[]...] or [-...]|used to include the char `]` and `-` into set|
|[=C=]|match any char that is equivalent to collation weight of C (current local)|
|[[.SYMBOL.]]|matches the collating symbol SYMBOL|
|===

(转自[这里](http://wiki.bash-hackers.org/syntax/pattern))

值得注意的是，所有上述的表达式（除了`[XYZ]`）需要和`[]`同时被展开（同一优先级），否则，如果`[]`内部的表达式先被展开（例如表达式是一个变量），那么这个展开后的表达式会被当作一个字符串，然后整个branchet表达式会被作为`[XYZ]`的形式展开。

例如：

    💤  test  ls
    1  2  3
    💤  test  a=1-3
    💤  test  echo [1-3]
    1 2 3
    💤  test  echo [$a]
    1 3


# 引号

详见：[Bash Guide](http://mywiki.wooledge.org/Quotes)

## 引号的类型

### Shell

* **Single Quotes** (`'...'`): 括起来的内容均为字面值，但不包括`'`本身。
* **Double Quotes** (`"..."`): 括起来的内容均为字面值，除了：以`$`开头的展开，`` `...` `` *command substitution*，以及escape character。

### Bash 扩展

* `$'...'`：括起来的内容均为字面值，除了：escape character 和 escape sequence。
* `$"..."`：关于localization的支持。

## 不使用引号

大部分情况下建议用`"`将 *parameter* 引起来。除了以下几个特例：

1. `[[` 中使用 `=` 作 *parameter* 之间等值比较的时候，如果等号右边的用双引号引起来，就代表字符串的值比较；如果没有，则代表对 *pattern(glob or ext-glob)* 的匹配
2. bash 3.2 及其以后版本的 `[[` 中，要使用正则表达式匹配( `=~` )的时候，建议做法是将表达式赋值给临时变量，然后在`=~`符号右边对该变量进行展开，此时不要用引号！如果用了引号，则进行 `=` 比较。
3. `[[` 是个 *keyword*，它在内部已经做了特殊逻辑，可以保证操作符左右的参数被展开为一个整体，避免了 *word split*。例如：

        $ [ $a = $b ] || echo "not equal"
        -bash: [: too many arguments
        not equal
        $ [[ $a = $b ]] || echo "not equal"
        not equal

4. 需要使用**glob**的时候不应该用括号。
5. 在 *HereDoc* 中希望对变量进行展开的时候， *sential* 不可以用引号括起来。

# 重定向

## `fd>some_where`和`fd>&other_fd` 的区别

* 前者重新创建了一个文件及其文件表项，然后讲指定的fd指定到这个新创建的文件表项。
* 后者称为 *duplicate* , 将`fd`指向`other_fd`的文件表项

## 使用场景

1. 将命令的标准错误和标准输出存在一个文件里:

        $ command 1>some_file 2>&1

  不过这里有个需要注意的点，如果`command`中的输出命令是以 *truncate* 模式打开标准输出/错误，则最终的输出到`some_file`中的内容会被截断。

2. 将命令的标准输出和标准错误通过管道传到另一个命令中去：

        $ { command A; command B; } 2>&1 | other_command

3. 输出到标准错误：

  `echo`默认输出到标准输出，如果想输出到标准错误，有以下几个方法：

  * `$ echo foo >&2`
  * `$ echo foo >>/dev/fs/2  # 比 /dev/stderr 更有兼容性`

4. 暂时将当前终端的标准输出重定向到文件，最后再恢复：

        $ exec 8>&1 1>stdout.txt

        $ # do something...

        $ exec 1>&8 8>&-
  详见[这里](https://stackoverflow.com/questions/25474854/after-using-exec-1file-how-can-i-stop-this-redirection-of-the-stdout-to-file)

## 输入

对于输入重定向或者duplicate，则只需将`>`改为`<`即可。

# 判断变量是否set

详见：[Stack Overflow](https://stackoverflow.com/questions/3601515/how-to-check-if-a-variable-is-set-in-bash)

正确的做法：

{% highlight shell %}
if [ -z ${var+x} ]; then
    : # var is unset
else
    : # var is set
fi
{% endhighlight %}

这里要区别“变量被set”和“变量非空”的区别，利用了参数展开中的`${parameter+word}`，详见[Bash Manual](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html#Shell-Parameter-Expansio://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html#Shell-Parameter-Expansion).

# While Read

详见：[Bash Guide](http://mywiki.wooledge.org/BashFAQ/001)

（一般）的正确做法：

{% highlight shell %}
while IFS= read -r line; do
    # process "$line"
done < "$file"
{% endhighlight %}

* `-r`: 防止`\newline`导致将两行并成一行。例如：
  
  有一个文件*foobar*, 拥有如下内容：

      abc\
      xyz

  如果不加`-r`:

      $ while IFS= read line; do
        >echo "$line"
        >done < foobar
      abcxyz
  
  这显然不是希望的结果，如果加上`-r`，则输出的是:

      abc
      xyz

  所以，用`read`指令的时候，总是要加上`-r`.

* `IFS=`: 如果`IFS`没有被set（shell的默认情况），那么`read`会将每一行的起始部分的空格和结束部分的空格去掉。如果你希望这种行为，则可以将这句忽略。
  
  事实上，`IFS`是作为每一行中不同field的定界符。因为，`read`后面可以跟不止一个参数后面可以跟不止一个参数。

更正确的做法是：

{% highlight shell %}
while IFS= read -r line || [[ -n $line ]]; do
    : # do something
done < "$file"
{% endhighlight %}

这里的区别在于`[[ -n $line ]]`的加入。这是为了防止输入文件不是以`\n`结尾。这种情况下`read`会将最后一行读入并保存在变量中，但是返回false。因此，最后一行没有被处理，所以需要额外判断一下这个情况。

## find and read

正确做法：

{% highlight shell %}
while IFS= read -r -d '' file; do
    # process "$file"
done < <(find <some path> <some condition> -print0)
{% endhighlight %}

* `-print0` in `find`: 默认`find`会使用`-print`将所有的match以`\n`分隔，然后输出到stdout。如果某些文件名包含换行符(e.g.  `touch $'a\nb'`会创建一个名为"a\nb"的文件)，则会导致后续的处理产生错误。
  由于文件名实际上是以C字符串实现的，这意味着NULL字符不会出现在文件名中，因此它可以用于分隔`find`的输出，给后续的其他命令作处理。

* `-d ''` in `read`: 由于`find`以NULL符分隔找到的文件名，因此`read`需要以NULL来分隔这些输入行（即文件名），`-d ''`就是指定这一点的。如果没有这个选项，`read`默认是用`\n`来分隔输入的行的。

# read

`read`有时候可以用于把字符串中以某个特殊字符分隔的部分读入到若干变量或者一个数组中，例如：有一个字符串"1|2|3"，如果想将1，2，3分别保存到a，b，c三个变量中，则可以用以下的方式：

    $ IFS="|" read -r a b c < <(echo -n "1|2|3")

IFS的作用在**Word Splitting**章节提到：

> The shell treats each character of $IFS as a delimiter, and splits the results of the other expansions into words using these characters as field terminators.

值得注意的是，这里不要用herestring, 因为herestring会在最后加上一个换行符，导致`c`会保存了"3\n".

如果想保存到数组，则使用`-a`选项即可：

    $ IFS="|" read -r -a array < <(echo "1|2|3")

# pipe

对于不同的shell，有管道操作符的指令在进程管理上是不一样的：

* 不支持作业控制的shell（例如：sh），其命令中的最后的一个为当前shell的子进程，而执行管道中其他命令的进程则是这个最后进程的子进程。
* 支持作业控制的shell（例如：bash），其命令中所有进程都是当前shell的子进程，并且都在一个进程组内。至于是前台进程组还是后台进程组，取决于这串命令是否跑在前台。并且，它们和当前shell都属于一个会话。

所有在管道命令中的操作都会随着这个命令的结束（进程的结束）而失去作用：

{% highlight shell %}
$ message=Test
$ echo 'Salut, le monde!' | read message
$ echo "The message is: $message"
The message is: Test
$ echo 'Salut, le monde!' | { read message; echo "The message is: $message"; }
The message is: Salut, le monde!
$ echo "The message is: $message"
The message is: Test
{% endhighlight %}

# Command Grouping

详见：[Bash Guide](http://mywiki.wooledge.org/BashGuide/CompoundCommands)

用于将多个命令组合在一起，从而使之称为一个整体作用于当前代码逻辑（例如：重定向）。所有的改动都发生在当前进程，例如：对变量的改动，或者退出进程等。

## 重定向

对于输入的重定向，不但作用于 *command grouping* 中的每一条命令，并且还会作用于其中的 *expansions*：

{% highlight shell %}
$ { echo "$(cat)"; } <<<'foobar'
foobar
$ { "$(</dev/stdin)" <<<"$_"; } <<< 'cat'
foobar
{% endhighlight %}

# 判断值

在shell语言中，逻辑判断都是基于返回值的判断，当返回值为0的时候，认为是成功的；当返回值为非0的时候，认为是失败的。

例如shell默认有两个 **builtin** : `true` 和 `false`，分别在调用后返回0和1:

{% highlight shell %}
$ true && echo "it's true"
it's true
$ if true; then echo "it's true"; fi
it's true
{% endhighlight %}

对于 `if` 语句:

{% highlight shell %}
if <command>; then
    # do something
fi
{% endhighlight %}

其中的 `<command>` 有好几种情况：

1. 直接调用某些命令;
2. `[` built-in，它的返回值取决于后面跟的表达式的结果；
3. `[[` keyword，同上；
4. `((...))` （arithmetic evaluation），它的返回值取决于内部的数学表达式，如果为0，则返回1（也即为false）；如果为非0，则返回0（也即为true）；

# 逻辑运算符 `&&` 和 `||` 有相同的优先级

在C或者Python等语言中，逻辑运算符 `与` 和 `或` 一般是有不同的优先级的，前者优先级高于后者。

在计算一个表达式的逻辑值的时候，是按照如下的顺序：

* 从左到右
* 短路原则

请看下面两个例子：

**Python**

{% highlight python %}
>>> True or print("abc") and print("def")
True
{% endhighlight %}

**C++**

{% highlight cpp %}

#include <iostream>

using namespace std;

bool echoABC()
{
    cout << "abc" <<endl;
    return true;
}

bool echoDEF()
{
    cout << "def" << endl;
    return true;
}

int main()
{
    true || echoABC() && echoDEF();
}

/* 编译运行，输出nothing... */
{% endhighlight %}

以上均为输出任何内容，这是因为根据 *从左到右* 和 *短路* 的评估顺序，由于第一个 `或` 符号左边已经是 *true* 了，因此没有必要评估它右边的表达式。

这里 **右边的表达式** 是指 `与` 符号两边表达式组成的整体。这是因为， *从左到右* 的评估顺序是针对相同优先级的表达式而言的，在这里，以下两个表达式是具有相同优先级的：

* `true`
* `(echoABC() && echoDEF())`

那么，再来看看bash里的情况：

**bash**

{% highlight shell %}
[magodo@t460p test]$ true || echo abc && echo def
def
{% endhighlight %}

输出了 `def`... 这是因为在bash中，`||` 和 `&&` 是有相同的优先级的，因此这里相同优先级的表达式有三个：

* true
* echo abc
* echo def

评估顺序为：第一个 `||` 看到左边是 `true` ，则跳过了右边的表达式 `echo abc` ； `&&` 看到左边的表达式整体为 `true` ，于是再去评估右边的表达式 `echo def` （输出），然后最终返回 0.

