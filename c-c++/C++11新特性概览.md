# C++11新特性概览

------
##### 预定义宏
\__STDC_HOSTED__ , \__STDC__ , \__STDC_VERSION__ , \_STDC_ISO_10646__

------
##### _Pragma__操作符
\#pragma once 等价于 _Pragma("once")

------
##### ____func____
返回所在函数的名称，在结构体构造函数中也可正常使用
```C++
const char * hello() { return __func__; }
struct Test{
    Test() : name(__func__){}
    const char * name;
};
```

------
##### __static_assert__
编译时断言

------
##### ____VA_ARGS____宏
在宏定义的实现部分替换可变参数宏的省略号所代表的字符串
```c++
#define PR(...) printf(__VA_ARGS__)
#define LOG(...) {\
    fprintf(stderr,"%s:Line:%d\t",__FILE__,__LINE__);\
    fprintf(stderr,__VA_ARGS__);\
    fprintf(stderr,"\n");\
}
```

------
##### __noexcept__ & __noexcept(constexpr)__
constexpr:常量表达式，false:函数会抛出异常，true：函数不会抛出异常
```c++
int max(int a,int b) noexecpt
{
    return a > b ? a : b;
}

int max(int a,int b) noexecpt(true)
{
    return a > b ? a : b;
}
```
noexcept可用于模板
noexcept性能比普通的抛出异常更好，因为异常机制有额外的开销，比如函数抛出异常时会导致函数的栈诊被依次展开（unwind），并调用本帧中已构造的自动变量的析构函数

------
##### __long long int__
long long int至少64位
c++11规定了5中标准的整形 signed char，signed short，signed int，signed long，signed long long，并规定这每种整形都有一个与之对应的unsigned类型
c++11支持非标准整形扩展（例如某些嵌入式系统中使用48位整形），不限自定义整形的长度，但是规定signed类型和unsigned类型必须一样长
自定义整形的等级比标准整形低（隐式类型转换时用到）
```c++
#include <climits>
#include <cstdio>
#include <iostream>
#include <bitset>

using namespace std;

int main()
{
    cout << "int min:"<< INT_MIN << ",int max:" << INT_MAX << endl;
    cout << "unsigined int max:" << UINT_MAX << endl;
    cout << "long int min:" << LONG_MIN << ",long int max:" << LONG_MAX << endl;
    cout << "unsigned long int max:" << ULONG_MAX << endl;
    cout << "long long int min:" << LLONG_MIN << ",long long int max:" << LLONG_MAX << endl;
    cout << "usigned long long int max:" << ULLONG_MAX << endl;
    cout << "usigned long long int max hex:" << std::hex << ULLONG_MAX << endl;
    cout << "usigned long long int max binary:" << bitset<128>(ULLONG_MAX) << endl;
    return 0;
}
```
----
##### __快速初始化成员变量__
c++11支持=，{}就地初始化类成员变量
```c++
#include <iostream>
#include <string>

using namespace std;

class Mem
{
public:
    Mem(){ cout << "x:" << x << " ,y:" << y << endl; }
    Mem(int x1,float y1):x(x1),y(y1)
    {
        cout << "x:" << x << " ,y:" << y << endl;
    }
private:
    int x = 1;
    float y = 1.1;
};

class TestInitClassMem
{
public:
    TestInitClassMem()
    {
        cout << "a:" << a << " ,b:" << b << " ,s:" << s << endl;
        cout << "=============================" << endl;
    }
    TestInitClassMem(int a1,double b1,string s1):a(a1),b(b1),s(s1)
    {
        cout << "a:" << a << " ,b:" << b << " ,s:" << s << endl;
        cout << "=============================" << endl;
    }
private:
    int a = 0;
    double b {0.1};
    string s{"hello"};
    //string s("hello"); //compile fail
    //static int si = 10; //compile fail
    Mem m2;
    Mem m1{2,2.2};
    //Mem m1(2,2.2); //compile fail
};

int main()
{
    TestInitClassMem test_mem1;
    TestInitClassMem test_mem2(100,100.1,"linux");
    return 0;
}

输出：
x:1 ,y:1.1
x:2 ,y:2.2
a:0 ,b:0.1 ,s:hello
=============================
x:1 ,y:1.1
x:2 ,y:2.2
a:100 ,b:100.1 ,s:linux
=============================
结论：
1.就地初始化执行早于构造函数初始化列表
2.普通静态成员不可直接就地初始化
```

-----
##### 非静态成员sizeof
非静态成员可以直接使用sizeof求大小，如：sizeof(student.score)

-----
##### 扩展的friend语法
friend声明友元类时不再需要class关键字，且可以使用type'de'f别名
```c++
class LiLei;
typedef LiLei Lee;
template<typename T>
class HanMeiMei
{
public:
    ......
    friend LiLei;
    friend T;                 //可以使用模板了！！！
    //friend class LiLei;     //OK
    //friend Lee;             //OK
private:
    string name;
};
```
C++11支持不带class关键字的friend友元类声明使得友元类可以应用在模板上，特别的是**如果模板类是原生类型如int，long等则编译器会默默地将其实例化为一个不带友元类的普通类**

-----
##### final & override
final：使用final声明的类虚成员函数禁止被派生类重写
override：派生类中使用override声明的基类虚函数必须被重写
```c++
#include <iostream>

using namespace std;

class Base
{
public:
    virtual void print() = 0;
    virtual void func() = 0;
};

class DerivedA : public Base
{
public:
    virtual void print() final { cout << "DerivedA print() function" << endl; }
    virtual void func() { cout << "DerivedA func()" << endl; }
};

class DerivedB : public DerivedA
{
public:
    //void print() { cout << "DerivedB print() function" << endl; } //complie fail
    void print(int a) { cout << "DerivedB print(int a) functoin" << endl; }
    //void func() override; //compile fail
    void func() override { cout << "DerivedB override func()" << endl; } //must be implemented
};

int main()
{
    DerivedB d;
    d.print(10);
    d.func();
    return 0;
}
```

-----
##### 默认模板参数
C++98支持类模板的默认模板参数，但不支持函数模板的默认模板参数，C++11新增了对函数模板的默认模板参数的支持，类模板的默认模板参数需遵循“从右至左”原则，而函数模板则不需要
```c++
template<typename T = int> class Test {}; //ok
template<typename T = int> void func(){} //c++98 error,C++11 ok
template<typename T,typename U = double> class Test{}; // ok
template<typename T = int,typename U> class Test{}; //error,未遵循从右至左
template<typename T,typename U = double> void func(){} //ok
template<typename T = int,typename U> void func(){} //ok,函数模板不需要遵从右至左原则

```

-----
##### 外部模板 & 局部和匿名类型作为模板实参
```c++
//外部模板
extern template void func<int>(int)
template<typename T> class Test {}
//局部和匿名类型作为模板实参
template<typename T> tmp_func(T t){};
typedef struct { int i } B;//B是匿名类型
str
uct {int i} b; //b是匿名类型变量
void Func()
{
    struct C {int i;} c;
    Test<B> b;//C++98编译失败，C++11通过
    Test<C> c;//C++98编译失败，C++11通过
    tmp_func(b);//C++98编译失败，C++11通过
    tmp_func(c);//C++98编译失败，C++11通过
}
```
-----
##### 继承构造函数
```c++
#include <iostream>
#include <string>

using namespace std;

class Base{
public:
    Base(){}
    Base(int ii):i(ii){ print(); }
    Base(int ii,float ff):i(ii),f(ff){ print(); }
    Base(int ii,float ff,double dd):i(ii),f(ff),d(dd){ print(); }
    Base(int ii,float ff,double dd,string ss):i(ii),f(ff),d(dd),s(ss){ print(); }
private:
    void print(){ cout << "Base:i=" << i << ",f=" << f << ",d=" << d << ",s=" << s << endl;}
public:
    int i = 0;
    float f = 0.0;
    double d = 0.0;
    string s = "";
};

class Derived : public Base{
public:
    using Base::Base;//继承构造函数
    Derived():Base(){}
    Derived(const Derived & D){
        this->d = D.d;
        this->f = D.f;
        this->d = D.d;
        this->s = D.s;
    }
private:
    void print() { cout << "Derived:i=" << i << ",f=" << f << ",d=" << d << ",s=" << s << endl; }
};

class Basic{
public:
    Basic(int aa,int bb):a(aa),b(bb){}
protected:
    int a {0};
    int b {1};
};

class DeriveBasic : public Basic{
public:
    using Basic::Basic;
    DeriveBasic(int aa):Basic(aa,aa){}
};

int main()
{
    //Derived D0;
    Derived D1(1);
    Derived D2(1,1.1);
    Derived D3(1,1.1,2.2);
    Derived D4(1,1.1,2.2,string("hello c++11"));
    DeriveBasic b(1);
    return 0;
}
要点：
1.继承构造函数是按需生成的，代码中未被使用的编译器不会为其生成代码，例如将上的D1删掉则Derived的Derived(int)函数代码不会被生成
2.使用了继承构造函数也可以在继承构造函数的基础上自定义构造函数
3.基类构造函数的默认值是不被继承的
4.多继承可能导致继承构造函数冲突，这时我们需要显示定义冲突的继承构造函数
5.私有构造函数不能被继承
6.使用了继承构造函数，编译器就不再为继承类生产默认构造函数了
```
-----
##### 委派构造函数
```c++
#include <iostream>
#include <string>

using namespace std;

class Test{
public:
    Test(int a,double d) : Test(a,d,""){} //委派构造函数
    //Test(int a) : Test(),d(1),s("hello"){} //compile fail,不能有初始化列表
    Test(int a,double d,string s):aa(a),dd(d),ss(s){}
private:
    int aa{0};
    double dd{0.0};
    string ss{""};
};

int main()
{
   Test t(1,2.2);
   Test t1(1,2.2,"c++11");
   return 0;
}
要点
1.委派构造函数不能有初始化列表
2.委派不能形成“委托环”
3.目标委托构造函数中产生的异常可以委托构造函数中捕获
```
-----
##### 右值引用：移动语义和完美转发
左值：可以取地址，有名字
右值：不能取地址，没有名字
移动构造函数：
```c++
#include <iostream>
#include <string.h>

using namespace std;

class Test{
public:
    Test() { cout << "empty construct" << endl; }
    Test(int aa,const char * pp){
        cout << "normal construct" << endl;
        a = aa;
        p = new char[64];
        strcpy(p,pp);
        p[64] = 0;
    }

    Test(Test && t){//移动构造函数
        cout << "move construct" << endl;
        a = t.a;
        t.p = nullptr;
        p = t.p;
    }

    Test & operator=(Test && t){
        cout << "move assignment construct" << endl;
        a = t.a;
        t.p = nullptr;
        p = t.p;
        return *this;
    }
    ~Test(){ cout << "destruct" << endl; }
    Test(const Test & t) = delete;
private:
    int a = 0;
    char * p = nullptr;
};

int main()
{
    Test t = Test(100,"hello c++11");
    cout << "end" << endl;
    return 0;
}
```
g++ -std=c++11 -Wall -fno-elide-constructors -O0 -o test move_constructor.cpp注意编译的时候增加-fno-elide-constructors参数，去除编译器[copy elision][1]的影响

引用折叠：
T & && 自动折叠为 T &
T && && 自动折叠为 T &&
```c++
//完美转发
//template<typename T> void PerfectForward(T && t)
#include <iostream>
using namespace std;

void RunCode(int && m) { cout << "rvalue reference" << endl; }
void RunCode(int & m) { cout << "lvalue reference" << endl; }
void RunCode(const int && m) { cout << "const rvalue reference" << endl; }
void RunCode(const int & m) { cout << "const lvalue reference" << endl; }

template<typename T> void PerfectForward(T && t) { RunCode(forward<T>(t)); }

int main()
{
    int a;
    int b;
    const int c = 0;
    const int d = 0;
    PerfectForward(a);
    PerfectForward(move(b));
    PerfectForward(c);
    PerfectForward(move(d));
    return 0;
}
输出：
lvalue reference
rvalue reference
const lvalue reference
const rvalue reference
```
-----
##### 显示操作符转换
```c++
#include <iostream>

using namespace std;

class Test
{
public:
    Test(int aa):a(aa){}
    //显示类型转让，将Test类型转换成int
    explicit operator int() { return this->a; }
private:
    int a;
};

void func(int i) { cout << "func using param type int" << endl; }
int main()
{
    Test t(1);
    int i = static_cast<int>(t);
    //当转换操作符 operator int()没有explicit修饰的时候，func(t)可以正常运行，编译器会自动调用我们定义的隐式类型转换，当使用explicit修饰后则必须使用static_cast等这样的显示类型转换
    func(i);
    return 0;
}
```
-----
##### 列表初始化
1.列表表达式使用
```c++
#include <iostream>
#include <vector>
#include <set>
#include <list>
#include <map>

using namespace std;

int main()
{
        vector<char> v = {'a','b','c'};
        list<int> l = {1,2,3,4,5};
        set<string> s = {"hello","world","am","linux"};
        map<int,double> m = { {1,1.1},{2,2.2},{3,3.3}};
        //也可使用不带等号的方式
        //vector<char> v {'a','b','c'};
        //list<int> l {1,2,3,4,5};
        //set<string> s {"hello","world","am","linux"};
        //map<int,double> m { {1,1.1},{2,2.2},{3,3.3}};
        for(int c : v) { cout << (char)c << endl; }
        return 0;
}
```
2.自定义对象初始化支持列表表达式 & 函数参数支持列表表达式
```c++
#include <iostream>

using namespace std;

class IntArray
{
public:
    //提供列表表达式初始化版本的构造函数支持自定义对象的列表表达式初始化
    IntArray(initializer_list<int> ints)
    {
        for(auto i : ints)
            int_list.push_back(i);
    }
private:
    vector<int> int_list;
};
//函数参数支持列表表达式
void print(initializer_list<string> str_list)
{
    for(auto s : str_list) 
        cout << s << endl;
}

int main()
{
    IntArray int_array{1,2,3};
    print({"hello","C++","linux"});
    return 0;
}
```
3.防止类型收窄
列表表达式初始化方式是C++11中唯一可以防止类型收窄的方式
```c++
int i = 1.2;//实际存储的是1，float/double收窄为int
int j {2.0f};//编译失败，防止类型收窄
```
-----
##### POD
Plain Old Data定义
>* 1.拥有平凡的默认构造函数（trivial constructor）和析构函数(trivial destructor)，所谓平凡的构造函数即什么都不做的构造函数，自定义的构造函数都不是平凡的构造函数
>* 2.拥有平凡的拷贝构造函数和移动构造函数，使用简单的为拷贝方式进行拷贝
>* 3.拥有简单的拷贝赋值运算符和移动赋值运算符
>* 4.不包含虚函数、虚基类

标准布局：
1.所有非静态成员有相同的访问权限
2.在类或者结构体继承时，满足以下两种情况之一
    >* 派生类中有非静态成员，且只有一个仅包含静态成员的基类
    >* 基类中有非静态成员，而派生类没有非静态成员
    
3.类中第一个非静态成员的类型与其基类不同
4.没有虚函数和虚基类
5.所有非静态成员均符合标准布局类型，其基类也符合标准布局

-----
##### 非受限联合体
任何非引用类型都可成为联合体的数据成员
联合体拥有非POD成员且该成员有非平凡构造函数，则该联合体的默认构造函数将被编译器删除（其他的各种构造函数也一样），这种情况下程序员应该自己定义联合体的构造函数

-----
##### 用户自定义字面量
```c++
#include <iostream>

using namespace std;

struct Weight
{
        Weight(unsigned long long ww) { w = (unsigned int)ww; }
        unsigned int w;
};
//字面量操作符函数语法
Weight operator "" _kg(unsigned long long w)
{
        return Weight(w);
}

int main()
{
        //使用方式
        Weight one_ton = 1000_kg;
        return 0;
}
```
c++11字面量什么规则：
>* 如果字面量为整数，那么字面量操作符函数的参数只能是unsigned long long或者const char *，unsigned long long无法容纳字面量的时候，编译器会自动将该字面量转换为以'\0'结尾的字符串，并调用已const char *为参数的版本的字面量操作符函数处理字面量
>* 如果字面量为浮点型，那么字面量操作符函数的参数只能是long double或者const char *（该类型的调用规则同上）
>* 如果字面量为字符串，则字面量操作符函数只能接受const char *，size_t为参数（已知长度的字符串）
>* 在字面量操作符函数声明中 operator ""与用户自定义后缀之间必须要有空格
>* 后缀建议以下划线开始（20120102L表示一个合法的长整型数，如果自定义一个不以下划线开头L为后缀的类型则容易引起混乱）

-----
##### 内联名字空间
这个自己google吧

-----
##### 模板的别名
```c++
//C++98:typedef,C++11新增using语法
typedef unsigned char uint8; 
using uint8 = unsigned char;
template<typename T> using MapStr = map<T,string>;
MapStr<int> number_string;
```
-----
##### auto类型推导
>* auto可以与指针和引用结合起来使用
>* 可以与cv(const  & volatile)限定符一起结合使用
>* auto不会带走cv特性
>* auto会带走指针和引用类型
>* auto可以一次申明多个变量的类型，但是这些变量的类型必须是一样的

auto不能推导的4种情况
>* 函数的形参不能是auto的
>* 对结构体来说，非静态成员变量的类型不能是auto的
>* 声明auto数组
```c++
char x[3];
auto y = x;
auto z[3] = x;//compile error
```
>* 实例化模板的时候使用auto作为模板参数，入vector<auto> v
```c++
auto i = 1; //int
auto d = 1.11f; //float 
auto & j = i; // int &
auto * p = &i; // int *
const auto ci = 1; // const int
const auto & cir = i; //const int &
volatile auto & g = d; //volatile float &
const string s = "hello"
auto s1 = s;//string，不是const string，auto不会带走cv特性，但是引用和指针会带走
```
##### typeid & decltype
__typeid__
```c++
//查询变量的类型
#include <iostream>
#include <typeinfo>
using namespace std;
class Test1{};
class Test2{};
int main()
{
    Test1 t1;
    Test2 t2;
    cout << typeid(t1).name() << endl;
    cout << typeid(t2).name() << endl;
    cout << "t1 and t2 is same type:" << (typeid(t1).hash_code() == typeid(t2).hash_code() ? "True" : "False") << endl;
    return 0;
}
```
__decltype__
语法：decltype(expression)
decltype总是以一个普通的表达式为参数，类型推导是在编译时进行的，它可以与typedef/using结合使用，他可以用来推导匿名类/结构体/联合体类型，
**decltype能够带走表达式的cv限制符，不过如果对象的定义中有const或者volatile限制符，使用decltype推导时其成员不会继承const或者volatile，另外delctype推导出来的类型的*不会被省略**

推导规则：
>* 如果e是一个没带括号的标记符表达式或者类成员访问表达式，那么decltype(e)就是e所命名的尸体的类型，如果e是一个被重载的函数则会导致编译报错
>* 否则，假设e的类型是T，如果e是一个右值，那么decltype(e)的类型是T &&
>* 否则，假设e的类型是T，如果e是一个左值，则decltype(e)为T&
>* 否则，假设e的类型是T，则decltype(e)为T

ps：标记符表达式：所有出去关键字、字面量等编译器需要使用的标记之外的程序员自定义的的标记都可以是标记符，单个标记符对应的表达式就是标记符表达式
```c++
using size_t = decltype(sizeof(0));
using ptrdiff = decltype((int*)0-(int*)0);
using nullptr_t = decltype(nullptr);
vector<int> int_vec;
typedef decltype(int_vec.begin()) IntVecIter;
int i = 0; 
int * p = &i;
delctype(pp)* pp = &p; //pp的类型是 int **，*不会被省略
```
在模板中的应用：
```c++
template<typename T1,typename T2>
void sum(T1 t1,T2 t2,decltype(t1+t2) & ret_var){
    ret_var = t1 + t2;
}
template<typename T1,typename T2>
auto sum(T1 t1,T2 t2) -> decltype(t1+t2){
    return t1+t2;
}
//以上两个模板如果是在c++98中我们必须重载出多种返回值的模板，而在C++11中这些事情可以让编译器帮你做
```
-----
##### 追踪返回类型
```c++
template<typename T1,typename T2>//在模板中的应用
auto sum(T1 t1,T2 t2) -> decltype(t1+t2){
    return t1+t2;
}
auto (*fp)(int,int) -> int;//在函数指针上的应用
auto (&fp)(int,int) -> int;//在函数引用上的应用
ps:可以用在类或者结构体成员函数、模板类成员函数上
```
-----
##### 新的for循环
```c++
vector<int> int_vec;
for(auto i : int_vec) //for(int & i : int_vec)使用引用的方式
    cout << i << endl;
使用这种for循环需要迭代的对象实现++和==等操作符
```
-----
##### 强枚举类型
```c++
enum class Gender { Male,Female };//旧枚举没有class关键字
enum class RGB : char { R='r',G='g',B='b'};//指定底层类型的强枚举
```
>* 强枚举类型枚举成员的名称不会被输出到其父作用域
>* 强枚举成员的值不可以与整形隐式的相互转化，强枚举类型要转化成其他类型时必须用显示转换
>* 可以指定底层类型，默认底层类型为int
>* enum class和enum struct是一样的

-----
##### C++11中的智能指针(unique_ptr,shared_ptr,weak_ptr)
-----
__unique_ptr__
指针所指对象的所有权独占，所有权仅能通过标准库的move函数实现转移，转移后原来占有内存对象的指针会失去内存对象的所有权，从实现上来讲，unique_ptr是一个删除了拷贝构造函数、保留了移动构造函数的指针封装类型，程序员仅可以通过右值对unique_ptr对象进行构造
__shared_ptr__
指针所指的对象所有权可以在多个shared_ptr共享，shared_ptr是采用引用计数实现的，一旦一个shared_ptr放弃了所有权计数就减一，只有在计算器值为0的时候对象所占内存才被真正的释放
__weak_ptr__
它可以指向shared_ptr指针所指向的对象的内存，却并不拥有该对象的内存，使用weak_ptr成员lock则返回一个shared_ptr对象，且在所指对象内存失效时返回空指针（可以用来验证shared_ptr的有效性）
```c++
#include <iostream>
#include <memory>

using namespace std;

int main()
{
        unique_ptr<int> up1(new int(100));//仅能通过右值进行初始化
        //unique_ptr<int> up2 = up1;//编译报错，所有权独占，不可被其他shared_ptr共享
        cout << *up1 << endl;
        unique_ptr<int> up3 = move(up1);//所有权仅能使用move进行转移
        cout << *up3 << endl;//现在所有权属于up3
        //cout << *up1 << endl;//已失去所有权，运行时错误
        up3.reset();//显示释放内存，放弃所有权
        up1.reset();//不会报错

        shared_ptr<int> sp1(new int(1111));
        shared_ptr<int> sp2 = sp1;//所有权可以共享
        cout << *sp1 << endl;
        cout << *sp2 << endl;
        sp1.reset();
        cout << *sp2 << endl;//不受sp1.reset()的影响
        return 0;
}
```
```c++
#include <iostream>
#include <memory>

using namespace std;

void check(weak_ptr<int> & wp)
{
    shared_ptr<int> sp = wp.lock();//转为shared_ptr
    if(sp != nullptr) cout << "still " << *sp << endl;
    else cout << "pointer is invalid" << endl;
}

int main()
{
    shared_ptr<int> sp1(new int(2222));
    shared_ptr<int> sp2 = sp1;
    weak_ptr<int> wp = sp1;
    cout << *sp1 << endl;
    cout << *sp2 << endl;
    check(wp);//still valid
    sp1.reset();
    cout << *sp2 << endl;
    check(wp);//still valid
    sp2.reset();
    check(wp);//invalid
    return 0;
}
```
-----
##### 常量表达式--constexpr
constexpr：编译时常量，虽然编译器不一定这么实现但是至少效果看起来是这样的，它可以作用于函数、数据声明、类的构造函数
__常量表达式函数__
常量表达式函数的要求
>* 函数体只有单一的return返回语句
```c++
//不是单一return语句，无法编译通过
constexpr int data(){ const int i = 0;return i;}
constexpr int func(int x){
    //ok，static_assert不产生实际的代码
    static_assert(0==0,"assert fail");
    return x;
}
```
>* 函数必须返回值不能是void返回
```c++
//没有返回值，编译出错
constexpr f(){}
```
>* 使用前必须已定义
```c++
constexpr int f();
int a = f();//a非常量表达式，编译器将f转换成函数调用，ok
const int b = f();//b非常量表达式，编译器将f转换成函数调用，ok
constexpr int c = f();//c是常量表达式,无法编译通过，f尚未定义
constexpr int f() {return 1;}//定义
constexpr int d = f();//ok，f已经定义过了
```
>* return语句表达式中不能使用非常量表达式函数、全局函数，且必须是一个常量表达式
```c++
const int e() { return 1;}
//编译失败，使用非常量表达式函数
constexpr int g() { return e(); }
```
__常量表达式值__
常量表达式值必须被一个常量表达式赋值，且在使用前必须被初始化，未被使用的常量表达式值编译器可以不为其生成实际代码，比较特殊的是浮点数常量表达式值，因为编译器编译时浮点数的精度和运行时精度可能不一致，所以浮点数常量表达式值要求运行时精度要大于等于编译时精度
自定义类型无法直接使用constexpr来修饰，如下面的写法是无法通过编译的：
```c++
constexpr struct MyType { int i; };
constexpr MyType mt = {0};
```
要使自定义类型可以成为常量表达式值我们需要为自定义类型对象提供常量表达式构造函数
```c++
struct MyType{
    constexpr MyType(int x):i(x){}
    int i;
};
constexpr MyType mt = {0};
```
常量表达式构造函数使用约束：
>* 函数体必须为空
>* 初始化列表只能由常量表达式来赋值
__常量表达式的其他应用__
>* 声明为常量表达式的模板函数实例化时如果不满足常量表达式的要求则constexpr会被自动忽略
>* 递归的常量表达式函数应该至少支持512层的递归

-----
##### 变长模板
-----
##### 原子类型
```c++
头文件：#include <cstdatomic>
```
__原子类型和内置类型对应表__
<table><thead><tr><th>原子类型名称</th><th>对应内置类型名称</th></tr></thead><tbody><tr><td>atomic_bool</td><td>bool</td></tr><tr><td>atomic_char</td><td>char</td></tr><tr><td>atomic_schar</td><td>signed char</td></tr><tr><td>atomic_uchar</td><td>unsigned char</td></tr><tr><td>atomic_int</td><td>int</td></tr><tr><td>atomic_uint</td><td>unsigned int</td></tr><tr><td>atomic_short</td><td>short</td></tr><tr><td>atomic_ushort</td><td>unsigned short</td></tr><tr><td>atomic_long</td><td>long</td></tr><tr><td>atomic_ulong</td><td>unsigned long</td></tr><tr><td>atomic_llong</td><td>long long</td></tr><tr><td>atomic_ullong</td><td>unsigned long long</td></tr><tr><td>atomic_char16_t</td><td>char16_t</td></tr><tr><td>atomic_char32_t</td><td>char32_t</td></tr><tr><td>atomic_wchar_t</td><td>wchar_t</td></tr></tbody></table>
除了上述已经定义好的方式，还可以使用atomic类模板定义任意类型的的原子类型
```c++
std::atomic<T> t;
```
C++11标准不允许原子类型进行拷贝构造、移动构造，以及使用operator=等进行构造，只能重启模板参数类型中进行构造，atomic类模板总是定义了从atomic< T >到T的类型转换函数，编译器会在必要的时候进行隐式类型转换
__atomic类型的操作__
<table><thead><tr><th>操作</th><th>atomic_flag</th><th>atomic_bool</th><th>atomic-integral-type</th><th>atomic&lt; bool &gt;</th><th>atomic&lt; T* &gt;</th><th>atomic&lt; integral-type &gt;</th><th>atomic&lt; class-type &gt;</th></tr></thead><tbody><tr><td>test_and_set</td><td>Y</td><td></td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>clear</td><td>Y</td><td></td><td></td><td></td><td></td><td></td><td></td></tr><tr><td>is_lock_free</td><td></td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td></tr><tr><td>load</td><td></td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td></tr><tr><td>store</td><td></td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td></tr><tr><td>exchange</td><td></td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td></tr><tr><td>compare_exchange_weak  compare_exchange_strong</td><td></td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td><td>Y</td></tr><tr><td>fetch_add,+=</td><td></td><td></td><td>Y</td><td></td><td>Y</td><td>Y</td><td></td></tr><tr><td>fetch_sub,-=</td><td></td><td></td><td>Y</td><td></td><td>Y</td><td>Y</td><td></td></tr><tr><td>fetch_or,|=</td><td></td><td>Y</td><td></td><td></td><td>Y</td><td></td><td></td></tr><tr><td>fetch_and,&amp;=</td><td></td><td></td><td>Y</td><td></td><td></td><td>Y</td><td></td></tr><tr><td>fetch_xor,^=</td><td></td><td></td><td>Y</td><td></td><td></td><td>Y</td><td></td></tr><tr><td>++,--</td><td></td><td></td><td>Y</td><td></td><td>Y</td><td>Y</td><td>Y</td></tr></tbody></table>

notice：
atomic_flag是无锁的，线程对其进行访问的时候是不需要加锁的
```c++
//使用atomic_flag实现自旋锁
#include <iostream>
#include <atomic>
#include <thread>
//g++ -o spinlock spinlock.cpp -std=c++11 -lpthread
using namespace std;

class SpinLock final
{
public:
    SpinLock() = default;
    SpinLock(const SpinLock &) = delete;
    SpinLock(SpinLock &&) = delete;
    SpinLock & operator=(const SpinLock &) = delete;
    ~SpinLock()
    {
        Unlock();
    }
public:
    void Lock()
    {
        while(lock.test_and_set(std::memory_order_acquire));
    }
    void Unlock()
    {
        lock.clear(std::memory_order_release);
    }
private:
    std::atomic_flag lock = ATOMIC_FLAG_INIT;
};

static int num = 0;

void Incr() { for(int i = 0; i < 10000000; ++i) ++num; }
void IncrWithLock(SpinLock * lock)
{
    for(int i = 0; i < 10000000; ++i)
    {
        lock->Lock();
        ++num;
        lock->Unlock();
    }
}
int main()
{
    SpinLock lock;
    thread t1(IncrWithLock,&lock);
    thread t2(IncrWithLock,&lock);
    t1.join();
    t2.join();
    cout << num << endl;
    return 0;
}
```
memory_order枚举值
<table><thead><tr><th>枚举值</th><th>定义规则</th></tr></thead><tbody><tr><td>memory_order_relaxed</td><td>不对执行顺序做任何保证</td></tr><tr><td>memory_order_acquire</td><td>本线程中，所有后续操作必须在本原子操作完成后执行</td></tr><tr><td>memory_order_release</td><td>本线程中，所有之前的写操作完成后才能执行本原子操作</td></tr><tr><td>memory_order_acq_rel</td><td>同时包含memory_order_acquire和memory_order_release标记</td></tr><tr><td>memory_order_consume</td><td>本线程中，所有后续的有关本原子类型的操作，必须在本条原子操作完成之后执行</td></tr><tr><td>memory_order_seq_cst</td><td>全部存取都按顺序执行</td></tr></tbody></table>


-----
##### 线程局部存储
```c++
int thread_local err_code;
//ps:thread_local变量的性能一般不会高于普通全局/静态变量
```
-----
##### 快速退出
```c++
abort,terminate:异常退出，不会自动调用析构函数，可以使用set_terminate改变terminate函数的默认行为，abort默认向符合POSIX的系统抛出一个SIGABRT信号
exit[at_exit]:正常退出，会正常调用自动变量的西沟函数，并且还会调用atexit注册的函数
quick_exit[at_quick_exit]:正常退出，不执行析构函数（某些时候不析构自动变量没什么问题）
```
-----
##### nullptr
nullptr是关键字，类型未nullptr_t，该类型是有decltype(nullptr)得到的，只能被隐式转换为指针类型（不可以是整形），nullptr是一个编译时期常量，nullptr到任何指针的转换是隐式的，nullptr_t对象的地址可以被用户使用，但是nullptr不能取地址
>* 所有定义为nullptr_t类型的的数据都是等价的，行为完全一致
>* nullptr_t类型数据可以隐式转换为任意指针类型
>* nullptr_t不能转换为非指针类型，即使使用reinterpret_cast<T>也不行
>* nullptr_t类型不适用于算术运算表达式，但可以用于关系运算表达式（==,<=,>=等）

-----
##### 默认函数控制:[=default,=delete]
```c++
#include <iostream>

using namespace std;

class Test
{
public:
    Test()=default;//使用编译器默认合成的构造函数
    Test(const Test & test) = delete;//删除Test类的拷贝构造函数
    Test(int v) = delete;//删除参数为int版本的构造函数，防止隐式类型转换
    //explicit Test(int v) = delete;//不要和explicit一起使用，这样只会删除显示转换，不会删除隐式转换
    void * operator new(std::size_t) = delete;//删除new操作符，禁止在堆上为Test分配内存
private:
    int var = 0;
};

int main()
{
    Test t;
    //Test t1(t);
    //Test t2 = t;
    //Test * pt = new Test();
    return 0;
}
```
-----
##### lambda表达式
```c++
[capture] (parameters) mutable -> return-type {statement}
```
>* capture:捕捉列表，捕捉上下文中的变量供lambda表达式使用
>* parameters:参数列表，如果不需要参数则可以和()一起省略
>* mutable:默认情况下lambda函数总是一个const函数，mutable可以取消其常量性，使用该修饰符是参数列表不可以省略
>* return-type:函数返回类型，不需要返回值的时候可以连同->一起省略，返回值明确的时候也可以省略让编译器自行推断
>* statement:和普通函数一样，可以使用parameters参数列表也可以使用capture捕获的变量

_捕捉列表语法_
>* [var]：传值方式捕捉变量var，值在被捕捉的时候就固定下来了，捕捉后到lambda函数调用前对var的更改不会影响之前被lambda捕捉到的var的值，这点千万注意
>* [=]：值传递方式捕捉所有父作用域的变量，包括this
>* [&var]：引用方式捕捉变量var
>* [&]：引用方式捕捉父作用域的变量，包括this
>* [this]：传值方式捕捉当前的this指针
ps：可以多种方式组合使用，用逗号分隔即可，仅能捕捉父作用域的自动变量，非此作用域或者非自动变量都会导致编译失败
 
[1]: http://en.cppreference.com/w/cpp/language/copy_elision
