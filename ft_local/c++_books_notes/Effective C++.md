# Effective C++

## 1. 自己习惯c++

### 条款1：把C++看成一个语言联邦

- 把C++看成一个语言联邦：C语言，Object-OrientedC++，Template C++， STL

### 条款2：尽量以const，enum，inline替换define

- 对于单纯常量，最好以const对象或enums替换#define

- 对于形似函数的宏，至于改用inline函数替换#defines

  ```c++
  int f(int a) {
      return 0;
  }
  #define CALL_WITH_MAX(a, b) f((a)>(b) ? (a) : (b))
  int a = 5, b = 0;
  CALL_WITH_MAX(++a, b); //a被累加两次
  CALL_WITH_MAX(++a, b + 10); //a被累加一次，因为a比b小
  ```

  ```
  inline void callWithMax(const T& a, const T& b){
      f(a > b ? a : b);
  }
  ```

### 条款3：尽可能使用const

- 右左法则：如果const出现在星号左边，表示被指物是常量，如果出现在星号右边，表示指针本身是常量，如果出现在星号两边，表示被指物和指针都是常量

```
std::vector<int> vec;
const std::vector<int>::iterator iter = vec.begin(); //iter的作用就是T* const
*iter = 10; //没问题，改变iter所指物
++iter; // 错误！iter是const

std::vector<int>::const_iterator cIter = vec.begin();
*cIter = 10; //错误！cIter指向const
++cIter;
```

- 在一个函数声明式中，const可以和函数返回值，各参数，函数自身（如果是成员函数）产生关联

```c++
const Rational operator* (const Rational& lhs, const Rational& rhs);//.这种函数声明可避免以下的错误
Rational a, b, c;
(a*b) = c;
```

- const成员函数：为了确认该成员函数可以作用于const对象身上
  1. 使class接口比较容易被理解，可以得知哪个函数可以改动对象内容哪个函数不行
  2. 使操作const对象成为可能
  3. 已定义成const的成员函数，一旦企图修改数据成员的值，则编译器按错误处理
  4. 加了const的成员函数可以被非const对象和const对象调用
  5. 不加const的成员函数，只能被非const对象调用

- tips:一个更改了“指针隶属物”的成员函数虽然不能算是const，但如果只有指针隶属于对象，那么此函数仍然可以编译通过

  ```c++
  class CTextBlock {
  public:
      CTextBlock(char* s) {
          pText = s;
      }
      char& operator[] (size_t position) const {
          return pText[position];
      }
  private:
      char* pText;
  };
  int main() {
      const CTextBlock cctb("hello");
      char* pc = &cctb[0];
      *pc = 'J';
      return 0;
  }
  ```

- 因此引出了：const成员函数也可以修改成员变量，利用mutable

  ```c++
  class CTextBlock {
  public:
      CTextBlock(char* s) {
          pText = s;
      }
      char& operator[] (size_t position) const {
          return pText[position];
      }
      std::size_t CTextBlock::length() const {
          if (!lengthIsValid) {
              textLength = std::strlen(pText);
              lengthIsValid = true;
          }
          return textLength;
      }
  private:
      char* pText;
      mutable std::size_t textLength;
      mutable bool lengthIsValid;
  };
  ```

- 将某些东西声明为const可帮助编译器侦测出错误用法。const可被施加于任何作用域内的对象，函数参数，函数返回类型，成员函数本体
- 编译器强制实时bitwise constness，但你编写程序时应该使用“概念上的常量性”
- 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复

### 条款4：确定对象被使用前已先被初始化

- 为内置型对象进行手工初始化，因为C++不保证初始化它们。
- 构造函数最好使用成员初始列，而不要在构造函数本体内使用赋值操作。初始列列出的成员变量，起排列次序应该和它们在class中的声明次序相同。
- 为免除“跨编译单元之初始化次序”问题，请以local static对象替换non-local static对象。



## 2. 构造，析构，赋值

### 条款5：了解C++默默编写并调用哪些函数

- 编译器可以暗自为class创建defaut构造函数、copy构造函数、copy assignment操作符，以及析构函数
- copy assignment在成员变量涉及引用，const时，编译器会拒绝生成默认的

### 条款06：若不想使用编译器自动生成的函数，就该明确拒绝

将函数声明在private中

```c++
class HomeForSale {
public:
private:
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(const HomeForSale&);
}
```

设计一个uncopyable类，并继承他

```c++
class Uncopyable{
protected:
    Uncopyable();
    ~Uncopyable();
private:
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
};
class HomeForSale: private Uncopyable {
    
}
```

- 为驳回编译器自动提供的机能，可将相应的成员函数声明为private并且不予实现。使用像Uncopyable这样的base class也是一种做法。

### 条款07：为多态基类声明virtual析构函数

- 对于base class，应该将其析构函数声明为virtual，避免有一个基类指针指向子类对象进行销毁时，出现子类无法被销毁的情况，导致内存泄漏（当derived class对象由一个base class指针被删除，而该base class带着一个non-virtual析构函数，其结果未有定义——实际执行时通常发生的是对象的derived成分没被销毁）

  ```c++
  class TimeKeeper {
  public:
  	TimeKeeper();
      virtual ~TimeKeeper();
  }
  TimeKeeper* ptk = getTimeKeeper();
  ...
  delete ptk; //行为正确
  ```

- 任何class只要带有virtual函数，都几乎可以确定应该带有一个virtual析构函数。
- 当class不被作为base class时，最好不要将其析构函数声明为virtual(否则会引入虚表指针，增加对象大小)
- 带多态性质的base class应该声明一个virtual析构函数。如果class带有任何virtual函数，就应该拥有一个virtual析构函数
- class的设计目的如果不是作为base class使用，或不是为了多态，就不该声明virtual析构函数

### 条款08：别让异常逃离析构函数

- 析构函数绝对不要吐出异常，如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕捉任何异常，然后吞下他们（不传播）或结束程序

  ```c++
  case1
  //close()有可能失败，抛出异常
  DBConn::~DBConn() {
      try {db.close();}
      catch(...){
          制作运转记录，记下对close的调用失败
          std::abort();
      }
  }
  case2
  DBConn::~DBConn() {
      try {db.close();}
      catch(...){
          制作运转记录，记下对close的调用失败
      }
  }
  ```

- 如果客户需要对某个操作函数运行期间抛出的异常做出反应，那么class应该提供一个普通函数（而非在析构函数中）执行该操作

  ```c++
  class DBConn {
  public:
      void close() {
          db.close();
          closed = true;
      }
      ~DBConn() {
          if (!closed) {
              try {
                  db.close();//关闭连接（如果客户不这么做）
              }catch (...){
                  制作运转记录，记下对close的调用失败
                  ...
              }
          }
      }
  private:
      DBConnection db;
      bool closed;
  };
  ```

### 条款09：绝不在构造和析构过程中调用virtual函数

- 在构造和析构期间不要调用virtual函数，因为这类调用从不下降至derived class（比起当前执行构造函数和析构函数的那层）、

  ```c++
  class Transaction {
  public:
      Transaction();
      virtual void logTransaction() const = 0;
  };
  Transaction::Transaction() {
      logTransaction();
  }
  class BuyTransaction : public Transaction {
  public:
      virtual void logTransaction() const;
      
  };
  class SellTransaction : public Transaction {
  private:virtual void logTransaction() const;
  };
  BuyTransaction b;//会先调用Transaction()，在此时只会调用base的logTransaction，因为子类还没构建出来，会先构建父类
  ```

### 条款10：令operator=返回一个reference to *this

- 令赋值操作符返回一个reference to *this

```c++
class Widget{
public:
    
};
Widget& operator=(const Widget& rhs){
    return *this;
}//同样适用于+=
```

### 条款11：在operator=中处理“自我赋值”

“自我赋值”发生在对象被赋值给自己时

```c++
class Widget{
public:
    
};
Widget w;
w = w;//赋值给自己
//一些潜在的自我赋值
a[i]=a[j];//i=j
*px=*py;
```

自我赋值的危险：如果*this和rhs是同一个对象，那么delete就不止销毁的是当前对象的bitmap，也销毁了rhs的bitmap，导致自己持有的指针指向了一个已经被删除的对象

```;
class Bitmap { ... };
class Widget{
...
private:
	Bitmap* pb;
}
Widget& Widget::operator=(const Widget& rhs){
	delete pb; //停止使用当前的bitmap
	pb = new Bitmap(*rhs.pb); //使用rhs's bitmap的副本
	return *this;
}
```

规避方式：

1. 在复制pb所指东西之前别删除pb

```
Widget& Widget::operator=(const Widget& rhs){
	if (this == &rhs) return *this; //证同测试，可不要
	Bitmap* pOrig = pb;
	pb = new Bitmap(*rhs.pb); //使用rhs's bitmap的副本
	delete pOrig; //停止使用当前的bitmap
	return *this;
}
```

2. copy and swap

```
void swap(Widget& rhs); //交换*this和rhs的数据
Widget& Widget::operator=(const Widget& rhs){
	Widget temp(rhs);
	swap(temp);
	return *this;
}
```

- 确保当对象自我赋值时operator=有良好行为，其中技术包括比较“来源对象”和“目标对象”的地址、精心周到的语句顺序、以及copy-and-swap
- 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确

### 条款12：复制对象时勿忘其每一个成分

- Copying函数应该确保复制“对象内的所有成员变量”及“所有base class成分”
- 不要尝试以某个copying函数实现另一个copying函数，应该将共同技能放进第三个函数中，并由两个copying函数共同调用。

## 3. 资源管理

### 条款13：以对象管理资源

- 为防止资源泄露，请使用RAII对象，他们在构造函数中获得资源并在析构函数中释放资源。这样当跳出某个函数时，对象会被自动析构，避免内存泄漏

  ```c++
  void f() {
      Investment* pInv = createInvestment(); //调用factory函数
      ... // 这里可能提前return，导致内存泄漏
      delete pInv; //释放pInv所指对象
  }
  ```

  

- 两个常被使用的RAII classes分别是tr1::shared_ptr和auto_ptr。前者通常是较佳选择，因为其copy行为比较直观（可以同时指向同一对象）。若选择auto_ptr,复制动作会使它（被复制物）指向null。

### 条款14：资源管理类中小心coping行为

- 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。
  - 禁止复制：如管理互斥器的类
  - 对底层资源使用”引用计数法“：内部使用shared_ptr(可以注册shared_ptr的删除器)
  - 复制底部资源：深拷贝
  - 转移底部资源的拥有权：如auto_ptr

### 条款15：在资源管理类中提供对原始资源的访问

- APIs往往要求访问原始资源，所以对每一个RAII class应该提供一个”取得其所管理之资源“的办法

  ```c++
  std::tr1::shared_ptr<Investmnt> pInv(createInvestment());
  int daysHeld(const Investment* pi);//这里需要传入原始资源
  int days = daysHeld(pInv);//错误
  int days = daysHeld(pInvg.et());//正确
  ```

- 对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对客户比较方便

### 条款16：成对使用new和delete时要采取相同形式

- 如果你在new表达式中使用[]，必须在相应的delete表达式中也使用[]。如果你在new表达式中不使用[]，一定不要在相应的delete表达式中使用[]。

### 条款17：以独立语句将newed对象置入智能指针

考虑以下这种情况，可能出现内存泄露，在执行processWidge前，编译器必须做三件事

- 调用priority
- 执行new widget
- 调用tr1::shared_ptr构造函数

```c++
int priority();
void processWidge(std::tr1::shared_ptr<Widget> pw, int priority);
processWidge(std::tr1::shared_ptr<Widget>(new Widget), priority());
```

但在c++中，对priority的调用可以在1 2 3 位次，如果出现在第2位，并且发生了异常，就会导致new widget返回的指针为空，导致没有被管理

1. 执行new widget
2. 调用priority
3. 调用tr1::shared_ptr构造函数

避免这种情况的办法就是以独立语句将newed对象放于智能指针中

```c++
std::tr1::shared_ptr<Widget> pw(new Widget);
processWidge(pw, priority());
```

- 以独立语句将newed对象放于智能指针中,如果不这样做，一旦异常被抛出，有可能导致难以察觉的资源泄漏

## 4. 设计与声明

### 条款18：让接口容易被正确使用，不易被误用

- 好的接口很容易被正确使用，不容易被误用，应该在所有接口中努力达成这些性质
- ”促进正确使用“的办法包括接口的一致性，以及与内置类型的行为兼容
- ”阻止误用“的办法包括建立新类型、限制类型上的操作，束缚对象值，以及消除客户的资源管理责任
- tr1::shared_ptr支持定制型删除器，这可防范DLL问题，可被用来自动解除互斥锁等等（参考条款14）

### 条款19：设计class犹如设计type

1. 新type的对象应该如何被创建和销毁
2. 对象的初始化和对象的赋值应该有什么样的差别
3. 新type的对象如果被passed by value（以值传递），意味着什么？
4. 什么是新type的”合法值“
5. 新type需要配合某个继承图系吗
6. 新type需要什么样的转换
7. 什么样的操作符和函数对此新type而言是合理的
8. 什么样的标准函数应该驳回
9. 什么是新type的”未声明接口“
10. 你的新type有多么一般化？
11. 真的需要一个新type吗

- class的设计就是types的设计，在定义一个新type之前，请确定已经考虑了本条款覆盖的所有讨论主题

### 条款20：宁以pass-by-reference-to-const替换pass-by-value

pass-by-value的几个问题：

1. 有额外的拷贝构造函数的调用以及析构函数的调用（包含基类的拷贝构造与析构，以及成员变量）
2. 传入子类对象，函数定义形参为基类，会导致子类对象被切割，子类特性消失，转为基类对象

- 尽量以pass-by-reference-to-const替换pass-by-value。前者通常比较高校，并可避免切割问题
- 以上规则并不适用于内置类型，以及stl的迭代器和函数对象。对它们而言，pass-by-value往往比较适合

### 条款21：必须返回对象时，别妄想返回其reference

- 绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或reference指向一个local static对象而有可能同时需要多个这样的对象。条款4已经为在单线程环境中合理返回reference指向一个loca static对象提供了一份设计实例

### 条款22：将成员变量声明为private

- 切记将成员变量声明为private。这可赋予客户访问数据的一致性、可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性
- protected并不比public更具封装性

### 条款23：宁以non-member、non-friend替换member函数

- 宁可拿non-member non-friend函数替换member函数。这样做可以增加封装性、包裹弹性和技能扩充性

### 条款24：若所有参数皆需类型转换，请为此采用non-member函数

- 如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是non-member（且构造函数是非explict的）

```c++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
    const Rational operator* (const Rational& rhs) const;
};
int main() {
    Rational oneEigth(1, 8);
    Rational oneHalf(1, 2);
    Rational result = oneHalf * oneEigth;
    result = result * oneEigth;

    result  = oneHalf * 2; // 2在参数列，被隐式转换
    result = 2 * oneHalf; //错误
    return 0;

}
```

```c++
const Rational operator* (const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.numerator() * rhs.numerator()) / (lhs.denominator() *rhs.denominator());
} //non-member
result = 2 * oneHalf; //正确
```

### 条款25：考虑写出一个不抛异常的swap函数

## 5. 实现

### 条款26：尽可能延后变量定义式的出现时间

有些变量定义过早，如果程序中抛出异常，会导致变量未被使用，浪费一次构造和析构。

- 尽可能延后变量定义式的出现，这样做可增加程序的清晰度并改善程序效率

### 条款27：尽量少做转型动作

1. const_cast通常被用来将对象的常量性去除，唯一有此能力的转型符
2. dynamic_cast用来执行安全向下转型
3. reinterpret_cast意图执行低级转型
4. static_cast用来强迫隐式转换，例如将non-const对象转为const对象，或将int转为double等。

- 如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_casts.如果有个设计需要转型动作，试着发展无需转型的替代方案
- 如果转型是必要的，试着将它隐藏于某个函数背后，客户随后可以调用该函数，而不需将转型放进他们的代码内
- 宁可使用新式转型，不要使用旧式转型。

### 条款28：避免返回handles指向对象内部成分

- 避免返回handles(包括references、指针、迭代器)指向对象内部，遵守这个条例可增加封装性，帮助const成员函数的行为像个const，并将”虚吊号码牌“的可能性降至最低

1. 例1：const函数，返回内部成员的引用，那么const函数无意义，用户仍可修改内部成员（即使为private）
2. 例2：一个class，成员函数返回一个内部成员变量的handles，此时一个函数传值返回一个class对象，然后通过这个临时对象调用内部成员函数（用一个指针接收），该语句执行完后，临时对象被析构，指针指向一个非法内存

### 条款29：为”异常安全“而努力是值得的

```c++
class PrettyMenu{
public:
    void changeBackGround(std::istream& imgSrc);
private:
    Mutex mutex; //互斥器
    Image* bgImage; //目前的背景图像
    int imageChanges; //背景图像被改变的次数
};
void PrettyMenu::changeBackGround(std::istream &imgSrc) {
    lock(&mutex);
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
    unlock(&mutex);
}
```

以上代码没做到以下两点：

1. **不泄露任何资源**：`new Image(imgSrc)` 出现异常，锁无法释放
2. **不允许数据败坏**：`new Image(imgSrc)` 出现异常，`imageChanges`已经累加，且`bgImage`指向一个被删除对象

解决**不泄露任何资源**的方法

```c++
void PrettyMenu::changeBackGround(std::istream &imgSrc) {
    Lock ml(&mutex);//RAII
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

异常安全函数保证以下三个保证之一：

1. **基本承诺**：如果异常被抛出,程序内的任何事物仍然保持在有效状态下。没有任何对象或数据结构会因此而败坏,所有对象都处于一种内部前后一致的状态(例如所有的 class约東条件都继续获得满足)。然而程序的现实状态( exact state)恐怕不可预料。举个例子,我们可以撰写 changebackground使得一且有异常被抛出时, Prettymenu对象可以继续拥有原背景图像,或是令它拥有某个缺省背景图像,但客户无法预期哪一种情况。如果想知道,他们恐怕必须调用某个成员函数以得知当时的背景图像是什么。
2. **强烈保证**:如果异常被抛出,程序状态不改变。调用这样的函数需有这样的认知:如果函数成功,就是完全成功,如果函数失败,程序到“调用函数之前”的状态
3. **不抛掷( nothrow)保证**：承诺绝不抛出异常,因为它们总是能够完成它们原先承诺的功能。作用于内置类型(例如ints,指针等等)身上的所有操作都提供nothrow保证。这是异常安全码中一个必不可少的关键基础材料

**以下方法可提供异常安全保障**

```c++
class PrettyMenu{
public:
    void changeBackGround(std::istream& imgSrc);
private:
    Mutex mutex; //互斥器
    std::tr1::shared_ptr<Image> bgImage; //目前的背景图像
    int imageChanges; //背景图像被改变的次数
};
void PrettyMenu::changeBackGround(std::istream &imgSrc) {
    Lock m1(&mutex);
    bgImage = bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
```

- 异常安全函数( Exception- safe functions)即使发生异常也不会泄漏资源或允许任何数据结构败坏。这样的函数区分为三种可能的保证:基本型、强烈型、不抛异常型
- 强烈保证”往往能够以 copy-and-swap实现出来,但“强烈保证”并非对所有函数都可实现或具备现实意义
- 函数提供的“异常安全保证”通常最高只等于其所调用之各个函数的“异常安全保证”中的最弱者。

### 条款30：透彻了解inlining的里里外外

- 将大多数inlining限制在小型、被频繁调用的函数身上。这可使日后的调试过程和二进制升级更容易，也可使潜在的代码膨胀问题最小化，使程序的速度提升机会最大化
- 不要只因为function templates出现在头文件，就将它们声明为inline

### 条款31：将文件间的编译依存关系降至最低

没太看懂

## 6.继承与面向对象设计

### 条款32：确定你的public继承塑模出is-a关系

- “public继承”意味is-a。适用于base classes身上的每一件事情一定也适用于derived classes身上，因为每一个derived class对象也都是一个base class对象

### 条款33：避免遮掩继承而来的名称

- derived class内的名称会遮掩base classes内的名称。在public继承下从来没有人希望如此。(因为作用域)
- 为了让被遮掩的名称再见天日，可使用using声明式或转交函数。

### 条款34：区分接口继承和实现继承

- 成员函数的接口总是会被继承
- 声明一个pure virtual函数的目的是为了让derived classes只继承函数接口
- 声明impure virtuall函数的目的，是让derived classes继承该函数的接口和缺省实现
- 声明non-virtual函数的目的是为了令derived classes继承函数的接口及一份强制性实现

### 条款35：考虑virtual函数以外的其他选择

- 使用non-virtual interface(NVI)手法，那是Template Method设计模式的一种特殊形式。它以public non-virtual成员函数包裹较低访问性（private或protected）的virtual函数
- 将virtual函数替换为“函数指针成员变量”，这是Strategy设计模式的一种分解表现形式
- 以tr1::function成员变量替换virtual函数，因为允许使用任何可调用物搭配一个兼容于需求的签名式
- 将继承体系内的virtual函数替换为另一个继承体系内的virtual函数，这是strategy设计模式的传统实现方式

### 条款36：绝不重新定义继承而来的non-virtual函数

- non-virtual函数都是静态绑定的
- virtual函数是动态绑定的
- 绝对不要重新定义继承而来的non-virtual函数

### 条款37：绝不重新定义继承而来的缺省参数值

- 绝对不要重新定义一个继承而来的缺省参数值，因为缺省参数值是静态绑定，而virtual函数——你唯一应该覆写的东西——却是动态绑定
- 如果想要继承缺省值，可以用NVI手法，定义一个non-virtual函数（带上缺省值，因为non-virtual函数不会被覆写），然后内部调用其他的

### 条款38：通过复合塑模出has-a或“根据某物实现出”

- 复合的意义和public继承完全不同
- 在应用域，复合意味has-a。在实现域，复合意味is-implemented-in-terms-of。

### 条款39：明智而审慎地使用private继承

- 如果classes之间的继承关系是private，编译器不会自动将一个derived class对象转换为一个base class对象
- 由private base class继承而来的所有成员，在derived class中都会变成private属性，纵使它们在base class中原本是protected或public属性
- 复合一般优于private继承，但三种情况除外：1.当一个意欲成为derived class者想要访问一个意欲成为base class者的protected成分 2.为了重新定义一个或多个virtual函数 3.空基类优化（empty base optimization，EBO）

### 条款40：明智而审慎地使用的多继承（后面再看）

- c++解析重载函数调用的规则：在看到是否有个函数可取用之前，C++首先确认这个函数对此调用之言是最佳匹配。找出最佳匹配函数后才检验其可取用性。
- 后面再看

## 7.模板与泛型编程

### 条款41：了解隐式接口和编译期多态

- 运行期多态：哪一个重载函数该被调用
- 编译期多态：哪一个virtual函数该被调用
- classes和templates都支持接口和多态
- 对classes而言接口是显式的，以函数签名为中心。多态则是通过virtual函数发生于运行期
- 对template参数而言，接口是隐式的，奠基于有效表达式。多态则是通过template具现化和函数重载解析发生于编译器。

### 条款42：了解typename的双重意义

- 任何时候当你想要在template中指涉一个嵌套从属类型名称，就必须在紧邻它的前一个位置放上关键字typename
