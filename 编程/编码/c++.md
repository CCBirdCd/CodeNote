<a name="pwoyJ"></a>
### 左值引用和右值引用
lvalue（locator value）代表一个在内存中占有确定位置的对象（换句话说就是有一个地址）。 <br />rvalue通过排他性来定义，每个表达式不是lvalue就是rvalue。因此从上面的lvalue的定义，rvalue是在不在内存中占有确定位置的表达式。<br />左值引用: T &<br />右值引用: T &&<br />——————————————————————————————————————————
<a name="rBjyp"></a>
### 引用折叠
我们把 引用折叠 拆解为 引用和 折叠 两个短语来解释。

首先，引用的意思众所周知，当我们使用某个对象的别名的时候就好像直接使用了该对象，这也就是引用的含义。在C++11中，新加入了右值的概念。所以引用的类型就有两种形式：左值引用T&和右值引用T&&。

其次，解释一下折叠的含义。所谓的折叠，就是多个的意思。上面介绍引用分为左值引用和右值引用两种，那么将这两种类型进行排列组合，就有四种情况：

左值-左值 T& &              ->  T&		左值引用<br />左值-右值 T& &&		->  T&		左值引用<br />右值-左值 T&& &		->  T&		左值引用<br />右值-右值 T&& &&	->  T&&		右值引用

**所有的引用折叠**最终都代表一个引用，要么是左值引用，要么是右值引用。<br />规则就是：<br />如果任一引用为左值引用，则结果为左值引用。否则（即两个都是右值引用），结果为右值引用。<br />《Effective Modern C++》<br />相对应的：左值引用类型的函数参数只能传入左值，右值引用类型的函数类型只能传入右值，如下图：参数和函数类型不同的话编译器直接报错<br />![image.png](media/image-20.png)<br />——————————————————————————————————————————
<a name="yDQpD"></a>
### 万能引用
所谓的万能引用并不是C++的语法特性，而是我们利用现有的C++语法，自己实现的一个功能。因为这个功能既能接受左值类型的参数，也能接受右值类型的参数。所以叫做万能引用。

万能引用的形式如下：<br />template<typename T><br />ReturnType Function(T&& parem)<br />{<br />    // 函数功能实现<br />}<br />原文链接：[https://blog.csdn.net/weixin_37910058/article/details/98078614](https://blog.csdn.net/weixin_37910058/article/details/98078614)[<br />](https://blog.csdn.net/weixin_37910058/article/details/98078614)——————————————————————————————————————————
<a name="F0E3I"></a>
### std::move
在C++11中，标准库在<utility>中提供了一个有用的函数std::move，std::move并不能移动任何东西，它唯一的功能是将一个左值强制转化为右值引用，继而可以通过右值引用使用该值，以用于移动语义。从实现上讲，std::move基本等同于一个类型转换：static_cast<T&&>(lvalue);

1.move函数除了把左值强制转为右值之外还有别的作用吗？<br />没有了，不要看它叫move，实际上干的活就只有这点。

2.有人说在move(a)之后就不应该再使用a这个变量了，这是为什么？<br />你都已经强制转化右值了，说明接下来你肯定要使用右值。一般来说使用右值的目的就是将对象移动走，移动后以后只会剩下一具待析构的尸体，一般来说你是不会想使用它的。<br />当然你可以std::move(a)但是不使用右值做移动操作，但是这样就没有使用std::move的意义了。

3.“使用”是指什么，赋值？访问a的值？<br />当你移动对象以后，原来的对象变成一具尸体。语言保证这个尸体可以正常析构。<br />因此，你可以对其重新赋值。<br />一般不要访问a的值，即使你知道那个值不会被移动走。你举的例子中，像int这样的基本类型，std::move对其没有什么影响。因为它就不存在所谓移动操作。单独一个std::move的强制转换为右值只在语法上有意义，汇编中一条代码都不会生成的。

4.move后a不是成了右值了吗？为什么还能赋值成功呢？<br />不是a成了右值，而是std::move(a)是右值<br />	原文链接：https://www.zhihu.com/question/467449795/answer/1960471015

std::move 的函数原型定义:<br />template <typename T><br />typename remove_reference<T>::type&& move(T&& t)<br />{<br />	return static_cast<typename remove_reference<T>::type&&>(t);<br />}<br /> 原型定义中的原理实现:<br /> 首先，函数参数T&&是一个指向模板类型参数的右值引用，通过引用折叠，此参数可以与任何类型的实参匹配（可以传递左值或右值，这是std::move主要使用的两种场景)。关于引用折叠如下：

公式一）X& &、X&& &、X& &&都折叠成X&，用于处理左值<br />string s("hello");<br />std::move(s) => std::move(string& &&) => 折叠后 std::move(string& )<br />此时：T的类型为string&<br />typename remove_reference<T>::type为string <br />整个std::move被实例化如下<br />string&& move(string& t) //t为左值，移动后不能在使用t<br />{<br />    //通过static_cast将string&强制转换为string&&<br />    return static_cast<string&&>(t); <br />}

公式二）X&& &&折叠成X&&，用于处理右值<br />std::move(string("hello")) => std::move(string&&)<br />//此时：T的类型为string <br />//     remove_reference<T>::type为string <br />//整个std::move被实例如下<br />string&& move(string&& t) //t为右值<br />{<br />    return static_cast<string&&>(t);  //返回一个右值引用<br />}

简单来说，右值经过T&&传递类型保持不变还是右值，而左值经过T&&变为普通的左值引用.

②对于static_cast<>的使用注意：任何具有明确定义的类型转换，只要不包含底层const,都可以使用static_cast。

double d = 1;<br />void* p = &d;<br />double *dp = static_cast<double*> p; //正确<br /> <br />const char *cp = "hello";<br />char *q = static_cast<char*>(cp); //错误：static不能去掉const性质<br />static_cast<string>(cp); //正确 

③对于remove_reference是通过类模板的部分特例化进行实现的，其实现代码如下

//原始的，最通用的版本<br />template <typename T> struct remove_reference{<br />    typedef T type;  //定义T的类型别名为type<br />};<br /> <br />//部分版本特例化，将用于左值引用和右值引用<br />template <class T> struct remove_reference<T&> //左值引用<br />{ typedef T type; }<br /> <br />template <class T> struct remove_reference<T&&> //右值引用<br />{ typedef T type; }   <br />  <br />//举例如下,下列定义的a、b、c三个变量都是int类型<br />int i;<br />remove_refrence<decltype(42)>::type a;             //使用原版本，<br />remove_refrence<decltype(i)>::type  b;             //左值引用特例版本<br />remove_refrence<decltype(std::move(i))>::type  b;  //右值引用特例版本 

总结：<br />std::move实现，首先，通过右值引用传递模板实现，利用引用折叠原理将右值经过T&&传递类型保持不变还是右值，而左值经过T&&变为普通的左值引用，以保证模板可以传递任意实参，且保持类型不变。然后我们通过static_cast<>进行强制类型转换返回T&&右值引用，而static_cast<T>之所以能使用类型转换，是通过remove_refrence<T>::type模板移除T&&，T&的引用，获取具体类型T。<br />原文链接：[https://blog.csdn.net/p942005405/article/details/84644069](https://blog.csdn.net/p942005405/article/details/84644069)<br />——————————————————————————————————————————
<a name="IqwOD"></a>
### = delete
1. 禁止使用编译器默认生成的函数,可以将其定义为private，或者使用=delete<br />![image.png](media/image-22.png)<br />2.delete 关键字可用于任何函数，不仅仅局限于类的成员函数<br />![image.png](media/image-21.png)<br />3.模板特化：在模板特例化中，可以用delete来过滤一些特定的形参类型<br />例如：Widget 类中声明了一个模板函数，当进行模板特化时，要求禁止参数为 void* 的函数调用。 <br />1. 如果按照 C++98 的 “私有不实现“ 思路，应该是将特例化的函数声明为私有型，如下所示：<br />![image.png](media/image-23.png)[<br />](https://blog.csdn.net/weixin_37910058/article/details/98078614)问题是，模板特化应该被写在命名空间域 (namespace scope)，而不是类域 (class scope)，因此，该方法会报错。<br />而在 C++11 中，因为有了 delete 关键字，则可以直接在类域外，将特例化的模板函数声明为 delete， 如下所示：[<br />](https://blog.csdn.net/weixin_37910058/article/details/98078614)![image.png](media/image-24.png)[<br />  这样，当程序代码中，有调用 void* 作形参的 processPointer 函数时，则编译时就会报错

](https://blog.csdn.net/weixin_37910058/article/details/98078614)

