---
layout: "post"
title: "\"using declaration\" vs \"using directives\""
categories: blog
published: true
comments: true 
excerpted: |
    This is an answer from StackOverflow...
tags: ['c++']

# Does not change and does not remove 'script' variables
script: [post.js]
---
Let's look at below example:

{% highlight cpp linenos %}
#include <algorithm>

namespace N
{

    template <typename T>
    class C
    {
    public:
        void SwapWith(C & c)
        {
            using namespace std; // (1)
            //using std::swap;   // (2)
            swap(a, c.a);
        }
    private:
        int a;
    };

    template <typename T>
    void swap(C<T> & c1, C<T> & c2)
    {
        c1.SwapWith(c2);
    }

}

namespace std
{

    template<typename T> void swap(N::C<T> & c1, N::C<T> & c2)
    {
        c1.SwapWith(c2);
    }

}
{% endhighlight %}

This code results into error:

    'void N::swap(N::C<T> &,N::C<T> &)' : could not deduce template argument for 'N::C<T> &' from 'int'.

However, if I comment out (1) and uncomment (2), it will compile OK. What is the difference between using namespace std and using std::swap that explains this behavior?

The reason is:

The first case is a __using directive__ (`using namespace X`), and what it means is that 

  _the names from namespace X will be available for __regular lookup__, in the first common namespace of X and the current scope_

In this case, the first common namespace ancestor of `::N` and `::std` is `::`, so the using directive will make `std::swap` available only if lookup hits `::`.

The problem here is that when lookup starts it will look inside the function, then inside the class, then inside N and it will find `::N::swap` there. Since a potential overload is detected, regular lookup does **not** continue to the outer namespace ::. Because ::N::swap is a function the compiler will do __ADL__ (Argument dependent lookup), but the set of associated namespaces for fundamental types is empty, so that won't bring any other overload. At this point lookup completes, and overload resolution starts. It will try to match the current (single) overload with the call and it will fail to find a way of converting from int to the argument ::N::C and you get the error.

On the other hand a __using declaration__ (`using std::swap`) provides the declaration of the entity in the current context (in this case inside the function itself). Lookup will find std::swap immediately and stop regular lookup with ::std::swap and will use it.

