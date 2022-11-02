# Effective C++ 阅读笔记

### 1. 让自己习惯 C++

#### 条款 01：视 C++ 为一个语言联邦

1）今天的 C++ 是个多重范型编程语言，一个同时支持过程形式、面向对象形式、函数形式、泛型形式、元编程形式的语言。

2）将 C++ 视为一个由相关语言组成的联邦而非单一语言。这些相关语言被称为次语言，共四个次语言。

* C。区块（blocks）、语句（statements）、预处理器（preprocessor）、内置数据类型（built-in data types）、数组（arrays）、指针（pointers）统统来自 C。

* Object-Oriented C++。这部分有 classes，封装（encapsulation)、继承（inheritance）、多态（polymorphism）、virtual 函数等。

* Template C++。

* STL。STL 是个 template 程序库，它对容器（containers）、迭代器（iterators）、算法（algorithms）以及函数对象（function objects）的规约有极佳的紧密配合与协调。

#### 条款 02：尽量以 const，enum，inline 替换 #define

1）如代码 `#define ASPECT_RATIO 1.653` ，记号名称 ASPECT_RATIO 也许从未被编译器看见，也许在编译器处理之前就已经被预处理器移走了，于是该记号名称可能没有进入记号表，此时会导致错误。解决方法是以一个常量替换宏`const double AspectRatio = 1.653;` 作为一个语言常量，该变量会被编译器看到而进入记号表。使用 const 可能导致较小量的目标码，因为预处理器会盲目将宏定义替换为 1.65，导致目标码中出现多份 1.65。

2）当定义常量指针时，由于常量定义式通常被放在头文件内（便于被不同源码含入），因此有必要将指针声明为 const。

`const char* const authorName = "Scott Meyers";` 

3）当定义 class 的专属常量时，为了将常量的作用域限定在 class 内，必须让它成为 class 的成员，为了确保此常量至多只有一份实体，必须定义为 static。

`class Player {` 

`private:` 

​	`static const int NumTurns = 5;  // 常量声明式` 

​	`int scores[NumTurns];  // 使用该常量` 

​	`…` 

`};` 

通常对所使用的任何东西提供一个定义式，但如果是个 class 专属常量且为整数类型，则只要不取地址就无须提供定义式。定义式为：

`const int GamePlayer::NumTurns;` 

此定义式应该放在实现文件而非头文件。 

无法利用 #define 创建一个 class 专属常量，因为 #define 不重视作用域，一旦定义其后都能使用。#define 也不能提供任何封装性。

4）旧式编译器也许不允许 static 成员在其声明式上获得初值。而编译器必须在编译期间知道数组的大小，上述例子可采用 the enum hack 补偿做法，其理论基础是“一个属于枚举类型的数值可权充 ints 被使用”。

`class GamePlayer {` 

`private:` 

​	`enum { NumTurns  = 5 };` 

​	`int scores[NumTurns];` 

​	`…` 

`};`  

enum 不能取地址。

5）使用 template inline 函数代替 #define 宏。

`#define CALL-WITH-MAX(a, b) f((a) > (b) ? (a) : (b))` 

这种宏有诸多缺点，应进行如下替换：

`template<typename T>` 

`inline void CALL_WITH_MAX(const T&a, const T&b)` 

`{` 

​	`f(a > b ? a : b);` 

`}` 

#### 条款 03：尽可能使用 const

1）如果关键字 const 出现在星号左边，表示被指物是常量；如果出现在星号右边，表示指针自身是常量；如果出现在星号两边，表示被指物和指针两者都是常量。如果被指物是常量，类型与 const 哪个放前面意义相同。

`void f1(const Widget* pw);` 

`void f2(widget const* pw);` 

2）声明迭代器为 const 就像声明指针为 const 一样，表示这个迭代器不得指向不同的东西，但它所指的东西的值是可以改动的。如果希望迭代器所指的东西不可被改动，需要的是 const_iterator。

`const std::vector<int>::iterator iter = vec.begin(); // 作用像个 T* const`  

`std::vector<int>::const_iterator citer = vec.begin(); // 作用像个 const T*` 

3）令函数返回一个常量值，往往可以降低因客户错误而造成的意外，而又不至于放弃安全性和高效性。  

4）如果想要在 const 成员函数中修改变量的值，则需要将这些变量定义为 mutable 的。

`class CTextBlock {` 

`public:` 

​	`std::size_t length() const;` 

`private:` 

​	`mutable std::size_t textLength;` 

​	`mutable bool lengthIsValid;`  

`};` 

`std::size_t CTextBlock::length() const` 

`{` 

​	`if (!lengthIsValid) {` 

​			`textLength = std::strlen(pText);` 

​			`lengthIsValid = true;` 

​	`}` 

​	`return textLength;` 

`};`  

5）当 const 和 non-const 成员函数有着实质等价的实现时，令 non-const 版本调用 const 版本可以避免代码重复。

`class TextBlock {` 

`public:` 

​	`const char& operator[](std::size_t position) const` 

​	`{` 

​			`return text[position];` 

​	`}`  

​	`char& operator[](std::size_t position)`  

​	`{`  

​		`return`  

​			`const_cast<char&> (`  

​				`static_cast<const TextBlock&>(*this)` 

​				`[position]`  

​			`);`  

​	`}` 

`};`  

#### 条款 04：确定对象被使用前已先被初始化

1）永远在使用对象之前先将它初始化。对于无任何成员的内置类型，必须手工完成此事。

2）内置类型以外的任何其他东西，初始化责任落在构造函数身上。重要的是不要混淆赋值和初始化。

`ABEntry::ABEntry(…) // 赋值`  

`{` 

​	`thename = name;` 

​	`thephones = phones;` 

​	`num = 0;` 

`}` 

`ABEntry::ABEntry(…) // 初始化`    

​	`:thename = name;`  

​		`thephones = phones;`   

​		`num = 0;`     

`{ }`  

初始化的发生时间要早于赋值。因此赋值版本在初始化之前会调用 default 构造函数设初值，再进行 copy assignment。对于内置型对象如 num，初始化和赋值的成本相同。

3）为避免需要记住成员变量何时必须在成员初始值列表中初始化，何时不需要，最简单的做法就是：总是使用成员初始列。初始列列出的成员变量，其排列次序应该和它们在 class 中的声明次序相同（同一个类中成员变量总是以其声明的次序被初始化）。

4）static 对象，其寿命从被构造出来直到程序结束为止，因此 stack 和 heap-based 对象都被排除。这种对象包括 global 对象、定义于 namespace 作用域内的对象、在 class 内、在函数内、以及在 file 作用域内被声明为 static 的对象。函数内的 static 对象称为 local static 对象，其他 static 对象称为 non-local static 对象。程序结束时 static 对象会被自动销毁，也就是它们的析构函数会在 main() 结束时被自动调用。

5）C++ 对“定义于不同编译单元内的 non-local static 对象”的初始化次序并无明确定义。如果某编译单元（单一源码文件加上其所含入的头文件）内某个 non-local static 对象的初始化动作使用了另一编译单元内的某个 non-local static 对象，它所用到的这个对象可能尚未被初始化。解决方法是将每个 non-local static 对象搬到自己的专属函数内（该对象在此函数内被声明为 static）。这些函数返回一个 reference 指向它所含的对象。然后用户直接调用这些函数，而不直接指涉这些对象。其原理是 C++ 保证函数内的 local static 对象会在该函数被调用期间，首次遇上该对象的定义式时被初始化。

`Directory& tempDir() // 函数替换 tempDir 对象` 

`{` 

​	`static Directory td;` 

​	`return td;` 

`}` 

函数往往比较简单，如上第一行定义并非初始化一个 local static 对象，第二行返回它。

### 2. 构造/析构/赋值运算

#### 条款 05：了解 C++ 默默编写并调用哪些函数

1）编译器会暗自为 class 声明一个 default 函数、一个 copy 构造函数、一个 copy assignment 操作符和一个析构函数。所有这些函数都是 public 且 inline 的。编译器产出的析构函数是个 non-virtual，除非这个 class 的 base class 自身声明有 virtual 析构函数。

2）copy 构造函数和 copy assignment 操作符，编译器创建的版本只是单纯地将来源对象的每一个 non-static 成员变量拷贝到目标对象。若要在一个“内含 reference 成员”或是“内含 const 成员”的 class 内支持赋值操作，必须自己定义 copy assignment 操作符。还有一种情况是如果某个 base classes 将 copy assignment 操作符声明为 private，编译器将拒绝为其 derived classes 生成一个 copy assignment 操作符。

#### 条款 06：若不想使用编译器自动生成的函数，就该明确拒绝

1）为驳回编译器自动提供的机能，可以将 copy 构造函数或 copy assignment 操作符声明为 private。借由明确声明一个成员函数，阻止了编译器暗自创建其专属版本；令这些函数为 private，得以阻止人们调用它。

`class HomeForSale {` 

`public:` 

​	`…` 

`private:` 

​	`…` 

​	`HomeForSale(const HomeForSale&);` 

​	`HomeForSale& operator=(const HomeForSale&);` 

`};`  

此时 member 函数和 friend 函数还是可以调用。解决方法是设计一个专门为了阻止 copying 动作的 base class。

`class Uncopyable {` 

`protected:` 

​	`Uncopyable() { }` 

​	`~Uncopyable() { }` 

`private:` 

​	`Uncopyable(const Uncopyable&);` 

​	`Uncopyable& operator=(const Uncopyable&);` 

`};` 

#### 条款 07：为多态基类声明 virtual 析构函数

1）polymorphic（带多态性质的）base classes 应该声明一个 virtual 析构函数。如当 derived class 对象经由一个 base class 指针被删除，而该 base class 带着一个 non-virtual 析构函数，其结果未有定义——实际执行时通常发生的是对象的 derived 成分没被销毁。给 base class 一个 virtual 析构函数，删除时就会销毁整个对象。

`class TimeKeeper {` 

`public:` 

​	`TimeKeeper();` 

​	`virtual ~TimeKeeper();` 

​	`…` 

`};` 

`TimeKeeper* ptk = getTimeKeeper();` 

`…` 

`delete ptk;`  

如果 class 带有任何 virtual 函数，它就应该拥有一个 virtual 析构函数。

2）有时希望有抽象 class，但手上没有任何 pure virtual 函数，可以声明一个 pure virtual 析构函数并提供一份定义。

`class AWOV {` 

`public:` 

​	`virtual ~AWOV() = 0;` 

`};` 

`AWOV::~AWOV() { }` 

3）classes 的设计目的如果不是为了作为 base classes 使用，或不是为了具备多态性，就不该声明 virtual 析构函数。

#### 条款 08：别让异常逃离析构函数

1）析构函数绝对不要吐出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下它们（不传播）或结束程序。

2）如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么 class 应该提供一个普通函数（而非在析构函数中）执行该操作。

#### 条款 09：绝不在构造和析构过程中调用 virtual 函数

1）在构造和析构期间不要调用 virtual 函数，因为这类调用从不下降至 derived class（比起当前执行构造函数和析构函数的那层）。根本原因是在 derived class 对象的 base class 构造期间，对象的类型是 base class 而不是 derived class。解决方法是在构造期间，可以借由“令 derived classes 将必要的构造信息向上传递给 base class 构造函数”替换之而加以弥补。

#### 条款 10：令 operator= 返回一个 reference to *this

1）为了实现“连锁赋值”，赋值操作符必须返回一个 reference 指向操作符的左侧实参。这个协议也适用于赋值相关运算。

`class Widget {` 

`public:` 

​	`…` 

​	`Widget& operator+=(const Widget& rhs)` 

​	`{` 

​			`…` 

​			`return *this;` 

​	`}` 

​	`Widget& operator=(int rhs)` 

​	`{` 

​			`…` 

​			`return *this;` 

​	`}` 

`};` 

#### 条款 11：在 operator= 中处理“自我赋值”

1）“自我赋值”发生在对象被赋值给自己时。一般而言如果某段代码操作 points 或 references 而它们被用来“指向多个相同类型的对象”，就需要考虑这些对象是否为同一个。实际上两个对象只要来自同一个继承体系，它们甚至不需声明为相同类型就可能指向同一个东西。

2）假设建立一个 class 用来保存一个指针指向一块动态分配的位图。

`class Bitmap {…};` 

`class Widget {` 

​	`…` 

`private:` 

​	`Bitmap* pb;` 

`};` 

下面为四种 operator= 版本。

版本一：

`Widget& Widget::operator=(const Widget& rhs)` 

`{` 

​	`delete pb; // 把自己释放了` 

​	`pb = new Bitmap(*rhs.pb);` 

​	`return *this;` 

`}` 

这个版本不具备“自我赋值安全性”，也不具备“异常安全性”。因为函数内 *this 和 rhs 有可能是同一个对象，此时 delete 销毁了当前对象和 rhs 的 bitmap，导致最后指向了一个已被删除的对象。

版本二：

`Widget& Widget::operator=(const Widget& rhs)` 

`{` 

​	`if (this == &rhs) return *this;` 

​	`delete pb;`   

​	`pb = new Bitmap(*rhs.pb);` 

​	`return *this;` 

`}` 

这个版本通过“证同测试”达到“自我赋值”的检验目的。if 表明如果是自我赋值，就不做任何事。但仍不具备“异常安全性”，因为如果 new 时出现异常，Widget 最终会持有一个指针指向一块被删除的 Bitmap。

版本三：

`Widget& Widget::operator=(const Widget& rhs)` 

`{` 

​	`Bitmap* pOrig = pb // 记住原先的 pb;`   

​	`pb = new Bitmap(*rhs.pb); // 令 pb 指向 *pb 的一个复件`  

​	`delete pOrig; // 删除原先的 pb`  

​	`return *this;` 

`}` 

版本四（copy and swap 技术）：

`Widget& Widget::operator=(const Widget& rhs)` 

`{` 

​	`Widget temp(rhs); // 为 rhs 数据制作一份复件`  

​	`swap(temp) // 将 *this 数据和上述复件的数据交换`  

​	`return *this;` 

`}` 

#### 条款 12：复制对象时勿忘其每一个成分

1）copying 函数应该确保复制“对象内的所有成员变量”及“所有 base class 成分”。

`class Customer {` 

`public:` 

​	`…` 

​	`Customer(const Customer& rhs);` 

​	`Customer& operator=(const Customer& rhs);` 

`private:` 

​	`std::string name;` 

`};`  

`Customer::Customer(const Customer& rhs)` 

​	`: name(rhs.name);` 

`{ }` 

`Customer& Customer::operator=(const Customer& rhs)` 

`{` 

​	`name = rhs.name;` 

​	`return *this;` 

`}` 

此时若 customer 类中添加成员，则 copying 也需要修改。

`class PCustomer: public Customer {`  

`public:` 

​	`…` 

​	`PCustomer(const PCustomer& rhs);` 

​	`PCustomer& operator=(const PCustomer& rhs);` 

`private:` 

​	`int priority;` 

`};`  

`PCustomer::PCustomer(const PCustomer& rhs)` 

​	`: Customer(rhs), // 调用 base class 的 copy 构造函数` 

​	 `  priority(rhs.priority)`  

`{ }` 

`PCustomer& PCustomer::operator=(const PCustomer& rhs)` 

`{` 

​	`Customer::operator=(rhs); // 对 base class 成分进行赋值动作` 

​	`priority = rhs.priority;` 

​	`return *this;` 

`}` 

2）不要尝试以某个 copying 函数实现另一个 copying 函数。应该将共同机能放进第三个函数中，并由两个 copying 函数共同调用。

### 3. 资源管理

#### 条款 13：以对象管理资源

1）把资源放进对象内，便可倚赖 C++ 的“析构函数自动调用机制”确保资源被释放。RAII 对象获得资源后立刻放进管理对象内，而后运用析构函数确保资源被释放。

2）auto_ptr 被销毁时会自动删除所指之物。若通过 copy 构造函数或 copy assignment 操作符复制它们，它们会变成 null，而复制所得的指针取得资源的唯一拥有权。

`std::auto_ptr<Investment> pInv1(create()); // pInv1 指向 create 返回物` 

`std::auto_ptr<Investment> pInv2(pInv1); // pInv1 被设为 null，pInv2 指向对象` 

`pInv1 = pInv2; // pInv1 指向对象，pInv2 被设为 null` 

3）auto_ptr 的替代方案是“引用计数型智慧指针”（RCSP）。RCSP 持续追踪共有多少对象指向某笔资源，并在无人指向它时自动删除该资源。

`void f()` 

`{` 

​	`std::tr1::shared_ptr<Investment>` 

​	`pInv1(create()); // pInv1 指向 create 返回物` 

​	`std::tr1::shared_ptr<Investment>` 

​	`pInv2(pInv1); // pInv1 和 pInv2 指向同一对象` 

​	`pInv1 = pInv2; // 无任何改变` 

​	`…`  

`} // pInv1 和 pInv2 被销毁` 

4）没有特别针对“C++ 动态分配数组”而设计的类似智慧指针的东西，因为 vector 和 string 几乎总是可以取代动态分配而得的数组。

#### 条款 14：在资源管理类中小心 copying 行为

1）复制 RAII 对象必须一并复制它所管理的资源，所以资源的 copying 行为决定 RAII 对象的 copying 行为。

2）普遍而常见的 RAII class copying 行为是：抑制 copying、施行引用计数法。

`class lock: private Uncopyable { // 条款6中禁止复制` 

`public:` 

​	`…` 

`};` 

tr1::shared_ptr 使用引用计数法，同时允许指定“删除器”。删除器是一个函数或函数对象，当引用次数为 0 时被调用。

`class lock {` 

`public:` 

​	`explicit lock(Mutex* pm)` 

​		`: mutexPtr(pm, unlock) // 以 unlock 函数为删除器` 

​	`{` 

​			`lock(mutexPtr.get()); // 条款15提到“get”` 

​	`}` 

`private:` 

​	`std::tr1::shared_ptr<Mutex> mutexPtr;` 

`};`  

3）其他行为也可能被实现。如通过深拷贝复制底部资源、转移底部资源的拥有权（如 auto_ptr）。

#### 条款 15：在资源管理类中提供对原始资源的访问

 1）APIs 往往要求访问原始资源，所以每一个 RAII class 应该提供一个“取得其所管理资源”的办法。

2）显示装换。tr1::shared_ptr 和 auto_ptr 都提供一个 get 成员函数，用来执行显示转换，也就是它会返回智能指针内部的原始指针（的复件）。

`int days = daysheld(pInv.get());` 

3）隐式转换。一般而言显示转换比较安全，但隐式转换对客户比较方便。

`class Font {` 

`public:` 

​	`…` 

​	`FontHandle get() const { return f; } // 显示转换函数` 

​	`operator FontHandle() const // 隐式转换函数` 

​	`{ return f; }` 

​	`…` 

`private:` 

​	`FontHandle f;` 

`};` 

#### 条款 16：成对使用 new 和 delete 时要采取相同形式

1）当使用 new，有两件事发生：第一，内存被分配出来；第二，针对此内存会有一个（或更多）构造函数被调用。当使用 delete，也有两件事发生：针对此内存会有一个（或更多）析构函数被调用，然后内存才被释放。

2）如果在 new 表达式中使用 []，必须在相应的 delete 表达式中也使用 []。如果在 new 表达式中不使用 []，一定不要在相应的 delete 表达式中使用 []。

3）以 new 创建 typedef 类型对象时，要考虑 delete 形式。

`typedef std::string AddressLines[4];` 

`std::string* pal = new AddressLines;` 

`delete pal; // 行为未有定义` 

`delete [ ] pal; // 正确` 

#### 条款 17：以独立语句将 newed 对象置入智能指针

1）以独立语句将 newed 对象存储于（置入）智能指针内。如果不这样做，一旦异常被抛出，有可能导致难以察觉的资源泄露。如下：

`process(std::tr1::shared_ptr<Widget>(new Widget), priority());` 

在调用 process 前，编译器必须创建代码，做以下三件事：

* 调用 priority

* 执行 “new Widget”

* 调用 tr1::shared_ptr 构造函数

由于不知道调用 priority 的顺序。若它在第二位执行，则调用导致异常的情况下，new Widget 返回的指针将会遗失，因为它尚未被置入 tr1::shared_ptr 内。避免这类问题的办法很简单：使用分离语句。

`std::tr1::shared_ptr<Widget> pw(new Widget);` 

`process(pw, priority());` 

### 4. 设计与声明

#### 条款 18：让接口容易被正确使用，不易被误用

1）好的接口很容易被正确使用，不容易被误用。“促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容。“阻止误用“的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任。

2）DLL 问题发生于“对象在动态连接程序库（DLL）中被 new 创建，却在另一个 DLL 内被 delete 销毁”。tr1::shared_ptr 没有这个问题。

#### 条款 19：设计 class 犹如设计 type

1）class 的设计就是 type 的设计。在定义一个新 type 之前，要考虑以下问题：

* 新 type 的对象应该如何被创建和销毁。这影响的是 class 的构造函数、析构函数、内存分配函数和释放函数的设计。
* 对象的初始化和赋值该有什么样的区别。这决定了构造函数和赋值操作符（assignment）的行为。
* 新 type 的对象如果被 pass by value，意味着什么。copy 构造函数用来定义一个 type 的 pass by value 如何实现。
* 什么是新 type 的合法值。
* 新 type 是否需要配合某个继承图系。即会受到其他 class 的束缚，特别是受到“它们的函数是 virtual 或 non-virtual 的影响”。
* 新 type 需要什么样的转换。
* 什么样的操作符和函数对此新 type 而言是合理的。这决定将声明哪些函数，其中哪些该是 member 的。
* 什么样的标准函数应该驳回。那些正是必须声明为 private 者。
* 谁该取用新 type 的成员。
* 什么是新 type 的“未声明接口”。
* 新 type 有多一般化。这决定是该定义一个 class 或是一个 class template。
#### 条款 20：宁以 pass-by-reference-to-const 替换 pass-by-value

1）尽量以 pass-by-reference-to-const 替换 pass-by-value。前者通常比较高效，并可避免切割问题。对象切割问题指当一个 derived class 对象以 by value 方式传递并被视为一个 base class 对象，base class 的 copy 构造函数会被调用，而“造成此对象行为像个 derived class 对象”的那些特化性质全被切割掉了，仅仅留下一个 base class 对象。

`class Window {` 

`public:` 

​	`…` 

​	`std::string name() const;` 

​	`virtual void display() const;` 

`};` 

`class WindowWith: public Window {` 

`public:` 

​	`…` 

​	`virtual void display() const;` 

`};` 

此时若有函数：
`void print(Window w)` 

`{` 

​	`w.display();` 

`}` 

则函数内调用的 display 总是 Window::display。解决办法就是以 by reference-to-const 的方式传递 w。

`void print(const Window& w)`  

`{` 

​	`w.display();` 

`}` 

references 往往以指针实现出来，因此 pass by reference 通常意味真正传递的是指针。

2）内置类型（如 int）、STL 对象和函数对象使用 pass-by-value 往往比较恰当。

#### 条款 21：必须返回对象时，别妄想返回其 reference

1）绝不要返回 pointer 或 reference 指向一个 local stack 对象，或返回 reference 指向一个 heap-allocated 对象，或返回 pointer 或 reference 指向一个 local static 对象而有可能需要多个这样的对象。

`class Rational {` 

`public:` 

​	`Rational(int num = 0, int den = 1);` 

​	`…` 

`private:` 

​	`int n, d;` 

​	`friend const Rational operator*`

​	`(const Rational& lhs, const Rational& rhs);`

`};` 

reference 只是个名称，代表某个既有对象。上述 operator*，若返回一个 reference，后者一定指向某个既有的 Rational 对象。此时必须自己创建该 Rational 对象。

`const Rational& operator* (const Rational& lhs,` 

​														`const Rational& rhs)` 

`{` 

​	`Rational result(lhs.n * rhs.n, lhs.d * rhs.d);` 

​	`return result;` 

`}` 

上述函数在 stack 空间创建对象。返回一个 reference 指向 result，但 result 是个 local 对象，而 local 对象在函数退出前被销毁了。

`const Rational& operator* (const Rational& lhs,` 

​														`const Rational& rhs)` 

`{` 

​	`Rational* result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);` 

​	`return *result;` 

`}` 

上述函数在 heap 内构造一个对象，并返回 reference 指向它。此时的问题是无法对 new 出来的对象实施 delete。

`const Rational& operator* (const Rational& lhs,` 

​														`const Rational& rhs)` 

`{` 

​	`static Rational result;` 

​	`result = …;`   

​	`return result;` 

`}` 

此时如果后面有类似 `if ((a * b) == (c * d))` ，其结果永远是 true。

综上，一个“必须返回新对象”的函数就不能返回其 reference。对于 operator*而言意味以下写法：

`inline const Rational operator*(const Rational& lhs,` 

​																`const Rational& rhs)` 

`{` 

​	`return Rational(lhs.n * rhs.n, lhs.d * rhs.d);` 

`}` 

#### 条款 22：将成员变量声明为 private

1）将成员变量声明为 private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供 class 作者以充分的实现弹性。

2）public 意味不封装，而几乎可以说，不封装意味着不可改变，特别是对被广泛使用的 classes 而言。假设有一个 public 成员变量，若最终取消了它，或许有不可知的大量的客户码都会被破坏。

3）protected 并不比 public 更具封装性。从封装的角度，其实只有两种访问权限：private（提供封装）和其他（不提供封装）。

#### 条款 23：宁以 non-member、non-friend 替换 member 函数

1）宁可拿 non-member non-friend 函数替换 member 函数。这样做可以增加封装性、包裹弹性和机能扩充性。

2）愈多东西被封装，愈少人可以看到它。愈少人看到它，就有愈大的弹性去变化它。考虑对象内的数据，愈少代码可以看到数据（也就是访问它），愈多的数据可被封装，也就愈能自由地改变对象数据。

3）namespace 和 classes 不同，前者可以跨越多个源码文件而后者不能。对于一些既不是 members 也不是 friends，但能为类提供便利的“便利函数”，比较自然的办法是让其成为一个 non-member 函数并位于类所在的同一个 namespace 内。

`namespace WebBrowserStuff {` 

​	`class WebBrowser { … }；` 

​	`void clearBrowser(WebBrowser& wb);` 

​	`…` 

`};` 

当有大量便利函数，分离它们的最直接方法是将不同函数声明于不同头文件。

`// 头文件 "webbrowser.h"--针对 class 自身及核心机能` 

`namespace WebBrowserStuff {` 

`class WebBrowser { … };` 

​	`…` 

`}` 

`// 头文件 "webbrowserbook.h"` 

`namespace WebBrowserStuff {` 

​	`…` 

`}` 

`// 头文件 "webbrowsercookies.h"` 

`namespace WebBrowserStuff {` 

​	`…` 

`}` 

#### 条款 24：若所有参数皆需类型转换，请为此采用 non-member 函数

1）若需要为某个函数的所有参数（包括被 this 指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个 non-menber。

`class Rational {` 

`public:` 

​	`Rational(int num = 0, int den = 1);` 

​	`int num() const;` 

​	`int den() const;` 

​	`const Rational operator*(const Rational& rhs) const;` 

`};` 

若进行混合式算数：

`result = oneHalf * 2; // 正确` 

`result = 2 * onehalf; // 错误`  

上述两个表达式相当于

`result = onehalf.operator*(2);` 

`result = 2.operator*(oneHalf);` 

对于第一个表达式，第二参数是整数 2，函数需要的实参是个 Rational 对象，这里发生了隐式类型转换（构造函数必须是 non-explicit）。即

`const Rational temp(2);` 

`result = oneHalf * temp;` 

当参数被列于参数列内，这个参数才是隐式转换的合格参与者。第二个表达式中，参数地位是 this 对象的隐喻参数，不是隐式转换的合格参与者，因此无法通过编译。

#### 条款 25：考虑写出一个不抛异常的 swap 函数

1）标准程序库提供的 swap 算法的典型实现：

`namespace std {` 

​	`template<typename T>` 

​	`void swap(T& a, T& b)` 

​	`{` 

​			`T temp(a);` 

​			`a = b;` 

​			`b = temp;` 	

​	`}` 

`}` 

2）通常不能改变 std 命名空间内的任何东西，但可以为标准 templates（如 swap）制造特化版本，使它专属于某个 class。

`class Widget {` 

`public:` 

​	`…` 

​	`void swap(Widget& other)` 

​	`{` 

​			`using std::swap;` 

​			`swap(pImpl, other,pImpl); // 只置换 Widget 的指针 plmpl` 

​	`}` 

`};` 

`namespace std {` 

​	`template<>` 

​	`void swap<Widget>(Widget& a, Widget&b)` 

​	`{` 

​			`a.swap(b);` 

​	`}` 

`}` 

3）如果提供一个 member swap，也该提供一个 non-member swap 用来调用前者。对于 classes，也要特化 std::swap。

`namespace WidgetStuff {` 

​	`…` 

​	`template<typename T>` 

​	`class Widget { … };` 

​	`…` 

​	`template<typename T>` 

​	`void swap(Widget<T>&a, Widget<T>& b)` 

​	`{` 

​			`a.swap(b);` 

​	`}` 

`}` 

4）调用 swap 时应针对 std::swap 使用 using 声明式，然后调用 swap 并且不带任何“命名空间资格修饰”。即

`using std::swap;` 

令标准的 swap 在函数内可用。C++ 首先在函数内寻找合适的 swap 函数，如果没有 T 专属的 swap 存在，就使用 std::swap。

5）为“用户定义类型”进行 std templates 全特化是好的，但千万不要尝试在 std 内加入某些对 std 而言全新的东西。

### 5. 实现

#### 条款 26：尽可能延后变量定义式的出现时间

1）尽可能延后的意义，不只应该延后变量的定义，直到非得使用该变量的前一刻为止，甚至应该尝试延后这份定义直到能够给它初始实参而已。如果这样，不仅能够避免构造和析构非必要对象，还可以避免无意义的 default 构造行为。更深一层说，以“具有明显意义之初值”将变量初始化，还可以附带说明变量的目的。

`std::string encrypt(const std::string& password)` 

`{` 

​	`using namespace std;` 

​	`string encrypted; // 若后面抛出异常，浪费了该变量的构造和析构成本。` 

​	`if (password.length() < min) {` 

​			`throw logic_error("short");` 

​	`}` 

​	`string encrypted;` 

​	`encrypted = password; // 需要调用一次 default 构造函数` 

​	`string encrypted(password); // 最好` 

`}` 

2）对于只在循环内使用的变量，将其定义于循环内或是循环外，需要考虑赋值成本和构造 + 析构成本。

#### 条款 27：尽量少做转型动作

1）旧式转型：`(T) expression`  `T (expression)` 

2）新式转型：

* `const_cast<T>(expression)` const_cast 通常被用来将对象的常量性转除。它也是唯一有此能力的 C++-style 转型操作符。
* `dynamic_cast<T>(expression)` dynamic_cast 主要用来执行“安全向下转型”，也就是用来决定某对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作。
* `reinterpret_cast<T>(expression)` reinterpret_cast 意图执行低级转型，实际动作可能取决于编译器。
* `static_cast<T>(expression)` static_cast 用来强迫隐式转换，例如将 non-const 转为 const 对象，将 int 转为 double，将 point-to-base 转为 point-to-derived。但它无法将 const 转为 non-const——这个只有 const_cast 才办得到。

宁可使用新式转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有分门别类的职掌。

3）如果可以，尽量避免转型。如果有个设计需要转型动作，试着发展无需转型的替代设计。如 dynamic_cast，通常是因为想在一个认定为 derived class 对象身上执行 derived class 操作函数，但手上却只有一个“指向 base”的 pointer 或 reference，只能靠它们来处理对象。有两个一般性做法可以避免这个问题。

第一，使用容器并在其中存储直接指向 derived class 对象的指针（通常是智能指针）。

`typedef std::vector<std::tr1::shared_ptr<Window> > VPW;` 

`VPW winPtrs;`

`for (VPW::iterator iter = winPtrs.begin();` 

 			`iter != winPtrs.end(); ++iter) {`

​	`if (SWindow* psw = dynamic_cast<SWindow*>(iter->get()))` 

​			`psw->blink();` 

`}` 

应该改为：

`typedef std::vector<std::tr1::shared_ptr<SWindow> > VPSW;` 

第二，在 base class 内提供 virtual 函数做想对各个派生类做的事。即“将 virtual 函数往继承体系上方移动”。

4）如果转型是必要的，试着将它隐藏于某个函数背后。客户随后可以调用该函数，而不需要将转型放进自己的代码内。

#### 条款 28：避免返回 handles 指向对象内部成分

1）避免返回 handles（包括 references、指针、迭代器）。遵守这个条款可增加封装性，帮助 const 成员函数的行为像个 const。

`class Point {` 

`public:` 

​	`Point(int x, int y);` 

​	`void setX(int newVal);` 

​	`void setY(int newVal);` 

​	`…` 

`};` 

`struct RectData {` 

​	`Point ulhc;` 

​	`Point lrhc;` 

`};` 

`class Rectangle {` 

`public:` 

​	`Point& upperLeft() const { return pData->ulhc; }` 

​	`Point& lowerRight() const { return pData->lrhc; }` 

​	`…` 

`private:` 

​	`std::tr1::shared_ptr<RectData> pData;` 

`};`  

这样的设计，upperLeft 和 lowerRight 被声明为 const 成员函数，但其返回的 references 指向 private 内部数据，调用者可以通过这些 references 更改内部数据，如：

`const Rectangle rec(cod1, cod2);` 

`rec.upperLeft().setX(50);` 

由此可见，返回 handles 指向对象内部会降低封装性。对于本例，修改方法是将两个成员函数声明为 const。

2）返回 handles 还可能导致 dangling handles（虚吊号码牌）：这种 handles 所指东西（的所属对象）不复存在。即 handles 比其所指对象更加长寿。

#### 条款 29：为“异常安全”而努力是值得的

1）当异常被抛出时，带有异常安全性的函数会：不泄露任何资源；不允许数据败坏。

2）异常安全函数提供以下三个保证之一：

* 基本承诺：如果异常被抛出，程序内的任何事物任然保持在有效状态下。没有任何对象或数据结构会因此而败坏，所有对象都处于一种内部前后一致的状态。然而程序的现实状态恐怕不可预料。
* 强烈保证：如果异常被抛出，程序状态不改变。调用这样的函数需要有这样的认知：如果函数成功，就是完全成功，如果函数失败，程序会回复到“调用函数之前”的状态。
* 不抛掷（nothrow）保证：承诺绝不抛出异常，因为它们总是能够完成它们原先承诺的功能。作用于内置类型身上的所有操作都提供 nothrow 保证。

3）强烈保证有个一般化的设计策略：copy and swap。原则很简单：为打算修改的对象（原件）做出一份副本，然后在那副本身上做一切必要修改。若有任何修改动作抛出异常，原对象仍保持未改变状态。待所有改变都成功后，再将修改过的那个副本和原对象在一个不抛出异常的操作中置换。

实现上通常是将所有“隶属对象的数据”从原对象放进另一个对象内，然后赋予原对象一个指针，指向那个所谓的实现对象。这种手法常被称为 pimpl idiom。

`class PrettyMenu {` 

`public:` 

​	`void change(std::istream& imgSrc);` 

`private:` 

​	`Mutex mutex;` 

​	`Image* bgImage;` 

​	`int imageChanges;` 

`};` 

`void PrettyMenu::change(std::istream& imgSrc)` 

`{` 

​	`lock(&mutex);` 

​	`delete bgImage;` 

​	`++imageChanges;` 

​	`bgImage = new Image(imgSrc);` 

​	`unlock(&mutex);` 

`}` 

上述代码很糟糕：一旦“new Image(imgSrc)”导致异常，对 unlock 的调用绝不会执行，bgImage 指向一个已被删除的对象，imageChanges 已被累加。使用 copy and swap 策略的典型写法：

`struct PMImpl {` 

​	`std::tr1::shared_ptr<Image> bgImage;` 

​	`int imageChanges;` 

`};` 

`class PrettyMenu {` 

`public:` 

​	`void change(std::istream& imgSrc);` 

`private:` 

​	`Mutex mutex;` 

​	`std::tr1::shared_ptr<PMImpl> pImpl;` 

​	`int imageChanges;` 

`};` 

`void PrettyMenu::change(std::istream& imgSrc)` 

`{` 

​	`using std::swap; // 条款25` 

​	`Lock(&mutex); // 获得 mutex 的副本数据` 

​	`std::tr1::shared_ptr<PMImpl> `

​			`pNew(new PMImpl(*pImpl));` 

​	`pNew->bgImage.reset(new Image(imgSrc)); // 修改副本` 

​	`++pNew->imageChanges;` 

​	`swap(pImpl, pNew);` 

`}` 

要注意的是，“强烈保证”并非对所有函数都可实现或具备现实意义。

4）函数提供的“异常安全保证”通常最高只等于其所调用之各个函数的“异常安全保证”中的最弱者。例如：

`void someFunc()` 

`{` 

​	`…` 

​	`f1();` 

​	`f2();` 

​	`…` 

`}` 

如果 f1 或 f2 的异常安全性比“强烈保证”低，就很难让 someFunc 成为“强烈异常安全”。

#### 条款 30：透彻了解 inlining 的里里外外

1）inline 函数背后的整体观念是，将“对此函数的每一个调用”都以函数本体替换之。这样做可能增加目标码大小，在一台内存有限的机器上，过度热衷 inlining 会造成程序体积太大。inline 造成的代码膨胀亦会导致额外的换页行为，降低指令高速缓存装置的击中率，以及伴随这些而来的效率损失。

2）inline 只是对编译器的一个申请，不是强制命令。这项申请可以隐喻提出，也可以明确提出。隐喻方式是将函数定义于 class 定义式内，即成员函数和 friend 函数默认是 inline 的。明确声明是在定义式（不能是声明式）前加上关键字 inline。

3）不要只因为 function templates 出现在头文件，就将它们声明为 inline。

4）一个表面上看似 inline 的函数是否真是 inline，取决于建置环境，主要取决于编译器。大多数编译器提供了一个诊断级别：如果它们无法将要求的函数 inline 化，会给出一个警告信息。编译器通常不对“通过函数指针而进行的调用”实施 inlining。

5）构造函数和析构函数往往是 inlining 的糟糕候选人。

`class Base {` 

`public:` 

​	`…` 

`private:` 

​	`std::string bm1,bm2;` 

`};` 

`class Derived: public Base {` 

`public:` 

​	`Derived() { }` 

​	`…` 

`private:` 

​	`std::string dm1, dm2, dm3;` 

`};` 

Derived 构造函数看起来是 inlining 的绝佳候选人。但实际上，C++ 对于“对象被创建和被销毁时发生什么事”做了各式各样的保证，事情如何发生是编译器的职责，但一定有某些代码让那些事情发生。那些代码肯定存在于某些地方，有时候就在构造和析构函数内。如上的 Derived 构造函数看起来为空，实际上可能为：

`Derived::Derived()` 

`{` 

​	`Base::Base();` 

​	`try { dm1.std::string::string(); }` 

​	`catch (…) {` 

​			`Base::~Base() // 销毁 base class 成分;` 

​			`throw // 传播该异常;` 

​	`}` 

​	`try { dm2.std::string::string(); }` 

​	`catch (…) {` 

​			`dm1.std::string::~string();` 

​			`Base::~Base();` 

​			`throw;` 

​	`}` 

​	`try { dm3.std::string::string(); }` 

​	`catch (…) {` 

​			`dm2.std::string::~string();` 

​			`dm1.std::string::~string();` 

​			`Base::~Base();` 

​			`throw;` 

​	`}` 

`}` 

上述代码并不能代表编译器真正制造出来的代码，但已能准确反映空白构造函数必须提供的行为。

6）将大多数 inlining 限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化。

#### 条款 31：将文件间的编译依存关系降至最低

1）支持“编译依存性最小化”的一般构想是：相依于声明式，不要相依于定义式。基于此构想的两个手段是 Handle classes 和 Interface classes。

2）main class 只内含一个指针成员，指向其实现类，这般设计常被称为 pimpl idiom。使用 pimpl idiom 的 classes，往往被称为 Handle classes。

`#include <string>` 

`#include <memory>` 

`class PersonImpl; // Person 实现类的前置声明` 

`class Date; // Person 接口用到的 classes 的前置声明` 

`class Person {` 

`public:` 

​	`Person(const std::string& name, const Date&` 

​					`birthday);` 

​	`std::string name() const;` 

​	`std::string birthDate() const;` 

​	`…` 

`private:` 

​	`std::tr1::shared_ptr<PersonImpl> pImpl; // 指向实现物` 

`};` 

Person 想要做点事情，办法之一是将它的所有函数转交给相应的实现类并由后者完成实际工作。如：

`#include "Person.h" // class 定义式`

`#include "PersonImpl.h"` 

`Person::Person(const std::string& name, ` 

​								`const Date& birthday)` 

​	`: pImpl(new PersonImpl(name, birthday))` 

`{ }` 

`std::string Person::name() const` 

`{` 

​	`return pImpl->name();` 

`}`  

3）不使用 pimpl idiom，可以令 class 成为一种特殊的 abstract base class，称为 Interface class。这种 class 的目的是详细一一描述 derived classes 的接口，因此它通常不带成员变量，也没有构造函数，只有一个 virtual 析构函数以及一组 pure virtual 函数，用来叙述整个接口。

`class Person {` 

`public:` 

​	`virtual ~Person();` 

​	`virtual std::string name() const = 0;` 

​	`virtual std::string birthDate() const = 0;` 

​	`…` 

`};` 

Interface class 的客户必须有办法为这种 class 创建新对象。他们通常调用一个特殊函数，此函数扮演“真正将被具现化”的那个 derived classes 的构造函数角色。这样的函数通常被称为 factory 函数或 virtual 构造函数。它们返回指针指向动态分配所得对象，该对象支持 Interface class 的接口。这样的函数又往往在 Interface class 内被声明为 static。

`class Person {` 

`public:` 

​	`…` 

​	`static std::tr1::shared_ptr<Person>` 

​		`create(const std::string& name,` 

​						`const Date& birthday)` 

`};` 

4）编译依存性最小化的本质：现实中让头文件尽可能自我满足，万一做不到，则让它与其他文件内的声明式（而非定义式）相依。其他每一件事都源自这个简单的设计策略：

* 如果使用 object references 或 object pointers 可以完成任务，就不要使用 objects。

* 如果能够，尽量以 class 声明式替换 class 定义式。

* 为声明式和定义式提供不同的头文件。

  `#include "datefwd.h" // 声明但未定义 class Date` 

  `Date today();` 

  `void clear(Date d); // 定义` 

5）程序头文件应该以“完全且仅有声明式”的形式存在。这种做法不论是否涉及 templates 都适用。

### 6. 继承与面向对象设计

#### 条款 32：确定你的 public 继承塑模出 is-a 关系

1）“public”继承意味 is-a。适用于 base classes 身上的每一件事情一定也适用于 derived classes 身上，因为每一个 derived class 对象也都是一个 base class 对象。

#### 条款 33：避免遮掩继承而来的名称

1）derived classes 内的名称会遮掩 base classes 内的名称。在 public 继承下从来没有人希望如此。因为编译器查找作用域时，在内层找到时就不会去外层寻找。

2）可以为原本会被遮掩的每个名称引入一个 using 声明式。

`class Base {` 

`private:` 

​	`int x;` 

`public:` 

​	`virtual void mf1() = 0;` 

​	`virtual void mf1(int);` 

​	`virtual void mf2();` 

​	`void mf3();` 

​	`void mf3(double);` 

`};` 

`class Derived: public Base {` 

`public:` 

​	`using Base::mf1; // 让 Base class 内名为 mf1 所有东西`

​												`可见（两个函数都可见）` 

​	`using Base::mf3;` 

​	`virtual void mf1();` 

​	`void mf3();`  

`};` 

3）使用转交函数也能让被遮掩的名称再见天日。假设 Derived 以 private 形式继承 Base，用转交函数可以指出想要继承的函数版本，即不像 using 那样只要名字相同即继承。

`class Derived: private Base {` 

`public:` 

​	`vurtual void mf1() // 转交函数` 

​	`{ Base::mf1(); } // 暗自成为 inline` 

​	`…` 

`};` 

inline 转交函数的另一个用途是为那些不支持 using 声明式的老旧编译器另辟一条新路，将继承而得的名称汇入 derived class 作用域内。

#### 条款 34：区分接口继承和实现继承

1）接口继承（继承声明）和实现继承不同。在 public 继承之下，derived classes 总是继承 base class 的接口。因此某个函数可施行于某 class 身上，一定也可施行于其 derived classes 身上。

`class Shape {` 

`public:` 

​	`virtual void draw() const = 0;` 

​	`virtual void error(const std::string& msg);` 

​	`int objectID() const;` 

​	`…` 

`};` 

`class Rectangle: public Shape { … };` 

`class Ellipse: public Shape { … };` 

2）pure virtual 函数只具体指定接口继承。Shape 类中的 draw 函数无法提供缺省实现，Derived classes 必须给出其实现。

3）简朴的（非纯）inpure virtual 函数具体指定接口继承及缺省实现继承。Shape 类中的 error 函数，如果 Derived classes 中没有自己的实现，可以使用 Shape 提供的缺省版本。

4）non-virtual 函数具体指定接口继承以及强制性实现继承。non-virtual 函数代表的意义是不变性凌驾特异性，所以它绝不该在 derived class 中被重新定义。

#### 条款 35：考虑 virtual 函数以外的其他选择

1）virtual 函数的替代方案：

* 使用 non-irtual interface（NVI）手法，那是 Template Method 设计模式的一种特殊形式。它以 public non-virtual 成员函数包裹较低访问性（private 或 protected）的 virtual 函数。
* 将 virtual 函数替换为“函数指针成员变量”，这是 Strategy 设计模式的一种分解表现形式。
* 以 tr1::function 成员变量替换 virtual 函数，因而允许使用任何可调用物搭配一个兼容于需求的签名式。这也是 Strategy 设计模式的某种形式。
* 将继承体系内的 virtual 函数替换为另一个继承体系内的 virtual 函数。这是 Strategy 设计模式的传统实现手法。

2）将机能从成员函数移到 class 外部函数，带来的一个缺点是，非成员函数无法访问 class 的 non-public 成员。

3）tr1::function 对象的行为就像一般函数指针。这样的对象可接纳“与给定之目标签名式兼容”的所有可调用物。

#### 条款 36：绝不重新定义继承而来的 non-virtual 函数

1）任何情况下都不该重新定义一个继承而来的 non-virtual 函数。

`class B {` 

`public:` 

​	`void mf();` 

​	`…` 

`};` 

`class D: public B {`

`public:` 

​	`void mf();` 

`};` 

`D x;` 

`B* pB = &x;` 

`pB->mf(); // 调用 B::mf` 

`D* pD = &x;` 

`pD->mf(); // 调用 D::mf` 

non-virtual 函数是静态绑定，由于 pB 被声明为一个 pointer-to-B，即使指向一个类型为“B 派生之 class”的对象，也调用的是 B 所定义的版本。

virtual 函数是动态绑定。如果 mf 是个 virtual 函数，不论通过 pB 或 pD 调用 mf，都会导致调用 D::mf，因为 pB 和 pD 真正指的都是一个类型为 D 的对象。

#### 条款 37：绝不重新定义继承而来的缺省参数值

1）静态类型是在程序中被声明时所采用的类型。动态类型是指“目前所指对象的类型”。

`class Shape {` 

`public:` 

​	`enum ShapeColor { Red, Green, Blue };` 

​	`virtual void draw(ShapeColor color = Red) const = 0;` 

​	`…` 

`};` 

`class Rectangle: public Shape {` 

`public:` 

​	`virtual void draw(ShapeColor color = Blue) const = 0;` 

​	`…` 

`};` 

`Shape* ps; // 静态类型为 Shape*，无动态类型` 

`Shape* pr = new Rectangle; // 静态类型为 Shape*，动态类型为 Rectangle*` 

2）virtual 函数是动态绑定，而缺省参数值是静态绑定。意思是可能会在“调用一个定义于 derived class 内的 virtual 函数”的同时，却使用 base class 为它所指定的缺省参数值。

`pr->draw(); // 调用 Rectangle::draw(Shape::Red) 而非 Rectangle::draw(Shape::Blue)` 

以上事实不止局限于指针，把指针换成 references 问题依旧存在。因此绝对不要重新定义一个继承而来的缺省参数值。

3）可以进行条款 35 中的替代设计，其中之一是 NVI 手法：令 base class 内的一个 public non-virtual 函数调用 private virtual 函数，后者可被 derived classes 重新定义。

 `class Shape {` 

`public:` 

​	`enum ShapeColor { Red, Green, Blue };` 

​	`void draw(ShapeColor color = Red) const` 

​	`{` 

​			`doDraw(color);` 

​	`}`  

​	`…` 

`private:` 

​	`virtual void doDraw(ShapeColor color) const = 0;` 

`};` 

`class Rectangle: public Shape {` 

`public:` 

​	`…` 

`private:` 

​	`virtual void doDraw(ShapeColor color) const;` 

​	`…` 

`};` 

#### 条款 38：通过复合塑模出 has-a 或“根据某物实现出”

1）复用是类型之间的一种关系，当某种类型的对象内含它种类型的对象，便是这种关系。复合的意义和 public 继承完全不同，public 继承带有一种 is-a（是一种）的意义，复合意味着 has-a（有一个）或 is-implemented-in-terms-of（根据某物实现出）。

2）is-a 和 is-implemented-in-terms-of 不同。假设需要一个 template，希望制造出一组 classes 用来表现不重复对象组成的 sets。set 可以由 list 实现，若为 is-a 的继承：
`template<typename T>` 

`class Set: public std::list<T> { … };` 

由于 list 可含重复元素而 set 不行，此方法错误。

正确的做法为：

`template<class T>` 

`class Set {` 

`public:` 

​	`bool member(const T& item) const;` 

​	`void insert(const T& item);` 

​	`void remove(const T& item);` 

​	`std::size_t size() const;` 

`private:` 

​	`std::list<T> rep;` 

`};` 

`template<typename T>` 

`bool Set<T>::member(const T& item) const` 

`{` 

​	`return std::find(rep.begin(), rep.end(), item) != rep.end();` 

`}` 

`template<typename T>` 

`void Set<T>::insert(const T& item)` 

`{` 

​	`if (!member(item)) rep.push_back(item);` 

`}` 

`template<typename T>` 

`void Set<T>::remove(const T& item)` 

`{` 

​	`typename std::list<T>::iterator it =`

​		`std::find(rep.begin(), rep.end(), item);` 

​	`if (it != rep.end()) rep.erase(it);` 

`}` 

`template<typename T>` 

`std::size_t Set<T>::size() const` 

`{` 

​	`return rep.size();` 

`}` 

这些函数如此简单，因此都适合成为 inlining 候选人。

#### 条款 39：明智而审慎地使用 private 继承

1）private 继承规则：

* 如果 classes 之间的继承关系是 private，编译器不会自动将一个 derived class 对象转换为一个 base class 对象。这和 public 继承的情况不同。
* 由 private base class 继承而来的所有成员，在 derived class 中都会变成 private 属性，纵使它们在 base class 中原本是 protected 或 public 属性。

private 继承意味 implemented-in-terms-of。如果让 class D 以 private 形式继承 class B，用意是为了采用 class B 内已经备妥的某些特性，不是因为 B 对象和 D 对象存在有任何观念上的关系。private 继承意味只有实现部分被继承，接口部分应略去。private 继承在软件“设计”层面上没有意义，其意义只及于软件实现层面。

2）private 继承通常比复合的级别低，但是当 derived class 需要访问 protected base class 的成员，或需要重新定义继承而来的 virtual 函数时，这么设计是合理的。

`class Timer {` 

`public:` 

​	`explicit Timer(int tickFrequency);` 

​	`virtual void onTick() const;` 

​	`…` 

`};` 

Widget 类要重新定义 onTick 函数，若 Widget 类用 public 继承，由 is-a 关系知并不合理。此时必须以 private 形式继承。

`class Widget: private Timer {` 

`private:` 

​	`virtual void onTick() const;` 

​	`…` 

`};` 

也可以用复合取而代之，只要在 Widget 内声明一个嵌套式 private class，后者以 public 形式继承 Timer 并重新定义 onTick，然后放一个这种类型的对象于 Widget 内。即模拟“阻止 derived classes 重新定义 virtual 函数”。

`class Widget {` 

`private:` 

​	`class WidgetTimer: public Timer {` 

​	`public:` 

​			`virtual void onTick() const;` 

​			`…` 

​	`};` 

​	`WidgetTimer timer;` 

​	`…` 

`};` 

3）和复合不同，private 继承可以造成 empty base 最优化（EBO）。这对致力于“对象尺寸最小化”的程序库开发者而言，可能很重要。EBO 一般只在单一继承（而非多重继承）下才可行。

`class Empty { }; // 其对象应该不使用任何内存` 

`class HoldsAnInt {` 

`private:` 

​	`int x;` 

​	`Empty e;` 

`};` 

不使用继承时，sizeof(HoldsAnInt) > sizeof(int)。在大多数编译器中 sizeof(Empty) 获得 1，因为面对“大小为 0 之独立对象”，通常 C++ 官方勒令默默安插一个 char 到空对象内。这个约束不适用于 derived class 对象内的 base class 成分，因为它们并非独立。

`class HoldsAnInt: private Empty {` 

`private:` 

​	`int x;` 

`};` 

此时有 EBO，即 sizeof(HoldsAnInt) == sizeof(int)。

#### 条款 40：明智而审慎地使用多重继承

1）多重继承比单一继承复杂。它可能导致新的歧义性，以及对 virtual 继承的需要。典型的“钻石型多重继承”，必须令 base class 成为一个 virtual base class。即令所有直接继承自它的 classes 采用“virtual 继承”。

`class File { … };` 

`class InputFile: virtual public File { … };` 

`class OutputFile: virtual public File { … };` 

`class IOFile: public InputFile,` 

​								`public OutputFile` 

`{ … };` 

2）virtual 继承会增加大小、速度、初始化（及赋值）复杂度等成本。如果 virtual classes 不带任何数据，将是最具实用价值的情况。因此，非必要不使用 virtual bases，如果必须使用，尽可能避免在其中放置数据。

3）多重继承的确有正当用途。其中一个情节涉及“public 继承某个 Interface class”和“private 继承某个协助实现的 class”的两相组合。即由 public 负责继承接口，private 负责继承实现。

### 7. 模板与泛型编程

#### 条款 41：了解隐式接口和编译器多态

1）classes 和 templates 都支持接口和多态。

2）对 classes 而言接口是显式的，以函数签名式（函数名称、参数类型、返回类型）为中心。多态是通过 virtual 函数发生于运行期，运行期多态主要是指在程序运行的时候，动态绑定所调用的函数，动态地找到了调用函数的入口地址。

`class Widget {` 

`public:` 

​	`Widget();` 

​	`virtual ~Widget();`

​	`virtual std::size_t size() const;` 

​	`void swap(Widget& other);` 

​	`…` 

`};` 

`void doProcessing(Widget& w)` 

`{` 

​	`if (w.size() > 10 && w != some) {` 

​			`Widget temp(w);` 

​			`temp.swap(w);` 

​	`}` 

`}` 

对于上面的 doProcessing：

* w 必须支持 Widget 接口，称为显式接口，即在源码中明确可见。
* 由于某些成员函数是 virtual，w 对那些函数的调用将表现出运行期多态，也就是将于运行期根据 w 的动态类型。

3）对 template 参数而言，接口是隐式的，奠基于有效表达式。多态则是通过 template 具现化和函数重载解析发生于编译期。

`template<typename T>` 

`void doProcessing(T& w)` 

`{` 

​	`if (w.size() > 10 && w != some) {` 

​			`T temp(w);` 

​			`temp.swap(w);` 

​	`}` 

`}` 

对于模板函数：

* w 必须支持哪种接口，由 template 中执行于 w 身上的操作来决定。size、swap、copy 构造等一组表达式就是 T 必须支持的一组隐式接口。
* 以不同的 template 参数具现化 function templates 会导致调用不同的函数。

#### 条款 42：了解 typename 的双重意义

1）使用 template 参数时，前缀关键字 class 和 typename 可互换。

2）template 内出现的名称如果相依于某个 template 参数，称之为从属名称，不相依的称为非从属名称。如果从属名称在 class 内呈嵌套状，称它为嵌套从属名称。在 template 中，要用 typename 标识嵌套从属类型名称。

`template<typename T>` 

`void print2(const T& container)` 

`{` 

​	`if (container.size() >= 2) {`

​			`typename T::const_iterator` 

​			` iter(container.begin());` 

​			`…` 

​	`}` 	

`}` 

T::const_iterator 可能是个类型，也有可能是个 static 成员变量而恰被命名为 const_iteraotr。C++ 有个规则可以解析此一歧义状态：如果解析器在 template 中遭遇一个嵌套从属名称，它便假设这个名称不是个类型。在 T::const_iterator 前放置关键字 typename，即告诉 C++ 这是个类型。

3）typename 不可以吃现在 base classes list 内的嵌套从属类型名称之前，也不可在 member initialization list（成员初始列）中作为 base class 修饰符。

`template<typename T>` 

`class Derived: public Base<T>::Nested { // base class list` 

`public:` 

​	`explicit Derived(int x)` 

​	`: Base::Nested(x) // 成员初始列` 

​	`{` 

​			`typename Base<T>::Nested temp;` 

​			`…` 

​	`}` 

`};` 

#### 条款 43：学习处理模板化基类内的名称

1）C++ 有“不进入 templatized base classes 观察”的行为。

`template<typename Company>` 

`class MsgSender {` 

`public:` 

​	`void sendClear(const MsgInfo& info)` 

​	`{ … }` 

​	`…` 

`};` 

`template<>` 

`class MsgSender<CompanyZ> { // 无 sendClear 函数` 

`public:` 

​	`…` 

`};` 

template<> 象征这是个特化版的 template，在实参是 CompanyZ 时使用。这是所谓的模板全特化，一旦类型参数被定义，再没有其他 template 参数可供变化。

这时候若有一个 derived class 继承了 MsgSender 并调用函数。

`template<typename Company>` 

`class LogMsgSender: public MsgSender<Company> {` 

`public:` 

​	`…` 

​	`sendClear(info);` 

`};` 

这段代码不合法，因为 class 并未提供 sendClear 函数。C++ 知道 base class templates 有可能被特化，而那个特化版本可能不提供和一般性 template 相同的接口。因此它拒绝在模板化基类内寻找继承而来的名称（如 SendClear）。

解决方法有三种：

* 第一是在 base class 函数调用动作之前加上 this->。

  `template<typename Company>` 

  `class LogMsgSender: public MsgSender<Company> {` 

  `public:` 

  ​	`…` 

  ​	`this->sendClear(info);` 

  `};` 
  
* 第二是使用 using 声明式。

  `template<typename Company>` 

  `class LogMsgSender: public MsgSender<Company> {` 

  `public:` 

  ​	`using MsgSender<Company>::sendClear;` 

  ​	`…` 

  ​	`sendClear(info);` 

  `};` 

* 第三是明白指出被调用的函数位于 base class 内。这个方法如果调用的是 virtual 函数会失效。

  `template<typename Company>` 

  `class LogMsgSender: public MsgSender<Company> {` 

  `public:` 

  ​	`…` 

  ​	`MsgSender<Company>::sendClear(info);` 

  `};` 

#### 条款 44：将与参数无关的代码抽离 templates

1）Templates 生成多个 classes 和多个函数，所以任何 template 代码都不该与某个造成膨胀的 template 参数产生相依关系。

2）因非类型模板参数而造成的代码膨胀，往往可以消除，做法是以函数参数或 class 成员变量替换 template 参数。

3）因类型参数而造成的代码膨胀，往往可降低，做法是让带有完全相同二进制表述的具现类型共享实现码。

#### 条款 45：运用成员函数模板接受所有兼容类型

1）使用 member function templates（成员函数模板）生成“可接受所有兼容类型”的函数。

`template<typename T>` 

`class SmartPtr {` 

`public:` 

​	`template<typename U>` 

​	`SmartPtr(const SmartPtr<U>& other)` 

​		`: heldPtr(other.get()) { … }` 

​	`T* get() const { return heldPtr; }` 

​	`…` 

`private:` 

​	`T* heldPtr;` 

`};` 

对任何类型 T 和任何类型 U，可以根据 SmartPtr<U> 生成一个 SmartPtr<T>。同时在该“构造模板”实现代码中约束转换行为——这个行为只有当“存在某个隐式转换可将一个 U* 指针转为一个 T* 指针”时才能通过编译。

2）如果声明 member templates 用于“泛化 copy 构造”或“泛化 assignment 操作”，还是需要声明正常的 copy 构造函数和 copy assignment 操作符，否则并不会阻止编译器生成它们自己的 copy 构造函数。

#### 条款 46：需要类型转换时请为模板定义非成员函数

1）编写一个 class template，而它所提供的“与此 template 相关的”函数支持“所有参数之隐式类型转换”时，将那些函数定义为“class template 内部的 friend 函数”。

`template<typename T>` 

`class Rational {` 

`public:` 

​	`Rational(const T& num = 0, const T& den = 1);` 

​	`const T num() const;` 

​	`const T den() const;` 

​	`…`  

`};` 

`template<typename T>` 

`const Rational<T> operator* (const Rational<T>& lhs,` 

​															`const Rational<T>& rhs)` 

`{ … }` 

`Rational<int> oenHalf(1, 2); // 条款 24` 

`Rational<int> result = oneHalf * 2; // 无法通过编译` 

这里的例子编译器不知道想要调用的是哪个函数，它们试图想出什么函数被名为 operator* 的 template 具现化出来。template 实参推导过程中从不将隐式转换函数纳入考虑，因此其无法将 int 推算为 Rational<int>。

将 operator* 声明为一个 friend，作为函数而非函数模板，编译器可在调用时使用隐式转换函数，也就可以解决这里的混合式调用。

`template<typename T>` 

`class Rational {` 

`public:` 

`…` 

`friend const Rational operator*(const Rational& lhs,` 

​																	`const Rational& rhs)` 

`{` 

​	`return Rational(lhs.num() * rhs.num(),` 

​										`lhs.den() * rhs.den());` 

`}` 

`};` 

虽然使用 friend，却与 friend 的传统用途“访问 class 的 non-public 成分”毫不相干。为了让类型转换可能发生于所有实参身上，需要一个 non-member 函数；为了令这个函数被自动具现化，需要将它声明在 class 内部；而在 class 内部声明 non-member 函数的唯一办法就是令它成为一个 friend。

#### 条款 47：请使用 traits classes 表现类型信息

1）STL 迭代器分类：

* input 迭代器，只能向前移动，一次一步，客户只可读取（不能涂写）它们所指的东西，而且只能读取一次。istream_iterators 是这一分类的代表。
* output 迭代器，只向前移动，一次一步，客户只可涂写它们所指的东西，而且只能涂写一次。ostream_iterators 是这一分类的代表。
* forward 迭代器，可以做前述两种分类能做的每一件事，而且可以读或写其所指物一次以上。
* bidirectional 迭代器，除了可以向前移动，还可以向后移动。STL 的 list，set，multiset，map，multimap 属于这一类。
* random access 迭代器，比上一个分类威力更大的地方在于可以执行“迭代器算术”，也就是它可以在敞亮时间内向前或向后跳跃任意距离。

对于这五种结构，C++ 标准程序库分别提供专属的卷标结构加以确认：

`struct input_iterator_tag {};` 

`struct output_iterator_tag {};` 

`struct forward_iterator_tag: public input_iterator_tag {};` 

`struct bidirectional_iterator_tag: public forward_iterator_tag {};` 

`struct random_access_iterator_tag: public bidirectional_iterator_tag {};` 

2）Traits 并不是 C++ 关键字或一个预先定义好的构件；它们是一种技术，这个技术的要求之一是，它对内置类型和用户自定义类型的表现必须一样好。iterator_traits 的运作方式是，针对每一个类型 IterT，在 struct iterator_traits<IterT> 内一定声明某个 typedef 名为 iterator_category。其以两个部分实现上述所言，首先它要求每一个“用户自定义的迭代器类型”必须嵌套一个 typedef，名为 iterator_category，用来确认适当的卷标结构。

`template<typename IterT>` 

`struct iterator_traits {` 

​	`typedef typename IterT::iterator_category iterator_category;` 

​	`…` 

`};` 

这对用户自定义类型行得通，但对指针行不通，因为指针不可能嵌套 typedef。第二部分专门用来对付指针。

`template<typename IterT>` 

`struct iterator_traits<IterT*>` 

`{` 

​	`typedef random_access_iterator_tag iterator_category;` 

​	`…` 

`};` 

即 traits 以 templates 和“templates 特化”完成实现。

3）整合重载技术后，traits classes 有可能在编译期对类型执行 if…else 测试。

#### 条款 48：认识 template 元编程

1）Template metaprogramming（TMP，模板元编程）可将工作由运行期移往编译期，因而得以实现早期错误侦测和更高的执行效率。

2）TMP 可被用来生成“基于政策选择组合”的客户定制代码，也可用来避免生成对某些特殊类型并不适合的代码。

3）TMP 并没有真正的循环构件，所以循环效果系藉由递归完成。TMP 的递归甚至不是正常种类，因为 TMP 循环并不涉及递归函数调用，而是涉及“递归模板具现化”。如在编译期计算阶乘：

`template<unsigned n>` 

`struct Factional {` 

​	`enum { value = n * Factional<n - 1>::value };` 

`};` 

`template<>` 

`struct Factional<0> { // 特殊情况值为1` 

​	`enum { value = 1 };` 

`};` 

### 8. 定制 new 和 delete

#### 条款 49：了解 new-handler 的行为

1）set_new_handler 允许客户指定一个函数，在内存分配无法获得满足时被调用。set_new_handler 是声明于 <new> 的一个标准程序库函数。

`namespace std {` 

​	`typedef void (*new_handler) ( );` 

​	`new_handler set_new_handler(new_handler p) throw();` 

`}` 

new_handler 定义出一个指针指向函数，该函数没有参数也不返回任何东西。set_new_handler 获得一个 new_handler 并返回一个 new_handler 的函数。

set_new_handler 参数是个指针，指向 operator new 无法分配足够内存时该被调用的函数。其返回值也是个指针，指向 set_new_handler 被调用前正在执行的那个 new_handler 函数。

`// 当 operator new 无法分配足够内存时，该被调用的函数` 

`void outOfMem()` 

`{` 

​	`std::cerr << "Unable\n";` 

​	`std::abort();` 

`}` 

`int main()` 

`{` 

​	`std::set_new_handler(outOfMem);` 

​	`int* p = new int[1000000L];` 

`}` 

如上，若无法为 1000000 个整数分配足够空间，outOfMem 会被调用。

2）一个设计良好的 new-handler 函数必须做以下事情：

* 让更多内存可被使用。这便造成 operator new 内的下一次内存分配动作可能成功。实现此策略的一个办法是，程序一开始执行就分配一大块内存，而后当 new-handler 第一次被调用，将它们释还给程序使用。
* 安装另一个 new-handler。这样目前的 new-handler 就可以安装另一个来替换自己。做法之一是令 new-handler 修改“会影响 new-handler 行为”的 static 数据、namespace 数据或 global 数据。
* 卸除 new-handler，也就是将 null 指针传给 set_new_handler。
* 抛出 bad_alloc 的异常。这样的异常不会被 operator new 捕捉，因此会被传播到内存索取处。
* 不返回，通常调用 abort 或 exit。

3）nothrow new 只能保证 operator new 不抛掷异常。它只适用于内存分配，后继的构造函数调用还是可能抛出异常。

#### 条款 50：了解 new 和 delete 的合理替换时机

1）替换编译器提供的 operator new 和 operator delete 的理由：

* 用来检测运用上的错误。
* 为了强化效能。定制版的 new 和 delete 可能需要的内存较少、更快。
* 为了收集使用上的统计数据。如分配区块的大小分布、寿命分布、分配和归还的次序、最大动态分配量等。
* 为了增加分配和归还的速度。泛用型分配器往往比定制型分配器慢。
* 为了降低缺省内存管理器带来的空间额外开销。
* 为了弥补缺省分配器中的非最佳齐位。齐位指计算机体系结构中要求特定的类型必须放在特定的内存地址上，如要求指针的地址必须是 4 倍数或 doubles 的地址必须是 8 倍数。
* 为了将相关对象成簇集中。如果知道特定的某个数据结构往往被一起使用，又希望在处理这些数据时将“内存页错误”的频率降至最低，那么为此数据结构创建另一个 heap 就有意义。
* 为了获得非传统的行为。

#### 条款 51：编写 new 和 delete 时需固守常规

1）operator new 应该内含一个无穷循环，并在其中尝试分配内存，如果它无法满足内存需求，就该调用 new-handler。它也应该有能力处理 0 bytes 申请，即即使要求 0 bytes，operator new 也得返回一个合法指针。

2）operator new 函数会被 derived classes 继承，此时最好将“内存申请量错误”的调用行为改为标准 operator new。

`class Base {` 

`public:` 

​	`static void* operator new(std::size_t size)`  						`throw(std::bad_alloc);` 

`};` 

`class Derived: public Base // 假设 Derived 未声明 operator new` 

`{ … };` 

`Derived* p = new Derived; // 调用 Base::operator new` 

`void* Base::operator new(std::size_t size)` 

​											`throw(std::bad_alloc)` 

`{` 

​	`if (size != sizeof(Base))` 

​			`return ::operator new(size); // 标准 operator new 处理` 

​			`… // 否则在这里处理` 

`}` 

3）operator delete 应该在收到 null 指针时不做任何事。即要保证“删除 null 指针永远安全”。

4）operator new 和 operator delete 的 class 专属版本都应该能处理“比正确大小更大的（错误）申请”。

#### 条款 52：写了 placement new 也要写 placement delete

1）如果 operator new 接受的参数除了一定会有的那个 size_t 之外还有其他，这便是个所谓的 placement new。类似的，operator delete 如果接受额外参数，便称为 placement deletes。当写出一个 placement operator new，也要写出对应的 placement operator delete。如果没这样做，程序可能会发生隐微而时断时续的内存泄漏。

`Widget* pw = new (std::cerr) Widget;` 

如上，调用 operator new 并传递 cerr 为其 ostream 实参。若内存分配成功，而 Widget 构造函数抛出异常，没有 operator delete 时就会发生内存泄漏。因此需要同时有 placement new 和 delete。

`class Widget {` 

`public:` 

​	`static void* operator new(std::size_t size,`  				
​    `std::ostream& log) throw(std::bad_alloc);`

​	`static void operator delete(void* pMemory) throw();`

​	`static void operator delete(void* pMemory,` 
​	`std::ostream& log) throw();` 

​	`…` 

`};` 

2）当声明了 placement new 和 placement delete，不要无意识地遮掩了它们的正常版本。如 base 中的 operator new 会遮掩 global 版本、derived class 中会遮掩 global 版本和继承版本。

3）缺省情况下 global 作用域内提供三种形式的 operator new。若非有意阻止，需要确保在所生成的任何定制型 operator new 之外还可用，对于每一个可用的 operator new 也提供相应的 operator delete。完成方法可以使建立一个 base class，内含所有正常形式的 new 和 delete。

`class StandarNewDeleteForms {` 

`public:` 

​	`// normal new/delete` 

​	`static void* operator new(std::size_t size) throw(std::bad_alloc)` 				

​	`{ return ::operator new(size); }` 

​	`static void operator delete(void* pMemory) throw();` 

​	`{ ::operator delete(pMemory); }` 

​	`// placement new/delete` 

​	`static void* operator new(std::size_t size. void* ptr) throw();` 

​	`{ return ::operator new(size, ptr); }` 

​	`static void operator delete(void* pMemory, void* ptr) throw();` 

​	`{ return ::operator delete(pMemory, ptr); }` 

​	`// nothrow new/delete` 

​	`static void* operator new(std::size_t size, const std::nothrow_t& nt) throw();` 

​	`{ return ::operator new(size, nt); }` 

​	`static void operator delete(void *pMemory, const std::nothrow_t&) throw()` 

​	`{ ::operator delete(pMemory); }` 

`};` 

凡是想以自定形式扩充标准形式的客户，可利用继承机制及 using 声明式。

`class Widget: public StandardNewDeleteForms {` 

`public:` 

​	`using StandardNewDeleteForms::operator new;` 

​	`using StandardNewDeleteForms::operator delete;` 

​	`…` 

`};` 

### 9. 杂项讨论

#### 条款 53：不要轻忽编译器的警告

1）严肃对待编译器发出的警告信息。努力在编译器的最高警告级别下争取“无任何警告的荣誉”。

2）不要过度倚赖编译器的报警能力，因为不同的编译器对待事情的态度并不相同。一旦移植到另一个编译器上，原本依赖的警告信息有可能消失。

#### 条款 54：让自己熟悉包括 TR1 在内的标准程序库

1）C++ 标准程序库的主要机能由 STL、iostream、locals 组成。并包含 C99 标准程序库。

2）TR1 添加了 14 个组件的支持，都放在 std 命名空间内。

* 智能指针 tr1::shared_ptr 和 tr1::weak_ptr。
* tr1::function，此物得以表示任何可调用物。
* tr1::bind，能够做 STL 绑定器。和前任绑定器不同的是，它可以和 const 及 non-const 成员函数协同运作，可以和 by-reference 参数协同运作，而且不需要特殊协助就可以处理函数指针。
* Hash tables，用来实现 sets，multisets，maps，multimaps。
* 正则表达式。
* Tuples。
* tr1::array。
* tr1::mem_fn，这个是语句构造上与成员函数指针一致的东西。
* tr1::reference_wrapper，可以造成容器“犹如持有 references”。
* 随机数生成工具。
* 数学特殊函数，包括 Laguerre 多项式、Bessel 函数、完全椭圆积分以及更多数学函数。
* C99 兼容扩充。
* Type traits。
* tr1::result_of，这是个 template，用来推导函数调用的返回类型。

3）TR1 自身只是一份规范。为获得 TR1 提供的好处，需要一份实物，一个好的实物来源是 Boost。

#### 条款 55：让自己熟悉 Boost

1）Boost 程序库对付的主题非常繁多，区分数十个类目，包括：

* 字符串与文本处理，覆盖具备类型安全的 printf-like 格式化动作、正则表达式，以及语汇单元切割和解析。
* 容器，覆盖 array、bitsets 及多维数组。
* 函数对象和高级编程，覆盖若干被用来作为 TR1 机能基础的程序库，如 Lambda。
* 泛型编程，覆盖一大组 traits classes。
* 模板元编程。
* 数学和数值。
* 正确性与测试，覆盖用来将隐式模板接口形式化的程序库，以及针对“测试优先”编程形态而设计的测试。
* 数据结构，覆盖类型安全的 unions 以及 tuple 程序库。
* 语言间的支持，包括允许 C++ 和 Python 之间的无缝互操作性。
* 内存，覆盖 Pool 程序库，用来做出高效率而区块大小固定的分配器，以及多变化的智能指针。
* 杂项，包括 CRC 检验、日期和时间的处理等。

2）Boost 网址：http://boost.org