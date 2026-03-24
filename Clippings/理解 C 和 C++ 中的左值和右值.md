---
title: Understanding lvalues and rvalues in C and C++ - Eli Bendersky's website
source: https://eli.thegreenplace.net/2011/12/15/understanding-lvalues-and-rvalues-in-c-and-c
author:
  - Eli Bendersky
published: 2001-12-15
created: 2026-03-24
description:
tags:
  - clippings
---
The terms *lvalue* and *rvalue* are not something one runs into often in C/C++ programming, but when one does, it's usually not immediately clear what they mean. The most common place to run into these terms are in compiler error & warning messages. For example, compiling the following with gcc:  
在 C/C++ 编程中， *左值* 和 *右值* 这两个术语并不常见，但一旦遇到，通常很难立即理解它们的含义。最常遇到这些术语的地方是编译器错误和警告信息中。例如，使用 gcc 编译以下代码：

```
int foo() {return 2;}

int main()
{
    foo() = 2;

    return 0;
}
```

You get:您将获得：

```
test.c: In function 'main':
test.c:8:5: error: lvalue required as left operand of assignment
```

True, this code is somewhat perverse and not something you'd write, but the error message mentions *lvalue*, which is not a term one usually finds in C/C++ tutorials. Another example is compiling this code with g++:  
没错，这段代码有点反常，你可能不会写出这样的代码，但错误信息中提到了 *左值（lvalue）* ，这在 C/C++ 教程中并不常见。另一个例子是用 g++ 编译这段代码：

```
int& foo()
{
    return 2;
}
```

Now the error is:  
现在的错误是：

```
testcpp.cpp: In function 'int& foo()':
testcpp.cpp:5:12: error: invalid initialization of non-const reference
of type 'int&' from an rvalue of type 'int'
```

Here again, the error mentions some mysterious *rvalue*. So what do *lvalue* and *rvalue* mean in C and C++? This is what I intend to explore in this article.  
这里，错误信息再次提到了神秘的 *右值（rvalue* ）。那么，在 C 和 C++ 中， *左值（lvalue）* 和 *右值（rvalue）* 究竟是什么意思呢？这就是我将在本文中探讨的内容。

### A simple definition <br>一个简单的定义

This section presents an intentionally simplified definition of *lvalues* and *rvalues*. The rest of the article will elaborate on this definition.  
本节对 *左值* 和 *右值* 给出了一个有意简化的定义。文章的其余部分将对此定义进行详细阐述。

An *lvalue* (*locator value*) represents an object that occupies some identifiable location in memory (i.e. has an address).  
*左值* （ *定位值* ）表示占据内存中某个可识别位置（即具有地址）的对象。

*rvalues* are defined by exclusion, by saying that every expression is either an *lvalue* or an *rvalue*. Therefore, from the above definition of *lvalue*, an *rvalue* is an expression that *does not* represent an object occupying some identifiable location in memory.  
*右值* 是通过排除法定义的，即每个表达式要么是 *左值* ，要么是 *右值* 。因此，根据上述 *左值* 的定义， *右值* 是指 *不* 表示内存中某个可识别位置的对象表达式。

### Basic examples <br>基本示例

The terms as defined above may appear vague, which is why it's important to see some simple examples right away.  
上述术语的定义可能比较模糊，因此，立即查看一些简单的例子非常重要。

Let's assume we have an integer variable defined and assigned to:  
假设我们定义并赋值了一个整型变量：

```
int var;
var = 4;
```

An assignment expects an lvalue as its left operand, and var is an lvalue, because it is an object with an identifiable memory location. On the other hand, the following are invalid:  
赋值操作的左操作数必须是左值，而 var 就是一个左值，因为它是一个具有可识别内存位置的对象。另一方面，以下赋值语句是无效的：

```
4 = var;       // ERROR!
(var + 1) = 4; // ERROR!
```

Neither the constant 4, nor the expression var + 1 are lvalues (which makes them rvalues). They're not lvalues because both are temporary results of expressions, which don't have an identifiable memory location (i.e. they can just reside in some temporary register for the duration of the computation). Therefore, assigning to them makes no semantic sense - there's nowhere to assign to.  
常量 4 和表达式 var + 1 都不是左值（因此它们是右值）。它们不是左值，因为它们都是表达式的临时结果，没有可识别的内存位置（也就是说，它们可能在计算期间驻留在某个临时寄存器中）。因此，给它们赋值在语义上没有意义——因为没有地方可以赋值。

So it should now be clear what the error message in the first code snippet means. foo returns a temporary value which is an rvalue. Attempting to assign to it is an error, so when seeing foo() = 2; the compiler complains that it expected to see an lvalue on the left-hand-side of the assignment statement.  
现在应该很清楚第一个代码片段中的错误信息是什么意思了。 foo 返回一个临时值，它是一个右值。尝试给它赋值是错误的，所以当编译器看到 foo() = 2; 时，它会报错，因为它期望在赋值语句的左侧看到一个左值。

Not all assignments to results of function calls are invalid, however. For example, C++ references make this possible:  
并非所有对函数调用结果的赋值都是无效的。例如，C++ 引用就允许这样做：

```
int globalvar = 20;

int& foo()
{
    return globalvar;
}

int main()
{
    foo() = 10;
    return 0;
}
```

Here foo returns a reference, *which is an lvalue*, so it can be assigned to. Actually, the ability of C++ to return lvalues from functions is important for implementing some overloaded operators. One common example is overloading the brackets operator \[\] in classes that implement some kind of lookup access. std::map does this:  
这里 foo 返回一个引用， *它是一个左值* ，因此可以赋值。实际上，C++ 函数能够返回左值对于实现某些重载运算符至关重要。一个常见的例子是在实现某种查找访问的类中重载方括号运算符 \[\] 。 std::map 的作用如下：

```
std::map<int, float> mymap;
mymap[10] = 5.6;
```

The assignment mymap\[10\] works because the non-const overload of std::map::operator\[\] returns a reference that can be assigned to.  
赋值 mymap\[10\] 之所以有效，是因为 std::map::operator\[\] 的非 const 重载返回一个可以赋值的引用。

### Modifiable lvalues <br>可变左值

Initially when lvalues were defined for C, it literally meant "values suitable for left-hand-side of assignment". Later, however, when ISO C added the const keyword, this definition had to be refined. After all:  
最初在 C 语言中定义左值时，它的字面意思是“适合赋值语句左侧的值”。然而，后来当 ISO C 标准添加了 const 关键字后，这个定义就需要进行细化。毕竟：

```
const int a = 10; // 'a' is an lvalue
a = 10;           // but it can't be assigned!
```

So a further refinement had to be added. Not all lvalues can be assigned to. Those that can are called *modifiable lvalues*. Formally, the C99 standard defines modifiable lvalues as:  
因此，还需要进一步细化。并非所有左值都可以赋值。可以赋值的左值称为 *可修改左值* 。形式上，C99 标准将可修改左值定义为：

> \[...\] an lvalue that does not have array type, does not have an incomplete type, does not have a const-qualified type, and if it is a structure or union, does not have any member (including, recursively, any member or element of all contained aggregates or unions) with a const-qualified type.  
> \[...\] 一个不具有数组类型、不具有不完整类型、不具有常量限定类型的左值，并且如果它是结构或联合体，则没有任何成员（包括递归地，所有包含的聚合或联合体的任何成员或元素）具有常量限定类型。

### Conversions between lvalues and rvalues <br>左值和右值之间的转换

Generally speaking, language constructs operating on object values require rvalues as arguments. For example, the binary addition operator '+' takes two rvalues as arguments and returns an rvalue:  
一般来说，对对象值进行操作的语言结构需要右值作为参数。例如，二元加法运算符 '+' 接受两个右值作为参数，并返回一个右值：

```
int a = 1;     // a is an lvalue
int b = 2;     // b is an lvalue
int c = a + b; // + needs rvalues, so a and b are converted to rvalues
               // and an rvalue is returned
```

As we've seen earlier, a and b are both lvalues. Therefore, in the third line, they undergo an implicit *lvalue-to-rvalue conversion*. All lvalues that aren't arrays, functions or of incomplete types can be converted thus to rvalues.  
正如我们之前看到的， a 和 b 都是左值。因此，在第三行中，它们会进行隐式的 *左值到右值的转换* 。所有非数组、非函数或非不完整类型的左值都可以这样转换为右值。

What about the other direction? Can rvalues be converted to lvalues? Of course not! This would violate the very nature of an lvalue according to its definition.  
反过来呢？右值可以转换为左值吗？当然不行！这违背了左值的定义 。

This doesn't mean that lvalues can't be produced from rvalues by more explicit means. For example, the unary '\*' (dereference) operator takes an rvalue argument but produces an lvalue as a result. Consider this valid code:  
这并不意味着不能通过更明确的方式从右值生成左值。例如，一元运算符 '\*' （解引用）接受一个右值参数，但结果是一个左值。考虑以下有效代码：

```
int arr[] = {1, 2};
int* p = &arr[0];
*(p + 1) = 10;   // OK: p + 1 is an rvalue, but *(p + 1) is an lvalue
```

Conversely, the unary address-of operator '&' takes an lvalue argument and produces an rvalue:  
相反，一元寻址运算符 '&' 接受一个左值参数并生成一个右值：

```
int var = 10;
int* bad_addr = &(var + 1); // ERROR: lvalue required as unary '&' operand
int* addr = &var;           // OK: var is an lvalue
&var = 40;                  // ERROR: lvalue required as left operand
                            // of assignment
```

The ampersand plays another role in C++ - it allows to define reference types. These are called "lvalue references". Non-const lvalue references cannot be assigned rvalues, since that would require an invalid rvalue-to-lvalue conversion:  
在 C++ 中，& 符号还有另一个作用——它允许定义引用类型。这些引用类型被称为“左值引用”。非常量左值引用不能被赋予右值，因为这需要进行无效的右值到左值的转换：

```
std::string& sref = std::string();  // ERROR: invalid initialization of
                                    // non-const reference of type
                                    // 'std::string&' from an rvalue of
                                    // type 'std::string'
```

Constant lvalue references *can* be assigned rvalues. Since they're constant, the value can't be modified through the reference and hence there's no problem of modifying an rvalue. This makes possible the very common C++ idiom of accepting values by constant references into functions, which avoids unnecessary copying and construction of temporary objects.  
常量左值引用 *可以* 被赋予右值。由于它们是常量，其值无法通过引用进行修改，因此不存在修改右值的问题。这使得在 C++ 中非常常见的通过常量引用向函数传递值的惯用法成为可能，从而避免了不必要的复制和临时对象的创建。

### CV-qualified rvalues <br>CV 限定的 r 值

If we read carefully the portion of the C++ standard discussing lvalue-to-rvalue conversions, we notice it says:  
如果我们仔细阅读 C++ 标准中关于左值到右值转换的部分 ，我们会发现它是这样写的：

> An lvalue (3.10) of a non-function, non-array type T can be converted to an rvalue. \[...\] If T is a non-class type, the type of the rvalue is the cv-unqualified version of T. Otherwise, the type of the rvalue is T.  
> 类型为非函数、非数组的左值（3.10）T 可以转换为右值。\[...\] 如果 T 是非类类型，则右值的类型是 T 的 cv 限定版本。否则，右值的类型为 T。

What is this "cv-unqualified" thing? *CV-qualifier* is a term used to describe *const* and *volatile* type qualifiers.  
什么是“cv-unqualified”？ *CV-qualifier* 是一个用来描述 *const* 和 *volatile* 类型限定符的术语。

From section 3.9.3:摘自第 3.9.3 节：

> Each type which is a cv-unqualified complete or incomplete object type or is void (3.9) has three corresponding cv-qualified versions of its type: a *const-qualified* version, a *volatile-qualified* version, and a *const-volatile-qualified* version. \[...\] The cv-qualified or cv-unqualified versions of a type are distinct types; however, they shall have the same representation and alignment requirements (3.9)  
> 每个类型，无论是未限定的完整对象类型、不完整对象类型还是 void 类型（3.9），都有三个对应的限定版本： *常量限定* 版本、 *易失性限定* 版本和 *常量易失性限定* 版本。\[...\] 类型的限定版本和未限定版本是不同的类型；但是，它们必须具有相同的表示和对齐要求（3.9）。

But what has this got to do with rvalues? Well, in C, rvalues never have cv-qualified types. Only lvalues do. In C++, on the other hand, class rvalues can have cv-qualified types, but built-in types (like int) can't. Consider this example:  
但这和右值有什么关系呢？在 C 语言中，右值永远不会有 cv 限定类型，只有左值才有。而在 C++ 中，类右值可以有 cv 限定类型，但内置类型（例如 int ）则不行。请看以下示例：

```
#include <iostream>

class A {
public:
    void foo() const { std::cout << "A::foo() const\n"; }
    void foo() { std::cout << "A::foo()\n"; }
};

A bar() { return A(); }
const A cbar() { return A(); }

int main()
{
    bar().foo();  // calls foo
    cbar().foo(); // calls foo const
}
```

The second call in main actually calls the foo () const method of A, because the type returned by cbar is const A, which is distinct from A. This is exactly what's meant by the last sentence in the quote mentioned earlier. Note also that the return value from cbar is an rvalue. So this is an example of a cv-qualified rvalue in action.  
main 中的第二个调用实际上调用了 A 的 foo () const 方法，因为 cbar 返回的类型是 const A ，它与 A 不同。这正是前面提到的引文中最后一句话的含义。还要注意的是， cbar 的返回值是一个右值。所以这是一个 cv 限定右值的实际应用示例。

### Rvalue references (C++11)<br>右值引用（C++11）

Rvalue references and the related concept of *move semantics* is one of the most powerful new features the C++11 standard introduces to the language. A full discussion of the feature is way beyond the scope of this humble article, but I still want to provide a simple example, because I think it's a good place to demonstrate how an understanding of what lvalues and rvalues are aids our ability to reason about non-trivial language concepts.  
右值引用及其相关的 *移动语义* 概念是 C++11 标准引入的最强大的新特性之一。对该特性的全面讨论远远超出了本文的范围 ，但我仍然想提供一个简单的例子，因为我认为这是一个很好的切入点，可以展示理解左值和右值如何帮助我们更好地理解复杂的语言概念。

I've just spent a good part of this article explaining that one of the main differences between lvalues and rvalues is that lvalues can be modified, and rvalues can't. Well, C++11 adds a crucial twist to this distinction, by allowing us to have references to rvalues and thus modify them, in some special circumstances.  
我刚才用了相当多的篇幅来解释左值和右值的主要区别之一在于左值可以被修改，而右值则不能。C++11 为这种区别增添了一个关键的转折，它允许我们在某些特殊情况下拥有对右值的引用，从而可以对其进行修改。

As an example, consider a simplistic implementation of a dynamic "integer vector". I'm showing just the relevant methods here:  
例如，考虑一个动态“整数向量”的简单实现。这里我只展示相关的方法：

```
class Intvec
{
public:
    explicit Intvec(size_t num = 0)
        : m_size(num), m_data(new int[m_size])
    {
        log("constructor");
    }

    ~Intvec()
    {
        log("destructor");
        if (m_data) {
            delete[] m_data;
            m_data = 0;
        }
    }

    Intvec(const Intvec& other)
        : m_size(other.m_size), m_data(new int[m_size])
    {
        log("copy constructor");
        for (size_t i = 0; i < m_size; ++i)
            m_data[i] = other.m_data[i];
    }

    Intvec& operator=(const Intvec& other)
    {
        log("copy assignment operator");
        Intvec tmp(other);
        std::swap(m_size, tmp.m_size);
        std::swap(m_data, tmp.m_data);
        return *this;
    }
private:
    void log(const char* msg)
    {
        cout << "[" << this << "] " << msg << "\n";
    }

    size_t m_size;
    int* m_data;
};
```

So, we have the usual constructor, destructor, copy constructor and copy assignment operator defined, all using a logging function to let us know when they're actually called.  
因此，我们定义了通常的构造函数、析构函数、复制构造函数和复制赋值运算符 ，所有这些都使用日志函数来让我们知道它们何时被实际调用。

Let's run some simple code, which copies the contents of v1 into v2:  
让我们运行一些简单的代码，将 v1 的内容复制到 v2 中：

```
Intvec v1(20);
Intvec v2;

cout << "assigning lvalue...\n";
v2 = v1;
cout << "ended assigning lvalue...\n";
```

What this prints is:  
打印出来的内容是：

```
assigning lvalue...
[0x28fef8] copy assignment operator
[0x28fec8] copy constructor
[0x28fec8] destructor
ended assigning lvalue...
```

Makes sense - this faithfully represents what's going on inside operator=. But suppose that we want to assign some rvalue to v2:  
这很合理——它忠实地反映了 operator= 内部发生的情况。但假设我们想给 v2 赋一个右值：

```
cout << "assigning rvalue...\n";
v2 = Intvec(33);
cout << "ended assigning rvalue...\n";
```

Although here I just assign a freshly constructed vector, it's just a demonstration of a more general case where some temporary rvalue is being built and then assigned to v2 (this can happen for some function returning a vector, for example). What gets printed now is this:  
虽然这里我只是赋值了一个新构造的向量，但这只是演示一个更一般的情况：先构造一个临时右值，然后将其赋值给 v2 （例如，对于返回向量的函数，这种情况可能会发生）。现在打印出来的结果是：

```
assigning rvalue...
[0x28ff08] constructor
[0x28fef8] copy assignment operator
[0x28fec8] copy constructor
[0x28fec8] destructor
[0x28ff08] destructor
ended assigning rvalue...
```

Ouch, this looks like a lot of work. In particular, it has one extra pair of constructor/destructor calls to create and then destroy the temporary object. And this is a shame, because inside the copy assignment operator, *another* temporary copy is being created and destroyed. That's extra work, for nothing.  
哎呀，这看起来工作量很大。特别是，它额外调用了一对构造函数/析构函数来创建和销毁临时对象。这很可惜，因为在复制赋值运算符内部，又创建并销毁了 *一个* 临时副本。这完全是多此一举。

Well, no more. C++11 gives us rvalue references with which we can implement "move semantics", and in particular a "move assignment operator". Let's add another operator= to Intvec:  
好了，现在不用再这样了。C++11 为我们提供了右值引用，我们可以用它实现“移动语义”，特别是“移动赋值运算符” 。让我们再给 Intvec 添加一个 operator= ：

```
Intvec& operator=(Intvec&& other)
{
    log("move assignment operator");
    std::swap(m_size, other.m_size);
    std::swap(m_data, other.m_data);
    return *this;
}
```

The && syntax is the new *rvalue reference*. It does exactly what it sounds it does - gives us a reference to an rvalue, which is going to be destroyed after the call. We can use this fact to just "steal" the internals of the rvalue - it won't need them anyway! This prints:  
&& 语法是新的 *右值引用* 。它的作用正如其名——提供一个对右值的引用，该引用在调用后将被销毁。我们可以利用这一点“窃取”右值的内部结构——反正它也用不着！以下代码会输出：

```
assigning rvalue...
[0x28ff08] constructor
[0x28fef8] move assignment operator
[0x28ff08] destructor
ended assigning rvalue...
```

What happens here is that our new move assignment operator is invoked since an rvalue gets assigned to v2. The constructor and destructor calls are still needed for the temporary object that's created by Intvec(33), but another temporary inside the assignment operator is no longer needed. The operator simply switches the rvalue's internal buffer with its own, arranging it so the rvalue's destructor will release our object's own buffer, which is no longer used. Neat.  
这里发生的情况是，由于右值被赋给了 v2 ，因此我们新的移动赋值运算符被调用。构造函数和析构函数仍然需要调用，以处理由 Intvec(33) 创建的临时对象，但赋值运算符内部不再需要另一个临时对象。该运算符只是简单地将右值的内部缓冲区与自身的缓冲区交换，从而使右值的析构函数能够释放我们对象自身的缓冲区，因为该缓冲区不再使用。很巧妙。

I'll just mention once again that this example is only the tip of the iceberg on move semantics and rvalue references. As you can probably guess, it's a complex subject with a lot of special cases and gotchas to consider. My point here was to demonstrate a very interesting application of the difference between lvalues and rvalues in C++. The compiler obviously knows when some entity is an rvalue, and can arrange to invoke the correct constructor at compile time.  
我再次强调，这个例子只是移动语义和右值引用的冰山一角。正如你可能已经猜到的，这是一个复杂的主题，有很多特殊情况和陷阱需要考虑。我在这里的目的是展示 C++中左值和右值区别的一个非常有趣的应用。编译器显然知道某个实体何时是右值，并且可以在编译时调用正确的构造函数。

### Conclusion <br>结论

One can write a lot of C++ code without being concerned with the issue of rvalues vs. lvalues, dismissing them as weird compiler jargon in certain error messages. However, as this article aimed to show, getting a better grasp of this topic can aid in a deeper understanding of certain C++ code constructs, and make parts of the C++ spec and discussions between language experts more intelligible.  
人们可以编写大量 C++ 代码而不去关注右值和左值的问题，甚至将它们视为某些错误信息中晦涩难懂的编译器术语。然而，正如本文旨在阐明的那样，更好地理解这一主题有助于更深入地理解某些 C++ 代码结构，并使 C++ 规范的部分内容以及语言专家之间的讨论更容易理解。

Also, in the new C++ spec this topic becomes even more important, because C++11's introduction of rvalue references and move semantics. To really grok this new feature of the language, a solid understanding of what rvalues and lvalues are becomes crucial.  
此外，在新版 C++ 规范中，由于 C++11 引入了右值引用和移动语义，这个主题变得更加重要。要真正理解这门语言的这一新特性，对右值和左值的概念有透彻的理解至关重要。

|     | rvalues can be assigned to lvalues explicitly. The lack of implicit conversion means that rvalues cannot be used in places where lvalues are expected.   右值可以显式地赋值给左值。由于缺乏隐式转换，因此不能在需要左值的地方使用右值。 |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |

|  | That's section 4.1 in the new C++11 standard draft.   这是新版 C++11 标准草案的第 4.1 节。 |
| --- | --- |

|  | You can find a lot of material on this topic by simply googling "rvalue references". Some resources I personally found useful: [this one](http://www.artima.com/cppsource/rvalue.html), [and this one](http://stackoverflow.com/questions/5481539/what-does-t-mean-in-c0x), and [especially this one](http://thbecker.net/articles/rvalue_references/section_01.html).   只需在谷歌上搜索“右值引用”，就能找到很多关于这个主题的资料。我个人觉得以下几个资源很有用： [这个](http://www.artima.com/cppsource/rvalue.html) 、 [这个](http://stackoverflow.com/questions/5481539/what-does-t-mean-in-c0x) ， [尤其是这个](http://thbecker.net/articles/rvalue_references/section_01.html) 。 |
| --- | --- |

|  | This a canonical implementation of a copy assignment operator, from the point of view of exception safety. By using the copy constructor and then the non-throwing std::swap, it makes sure that no intermediate state with uninitialized memory can arise if exceptions are thrown.   从异常安全的角度来看，这是一个规范的复制赋值运算符实现。通过使用复制构造函数以及不抛出异常的 std::swap ，可以确保即使抛出异常，也不会出现未初始化内存的中间状态。 |
| --- | --- |

|  | So now you know why I was keeping referring to my operator= as "copy assignment operator". In C++11, the distinction becomes important.   现在你应该明白我为什么一直把 operator= 称为“拷贝赋值运算符”了吧。在 C++11 中，这种区别变得非常重要。 |
| --- | --- |