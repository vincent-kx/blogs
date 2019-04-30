# Effective STL总结

### C++ STL的主要容器

+ 标准stl序列容器：string，vector，list，deque

+ 标准stl关联容器：map，multimap，set，multiset

+ 标准散列容器：unordered_set，unordered_multiset，unordered_map，unordered_multimap

+ 非标准序列容器：slist(单向链表实现的列表容器)，rope(重型的string)

+ 非标准散列容器：hash_set，hash_multiset，hash_map，hash_multimap

+ 其他标准非stl容器：bitset，valarray，stack，queue，priority_queue

  其中string，vector，deque，rope是连续内存容器(内存空间是连续的内存块，在中间插入或者删除元素时需要移动插入位置后面的元素来填充或者腾出空间)



###选择合适的容器

+ 如果在意查找速度，那么应该按 "散列容器(hash) -> 排序的vector(二分查找) -> 关联容器(红黑树)"的优先级考虑选择使用哪种容器
+ 大部分string和权威rope的实现是基于引用计数的，如果你对底层的引用计数很在意，那么应该考虑使用vector\<char\>来代替
+ list是唯一提供多元素插入**事务性**语义的标准stl容器(连续容器也可以实现事务性语义，不过有比较大的开销，可以参考《exceptional C++》条款17[8])
+ 一般基于连续内存的容器在插入和删除会使所有的迭代器、指针和引用都失效，而基于节点的迭代器不会

###使容器对象的拷贝轻量而正确

+ 容器原始的增加、移动等都是依赖copy完成的，也就是依赖元素对象的copy constructor 和 assignment constructor，由于继承的存在，将一个子类对象放入元素是父类对象的容器时，拷贝会导致分割，也就是实际拷贝的时候调用的是父类的copy constructor，这样子类对象的派生部分会被删除(<font color="#dd0000">关于分割问题更多的背景知识，请参考《Effective C++》条款22</font>>)，如果确实需要将子类对象放到元素是父类对象的容器中时，应该优先考虑使用指针容器

+ stl容器只建立你需要的个数的对象，而且他们只在你指定的时候做，比如：

  ```c++
  Test testArr[10]; //建立一个大小为10的Test数组，使用默认构造函数构造这10个元素
  vector<Test> testArr; //建立一个0个Test对象的vector，需要时才增加
  testArr.reserve(10); //能够容纳10个Test对象的空vector，不会构造Test对象
  ```



### 使用empty代替size()==0

+ 对于所有的标准容器，empty是o(1)时间复杂度的，但是对于list，有些实现size是o(n)时间复杂度的

  > list有一个splice函数，实现两个list的拼接功能，如果splice是o(1)时间复杂度，那么size就只能是o(n)复杂度的，因为splice是o(1)复杂度的，那么拼接之后是无法知道被接入的list的长度是多少，如果要知道，就只能遍历，也就成o(n)复杂度了



### 尽量使用区间成员函数代替它们的单元素兄弟

我们先来看一下下面的代码：

```c++
//假设有两个vector<int> vec1,vec2,vec1是空的，我们需要将vec2的第5个到第10个元素赋值给vec1那么我们可能有以下几种版本的写法
//版本1，使用循环
for(it = vec2.begin()+5; it != vec2.begin()+10; ++it) {
  vec1.push_back(*it);
}

//版本2，使用算法函数
copy(vec2.begin()+5,vec2.begin()+10,back_inserter(v1));

//版本3，使用成员函数insert
vec1.insert(v1.end(),vec2.begin()+5,vec2.begin()+10);

//版本4，使用成员函数assign
vec1.assign(vec2.begin()+5,vec2.begin()+10);
```

先说结论，区间成员函数的好处：

1. 写的代码更少，代码更清晰，这个没什么好说明的
2. 性能更优

我们重点分析一下以上几个版本代码的性能：

版本1和版本2：

- 版本1需要多次调用push_back函数，函数的调用是有成本的，版本2copy实际也是通过循环来实现的
- vector是动态扩展的，当vec2当前的load超过一定的值的时候会自动重新分配空间，将老的元素copy到新空间，然后释放老的空间。标准序列类型的元素都有这个问题，节点类的容器没有这个问题
- 如果插入的新元素的位置不是末尾而是中间，那么每次插入都需要将插入位置之后的所有元素向后移动一位来腾出空间给新插入的元素。加入需要插入5个元素，插入的位置后面还有10个元素，那么全部插入则需要移动50次元素（每个元素移动5次），当然节点类的容器没有这个问题

版本3和版本4：以上问题大部分情况下都不存在

+ 只会有一次函数调用
+ 通过区间迭代器的算术运算，区间实际执行前会计算到底有多少需要操作的元素，从而使得空间分配只需要一次，每个元素的移动也只需要一次，也避免了空间扩展和收缩过程中的多次拷贝和移动，只需要一次就可以完成。即使是少数不支持直接计算得到数量的迭代器—如istream_iterator,区间版本的成员函数实现方式性能上最多也只是和单元素形式的一样，而不会更低

### 当使用new的指针的容器时，记得在销毁容器前delete那些指针

```c++
vector<Test> testVec;
for(int i = 0; i < 10; ++i) {
  testVec.push_back(new Test)
}

//正确的销毁姿势(没考虑异常安全的问题)
for(auto it = testVec.begin(); it != testVec.end(); ++it) {
  if(*it != NULL) {
    delete *it;
  }
}

//永远不再在容器中存放auto_ptr指针，容器的元素的加入和取用都是使用的拷贝的方式(引用除外)，auto_ptr的特性是拷贝的时候所有权就转移了，比如
Test * tp = new Test;
auto_ptr<Test> tp1 = auto_ptr(tp);
auto_ptr<Test> tp2 = tp1; //此时原来tp1指向对象的所有权转移到了tp2，这个时候tp1=NULL
```

### 注意分配器的协定和约束

+ 分配器是对象，所有相同类型的分配器对象是等价的而且比较起来总是相等，这就进一步要求分配器是无状态的对象

  > 一个list\<T\> l1中的对象接合到另一个list\<T\> l2当中，是没有进行过拷贝操作的，只是调整了指针，当l2销毁时，必须必须回收之前接合进来的l1的对象，如果给l1分配内存的分配器和l2分配内存的分配器不是等价的，那么这种接合操作就很难实现，因为接合后原属于l1的对象的内存的回收，可能无法正常进行

+ 实现自定义分配器请遵循一下原则

  1. 把你的分配器做成一个模板，带有模板参数T，代表你要分配内存的对象类型
  2. 提供pointer和reference的typedef，但总是让pointer是T *，reference是T &
  3. 分配器一定不能带状态，也就是不能有非静态成员变量
  4. 给分配器的allocate成员函数传的参数是对象的个数而不是字节数，函数应该返回T *而不是void *(注意和malloc族函数的区别)
  5. 一定要提供标准容器依赖的内嵌rebind模板

  **建议你从Josuttis的样例allocator网页[23]或Austern的文章《What Are Allocators Good For?》[24]的 代码开始，而不是从头开始写样板**

### 尽量使用vector和string来代替动态分配的数组

如果你使用 new T[10];这种形式来动态分配数组，那么后续你就需要自己管理内存，要考虑何时释放内存，并且确保最终内存一定会被释放，否则就会出现内存泄露；如果你的数组空间已经满了，而你需要继续往里面加元素，你还得自己重新分配内存，而使用vector和string则不会有这些问题，他们会自动管理好内存

### 使用reserve来避免不必要的重新分配

对于标准序列容器，往容器中新增元素而容器空间不足的时候会产生内存的重新分配，大体会做以下工作：

1. 分配新的内存块，大部分实现中新的内存是原有的内存的两倍大小
2. 把原内存的原始拷贝到新内存
3. 销毁原内存中的元素
4. 回收原内存的空间

这是非常昂贵的操作，而reserve则按照你指定的大小提前一次性分配好你所需要的内存，这样可以避免内存的多次分配和昂贵的拷贝、销毁等操作，下面看一个例子对比：

```c++
//sample-1
vector<int> v;
for(int i = 0; i < 1000; ++i) {
	v.push_back(1000);
}
//sample-2
vector<int> v;
v.reserve(1000); //不会调用默认构造函数构建对象
for(int i = 0; i < 1000; ++i) {
	v.push_back(1000);
}
```

sample-1的操作大概会产生10次内存分配操作 + 9轮拷贝操作 + 9轮释放操作 + 9轮内存回收(1000约等于$2^{10}$，第一次push_back不会产生copy等操作)，而sample-2则只有一次内存分配操作，其他的都没有

### 小心string实现的多样性(sizeof str跟实际的实现相关)

string的实现是多种多样的，有的实现将存储数据部分的内存和string对象的内存放在一起，有的只是在string对象中存放了一个指向实际存储元素内存的指针，另外根据实现的不同string对象还可能有以下的一些类型的数据：

1. 多线程并发控制相关的数据
2. 小字符串特有的内存块(比如小于16个字符的时候直接存储在对象内存中，此时存储的指向实际存储元素的内存块的指针是NULL)
3. 内存分配器
4. 引用计数器

### 使用“交换技巧”来修整过剩容量

假设有一个vector<int> v 它的capacity是1000，但是其中只存放了10个元素(vector的元素被erase的时候只会析构掉元素对象，并不会自动收缩v的实际容量的大小)，我们可以使用以下技巧来实现空间的收缩

```c++
//sample-1
vector<int>(v).swap(v);
//sample-2
vector<int>(v.begin(),v.end()).swap(v)
```

copy construct只会拷贝需要的元素，所以生成的临时对象的空间是能容纳v中所有元素的最小空间（具体是多少取决于实现）,对于string也可以使用同样的方法

```c++
string(s).swap(s);
```

### 理解等价和相等的区别

等价：通常基于operator<

相等：通常基于operator==

find算法的查找是基于相等这个概念，而插入删除和成员函数find是基于等价这个概念的

"talk is cheap,show me the code"，我们用下面一段代码来做演示和分析（在注释中说明）：

```c++
#include <iostream>
#include <string>
#include <set>
#include <algorithm>

using namespace std;

class TestString {
    private:
        string data;
    public:
	static char toLower(char c) {
	    if(c >= 'A' && c < 'Z') {
	        return c - ('A' - 'a');
	    }
	    return c;
	}

	static string toLower(string s) {
            string str(s);
	    for(size_t i = 0; i < str.length(); ++i) {
	        str[i] = toLower(str[i]);
	    }
	    return str;
	}
    public:
	TestString(const string & data) {
	    this->data = data;
	}
	TestString(const TestString & str) {
	    this->data = str.getData();
	}
	void print()  {
	    cout << data << endl;
	}
        const string getData() const {
	    return data;
        }

	 size_t length()  {
	    return data.length();
	}

  //判断两个对象是否等价，不区分大小写
	bool operator<(const TestString & other) const {
            string s1 = toLower(this->data);
	    string s2 = toLower(other.getData());
	    return s1 < s2;
	}

  //判断两个对象的值是否相等
	bool operator==(const TestString & other) const {
	    return this->data == other.getData();
	}
};

int main ()
{
    string str1("ABCDEF");
    string str2("abcdef");

    TestString tstr1(str1);
    TestString tstr2(str2);
   
    set<TestString> setTestStr;
    setTestStr.insert(tstr1);
    //没有进行实际的插入，执行插入tstr2后setTestStr的size=1,因为根据operator<的定义tstr1和tstr2是等价的，当进行tstr2的插入时容器发现其中已经存在了一个与其等价的值，所以没有进行插入，后面的测试输出也印证了这一点
    setTestStr.insert(tstr2); 
  
    //这里不会输出，因为operator==定义决定了这两个对象是相等的
    if(str1 == str2) {
        cout << "str1 == str2" << endl;
    }
    
    //输出str1 found:ABCDEF
    auto it1 = setTestStr.find(tstr1);
    if(it1 != setTestStr.end()) {
        cout << "str1 found:" << it1->getData() << endl;
    }
    //输出str2 found:ABCDEF，实际tstr2的值是abcdef，但是因为等价的概念找到的是tstr1
    auto it2 = setTestStr.find(tstr2);
    if(it2 != setTestStr.end()) {
        cout << "str2 found:" << it2->getData() << endl;
    }
    //find算法是基于“相等”来查找的
    //输出algorithm found str1:ABCDEF，因为setTestStr中存在一个与tstr1相等的对象
    auto it3 = std::find(setTestStr.begin(),setTestStr.end(),tstr1);
    if(it3 != setTestStr.end()) {
        cout << "algorithm found str1:" << it3->getData() << endl;
    }
    //不会输出，因为setTestStr中不存在一个与tstr1“相等”的对象，实际存在的那个对象与tstr2是“等价”的而不是相等的
    auto it4 = std::find(setTestStr.begin(),setTestStr.end(),tstr2);
    if(it4 != setTestStr.end()) {
        cout << "algorithm found str2:" << it4->getData() << endl;
    }
    return 0;
}
```

以上规则同样适用于其他的**标准**关联容器

### 为指针的关联容器指定比较类型

我们先看一段代码：

```c++
set<int *> intSet;
for(int i = 0; i < 10; ++i) {
  intSet.insert(new int(i));
}

for(auto it = intSet.begin(); it != intSet.end(); ++it) {
  cout << **it << endl;
}
```

对于这段代码你可能期望它的输出是按0到9的顺序输出10个数字，而这很有可能是错误的预期，因为set中存储的是这些数据的地址，那么这些数据在set中的排序是按照它的地址进行排序的，而不是他们本身的值。要让set按你的预期对存储的指针进行排序，你必须提供额外的用户比较的类型。正确的代码应该是像下面这样的

```c++
template<typename PTR>
class Comp {
public:
  bool operator()(PTR p1,PTR p2) {
      return *p1 < *p2;
  }
};

set<int *,Comp<int*>> intSet;
for(int i = 0; i < 10; ++i) {
  intSet.insert(new int(i));
}

for(auto it = intSet.begin(); it != intSet.end(); ++it) {
  cout << **it << endl;
}
```

### 永远让比较函数对相等的值返回false

关联容器判断一个元素是否存在使用的是等价，而等价的定义转换成代码是这样的：

```c++
!(e1 < e2) && !(e2 < e1)
```

假设我们有一个set\<int\> s它已经包含了一个元素是10，我们又做了s.insert(10)的操作，如果对于相等的值我们返回的是true，也就是e1 <= e2，那么等价的定义就变成了：

```c++
!(e1 <= e2) && !(e2 <= e1) // !(10 <= 10) && !(10 <= 10) => false && false => false
```

如果是这样，那么我们第二次往set里面加入时，容器会认为新加入的10和原容器中已存在的10是不等价的，就会将第二个10加入到了容器中，这个时候set中就同时有了两个10，这显然不符合set的定义。对于multiset也同样存在问题，multiset使用equal_range的时候返回的是"等价的值的范围"，如果出现上述情况，调用equal_range(10)的时候，容器会认为参数中的10不等价于已存在于multiset中的两个10的任何一个，也就什么都不会返回，这显然是非预期的结果。

### 在map::operator[]和map-insert之间仔细选择

直接先说结论：

+ 如果是更新map中的元素operator[]更快
+ 如果是插入不存在的元素insert更快

operator[]函数：当元素已存在的时候，返回这个元素的引用，元素不存在时则新增一个元素，这说明operator[]存在一下几个步骤：

1. 查找元素是否存在
2. 如果存在直接返回元素的引用
3. 如果不存在则调用元素的默认构造函数构造一个临时对象，将临时对象copy到map的内存中，然后析构临时对象，最后将需要新增的元素的拷贝给刚才新增的元素并返回对它的引用

insert函数：

1. 查找插入的位置
2. 将要插入的元素拷贝到map中
3. 如果是待指示的insert(有一个参数是插入位置的迭代器)，那么可以省去查找插入位置的时间

从两个函数的执行原理上来看，很容易得出最开始的结论

###尽量用iterator代替const_iterator，reverse_iterator和const_reverse_iterator
原因：iterator在stl的函数接口中是使用最广泛的，而从其他三种转换成iterator通常很麻烦，而且有些是无法完成的，为了代码的便利性，我们尽量使用iterator

转换关系：

+ iterator —>const_iterator
+ iterator —>reverse_iterator—>const_reverse_iterator
+ reverse_iterator—base()—>iterator(通过reverse_iterator的base成员函数转换)
+ reverse_iterator —>const_reverse_iterator
+ const_reverse_iterator—>const_iterator
+ 无法从const_iterator直接转换成iterator，也无法从const_reverse_iterator直接转换成reverse_iterator

### 确保目标区间足够大

stl的算法是不保证目标存储空间的大小足够容纳算法输出的结果的，也就是说你需要自己保证用来存储算法输出的结果的容器的空间是足够容纳算法的输出结果的

```c++
int trasferFunc(int x);
vector<int> values;
... //values初始化
vector<int> results;
//results没有足够的空间来存储transform转换values得到的结果
//transform(values.begin(),values.end(),result.end(),trasnferFunc) //错误
transform(values.begin(),values.end(),back_inserter(results),trasnferFunc) //正确
```

确保results有足够的空间一般有以下方法：

1. 使用back_inserter、front_inserter、inserter，当然只能用于支持这些方法的容器上
2. 提前自己分配好空间，如：

```c++
results.reserve(values.size());
results.resize(values.size());
```



### 了解你的排序选择

+ 在标准序列容器上(vector/string/deque/数组)进行完全排序：sort、stable_sort(稳定的排序)
+ 在标准序列容器上(vector/string/deque/数组)取有序的top-N：partial_sort
+ 在标准序列容器上(vector/string/deque/数组)取无序的top-N：nth_element
+ 将一个标准序列容器分类为互补的A，B两大类：partition、stable_partition
+ 如果是list，可以使用sort(对于list排序是稳定的)、partition和stable_partition，如果需要在list上实现partial_sort或者nth_element的效果，你只能间接的完成，比如将元素拷贝给vector、将迭代器放入vector中进行排序

对于性能一般来说做更多工作的算法会花费更长的时间，必须稳定的算法比忽略稳定性的算法需要更多的时间，以上几个算法的性能排序时间大约是下面这样：

partition < stable_partition < nth_element < partial_sort < sort < stable_sort

###remove并不“真的”删除东西，因为它做不到

remove算法的签名是这样的：

```c++
template<class ForwardIterator, class T>
ForwardIterator remove(ForwardIterator first, ForwardIterator last,const T& value);
```

它接受的参数是迭代器，所以它并不知道自己操作的具体容器是什么(没有办法从一个迭代器获取对应于它的容器)，所以remove是没有办法从容器中删除元素的，唯一能从容器中删除元素的办法是调用容器的某种erase形式的成员函数，**remove真正做的是移动指定区间中的元素，直到所有"不删除的"元素在区间的开头，将容器氛围两个部分A|B，A是所有不被删除的原始，B则不太确定，返回的迭代器指向A的最后一个元素的下一个位置**

B则不太确定是因为remove的实现往往是：遍历整个容器，将要删除的元素覆盖为后面要保留的值，所以正确的做法应该是：

```c++
vector<int> v;
//remove(v.begin(),v.end(),99) 错误的做法
v.erase(remove(v.begin(),v.end(),99),v.end()); //正确的做法
```

ps：remove_if和unique算法也是一样的原理

从这几个函数的执行原理引申出另一点：对于存储动态分配的内存对象的指针的容器慎用这个几个算法，如果无法避免要使用，要确保自己在调用这几个算法之前已经将需要清除的对象的内存释放过了，否则极易造成内存泄露，当然如果是引用计数的智能指针则还是可以安全的使用的

### 注意哪些算法需要有序区间

需要有序区间的算法

binary_search

upper_bound 、lower_bound 、equal_range

set_union 、set_difference 、set_intersection 、set_symmetric_difference

merge 、inplace_merge

includes  

unique 和 unique_copy一般用于有序区间，但不要求一定是有序区间

### 把仿函数类设计为用于值传递

C++不允许把函数作为参数传递给其他对象(在C++中函数是二等公民)，你只能传指针(函数指针)或者函数对象，比如：

```c++
template<class InputIterator, class Function>
Function for_each(InputIterator first, InputIterator last, Function f);
```

指针是采用值传递的，因为STL函数对象是在指针之后才成型的，所以STL的**习惯**（不是必须）是传给函数和从函数返回时函数对象也是值传递的。这就使函数类的设设计最好需要满足以下一些条件：

1. 对象足够小，因为值传递是通过拷贝进行的，如果对象很大则会浪费很多性能
2. 对象必须是单态的，不能用虚函数，因为派生类对象传递给父类型参数的时候会造成对象切割，导致派生部分被切割掉了

然而想要做到上面两条也不太容易，但是有一个比较好的可行的办法是，函数对象存储另一个对象的指针，这个指针所指的对象可以很大，也可以有虚函数，用这个对象来存储你需要存储的大量数据和完成多态就可以了

### 用纯函数做判断式

纯函数：函数的返回结果只和传入的参数相关，传入的参数相同的时候无论何时调用，调用多少次得到的结果都是一样的

如果我们的判断式不是纯函数会有什么问题呢，我们看一个例子

```c++
class TestPredicate {
		bool operator()(const int & val) {
       return ++callTimes == 3;
    }
  private:
  	int callTimes;
}

vector<int> v;
//加入vector中的数据是1到10
v.erase(remove_if(v.begin,v.end(),TestPredicate()),v.end());

//remove_if可能是这么实现的
template <typename FwdIterator, typename Predicate>
FwdIterator remove_if(FwdIterator begin, FwdIterator end, Predicate p) {
	begin = find_if(begin, end, p); //找到了第三个数据，begin指向3
	if (begin == end) 
		return begin; 
	else {
		FwdIterator next = begin;//next指向3
		//将第4个数据到最后一个数据之间的所有数据如果满足p，则删除掉(将剩余的数据copy到begin也就是第四个数据及之后的位置)
		//将4到10之间的数据应用p进行判断，如果满足p则删除，将剩余的数据向前覆盖4之后的位置，调用完之后v中的数据是{1,2,4,5,7,8,9,10,9,10}，remove_copy_if返回的迭代器指向第二个9所在的位置
		return remove_copy_if(++next, end. begin, p); 
	}
}
```

因为Predicate是用值传递的，所以find_if(begin, end, p)中的p是一个"拷贝",p的callTimes=0，当find_if返回时callTimes=3，之后remove_if继续调用remove_copy_if，而这个时候p的callTimes还是0(之前被find_if改变的p是拷贝出来的副本)，当remove_copy_if运行返回之后实际上就erase掉了第3和第6个数据，而不止是第三个数据

### 了解使用ptr_fun、mem_fun和mem_fun_ref的原因

如果我有一个函数f和一个对象x，我希望在x上调用f，而且我在x的成员函数之外，C++有三种做法

```c++
f(x); //#1 f不是x的成员函数
x.f(); //#2 f是x的成员函数
p->f(); //#3 f是一个成员函数，而且p是一个指向对象的指针
```

假设我们有一个类：

```c++
class Test{
public:
    Test(string n) : name(n) {}
    ~Test(){}
    void hello() {
        cout << "hello " << name << endl;
    }
private:
  string name;
};
```

当我们需要遍历一个Test的vector容器，并让容器中的每个对象都调用一次hello我们需要这么做：

```c++
for_each(v.begin(),v.end(),&Test::hello);
```

然而这并不能通过编译，这无法使用C++中提供的三种做法的任何一种来实现函数的调用(v中的对象在for_each中是作为被调用函数的参数来使用的)，我们需要这样才可以：

```c++
for_each(v.begin(),v.end(),mem_fun(&Test::hello));
```

为什么这样可行，我们可以看看mem_fun的大体实现原理是怎么样的：

```c++
//pmf是一个成员函数指针，指向C类型类的某个成员函数
template<typename R, typename C>
mem_fun_t<R,C> mem_fun(R(C::*pmf)());

//仿函数对象类，存储一个其他类的成员函数指针，operator()方法将使用参数调用该参数类型的成员函数指针
template<typename R, typename C>
class mem_fun_t {
  typedef R(C::*pmf)();
  
public:
  R operator()(C c){
    return (c.p)()
  }
  R operator()(C * pc) {
    return (pc->p)();
  }
private:
  pmf p;
}
```

为了便于理解上面那段代码，我写了一个使用对象调用其成员函数指针的范例：

```c++
#include <iostream>
#include <string>
using namespace std;

class Test{
public:
    void print() {
        cout << "hello" << endl;
    }
};

typedef void(Test::*TestMemFunc)();

int main() {
    Test t;
    Test * pt = &t;
    TestMemFunc memFunc = &Test::print;
    (pt->*memFunc)();
    return 0;
}
```

### 使用stl的一些建议

+ 尽量使用算法代替手写循环(不是必须，还是得根据场景来决定)

> 1. 效率：算法通常比程序员写的循环更高效，因为算法的实现者往往更了解所操作的容器的内在实现，算法的具体实现一般都会根据具体的容器实现进行优化
>
> 2. 正确性：写循环比调用算法更容易产生错误，比如写循环给序列容器增加或者删除元素的时候会导致容器的迭代器失效，这个时候循环就需要自己去处理迭代器失效问题，而这往往会被很多程序员忽略，而算法则会自动帮忙处理好这些问题
> 3. 可维护性：算法通常比循环更简洁，更直观。算法的名字本身就具有了很好的功能说明性，而如果是循环的话你得仔细研究循环的实现才知道循环具体做了什么，这样会让代码更清晰。

在算法调用与手写循环正在进行的较量中，关于代码清晰度的底线是：这完全取决于你想在循环里做的是什么。如果你要做的是算法已经提供了的，或者非常接近于它提供的，调用泛型算法更清晰。如果循环里要做的事非常简单，但调用算法时却需要使用绑定和适配器或者需要独立的仿函数类，你恐怕还是写循环比较好。最后，如果你在循环里做的事相当长或相当复杂，天平再次倾向于算法。因为长的、复杂的通常总应该封装入独立的函数

+ 尽量使用成员函数代替同名算法，因为成员函数更快，行为也与对应的容器更一致

  下面通过一些列子来说明：

```c++
set<int> s; //假如有100w个元素
//更快,行为更一致，find、count、lower_bound，upper_bound等也是一样的原理
s.find(123456)；//o(logn)时间复杂度，使用“等价”进行查找
std::find(s.begin(),s.end(),123456); //o(n)时间复杂度，使用“相等”进行查找
//行为更一致这个需要特别说明一下：如果是map或者multi_map，"等价"往往是用key来作为判断的，而"相等"却是同时依赖key和value来做判断的，成员函数依赖"等价"，算法有些则依赖"相等"，当让你可以自己传入比较函数或者仿函数对象来实现"相等"，但这意味着你要写更多的代码
```

需要特别说明的是：每个被list作了特化的算法(remove、remove_if、unique、sort、merge和reverse)都要拷贝对象，而list的特别版本什么都没有拷贝;它们只是简单地操纵连接list的节点的指针。算法和成员函数的算法复杂度是相同的。list的成员函数和同名的算法实现上是有很大的差别的，比如list的remove，remove_if，unique这些成员函数是真的会实际执行他们名字所指定的操作的，而同名的算法却不是真的执行这些操作，同名算法要达到相同的效果在调用这些函数后还必须调用一次erase

+ 注意count、find、binary_search、lower_bound、upper_bound和equal_range的区别

> 1. binary_search、lower_bound、upper_bound和equal_range是基于"有序"这个基础的，他们提供对数时间复杂度，而count，count_if，find，find_if提供线性时间复杂度；
> 2. find在找到第一个后就立即返回，而count需要遍历全部元素；
> 3. find，find_if，count，count_if依赖"相等"，binary_search、lower_bound、upper_bound和equal_range依赖"等价"

下面是一张算法和成员函数的选择参考表格

----

你想知道的                                         使用的算法                                       使用的成员函数

----

| 期望值是否存在                                          | 无序区间   | 有序区间             | set/map     | multiset/multimap |
| ------------------------------------------------------- | ---------- | -------------------- | ----------- | ----------------- |
| 是否存在                                                | find       | binary_search        | count       | find              |
| 期望值是否存在? 如果有，第一个等 于这个值的对象在 哪里? | find       | equal_range          | find        | find/lower_bound  |
| 第一个不在期望值 之前的对象在哪 里                      | find_if    | lower_bound          | lower_bound | lower_bound       |
| 第一个在期望值之 后的对象在哪里                         | find_if    | upper_bound          | upper_bound | upper_bound       |
| 有多少对象等于期望值                                    | count      | equal_range+distance | count       | count             |
| 等于期望值的所有 对象在哪里                             | find(迭代) | equal_range          | equal_range | equal_range       |

+ 考虑使用函数对象替代函数作为算法的参数

  ```c++
  //#1 使用函数对象
  vector<double> v;
  sort(v.begin(), v.end(), greater<double>());
  //#2 使用函数
  inline bool doubleGreater(double d1, double d2) {
  	return dl > d2; 
  }
  sort(v.begin(), v.end(), doubleGreater);
  ```

\#1的greater\<double\>::operator()是内联的，所以编译器在编译的时候就已经将代码内联展开到sort函数中了

\#2的doubleGreater被默默的转化成了函数指针放到sort中调用（stl不支持传递函数，函数在C++中是二等公民），即使doubleGreater申明成了inline，但是实际sort中用的是函数指针调用