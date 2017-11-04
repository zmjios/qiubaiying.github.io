# C++ 右值引用笔记

这篇文章的原文在此[what are move?](http://stackoverflow.com/questions/3106110/what-are-move-semantics)

我发现理解move语义最简单的方式是看一个样例，让我们从持有一块动态分配的内存的简单string类型开始：


```

#include <cstring>
#include <algorithm>

class string
{
    char* data;

public:

    string(const char* p)
    {
        size_t size = strlen(p) + 1;
        data = new char[size];
        memcpy(data, p, size);
    }
}

```
既然我们要自己管理内存，那我们就应该遵守那三条原则（[the rule of three](http://stackoverflow.com/questions/4172722/what-is-the-rule-of-three)），如果你不知道什么三条原则，去c++ 98标准里查找。下面我将推迟赋值运算符的实现，先来实现复制构造函数和析构函数：


```
    ~string()
    {
        delete[] data;
    }

    string(const string& that)
    {
        size_t size = strlen(that.data) + 1;
        data = new char[size];
        memcpy(data, that.data, size);
    }

```

复制构造函数定义了怎样复制一个string对象。参数const string& that 可以为想要复制string的以下例子中的任何一种形式：

```
1 string a(x);                                    // line 1
2 string b(x + y);                                // line 2
3 string c(some_function_returning_a_string());   // line 3

```

现在我们开始分析move语义。你会发现，只有第一行(line 1)的x深度拷贝是有必要的，因为我们可能会在后边用到x，如果x改变了，我们会很奇怪。你有没有注意到我刚刚把x说了三遍（如果包括这次的话，是四遍），每一遍都是说的同一个对象？我们把x的这种表达式叫做左值(lvalues)。

第二行和第三行的参数就不是左值而是右值，因为表达式产生的string对象是匿名对象，之后没有办法再使用了。右值就是指在下一个分号（更准确的说是在包含右值的完整表达式的最后）销毁的临时对象。这一点非常重要，因为我们可以在b和c的初始化过程中，对源string对象（参数）做任何想要做的事情，并不让用户感觉到。

C++ 11引入了一种新的机制叫做“右值引用”，以便我们通过重载直接使用右值参数。我们所要做的就是写一个以右值引用为参数的构造函数。在这个函数的内部，我们对参数所指向的对象做任何事情，只要我们保持他的合理性：


```
string(string&& that)   // string&& is an rvalue reference to a string
{
    data = that.data;
    that.data = 0;
}

```

我们在这里是怎么做的呢？我们没有深度拷贝堆内存中的数据，而是仅仅复制了指针，并把源对象的指针置空。事实上，我们“偷取”了属于源对象的内存数据。再次，问题的关键变成：无论在任何情况下，都不能让客户觉察到源对象被改变了。在这里，我们并没有真正的复制，所以我们把这个构造函数叫做“转移构造函数”（move constructor, 不知道译法是否确切），他的工作就是把资源从一个对象转移到另一个对象，而不是复制他们。

祝贺你，你现在对move语义有了基础的理解，我们继续来进行赋值操作符的实现。如果你不理解copy and swap惯用法，先去学习 一下，然后再回来，因为这是c++异常安全的一个非常精彩的惯用法。


```

string& operator=(string that)
    {
        std::swap(data, that.data);
        return *this;
    }
};

```

呃，就这些？右值引用在哪里？你可能会这样问。我的答案是：在这里，我们不需要

注意到我们是直接对参数that传值，所以that会像其他任何对象一样被初始化，那么确切的说，that是怎样被初始化的呢？对于C++ 98，答案是复制构造函数，但是对于C++ 11，编译器会依据参数是左值还是右值在复制构造函数和转移构造函数间进行选择。

如果是a=b，这样就会调用复制构造函数来初始化that（因为b是左值），赋值操作符会与新创建的对象交换数据，深度拷贝。这就是copy and swap 惯用法的定义：构造一个副本，与副本交换数据，并让副本在作用域内自动销毁。这里也一样。

如果是a = x + y，这样就会调用转移构造函数来初始化that（因为x+y是右值），所以这里没有深度拷贝，只有高效的数据转移。相对于参数，that依然是一个独立的对象，但是他的构造函数是无用的（trivial），因此堆中的数据没有必要复制，而仅仅是转移。没有必要复制他，因为x+y是右值，再次，从右值指向的对象中转移是没有问题的。

总结一下：复制构造函数执行的是深度拷贝，因为源对象本身必须不能被改变。而转移构造函数却可以复制指针，把源对象的指针置空，这种形式下，这是安全的，因为用户不可能再使用这个对象了。 

我希望这个例子抓住了要点，为了简单起见，关于右值引用和move语义的很多内容我都故意略过了。


剩余译文在这里[深入理解C++ move语义](http://www.cnblogs.com/tingshuo/archive/2013/01/22/2871328.html)

