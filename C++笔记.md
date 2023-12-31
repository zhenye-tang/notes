







## 多态

### 1. 虚函数

* 利用 C++ 的虚函数机制来实现
* 子类重写父类中使用 virtual 修饰的成员函数，当使用父类指针指向子类对象时，调用成员函数就是调用的子类的成员函数
* 从汇编上分析，一旦父类中有 virtual，则申请子类对象时，子类对象的前 4 字节存放的虚表的地址，这个虚表中存放的是子类重写父类的成员函数

### 2. 虚析构

* 当使用父类指针指向子类对象时，析构父类指针，只会调用父类的析构函数，不会调用子类的虚构函数
* 对于上述现象，父类应该将自己的析构函数添加 virtual 修饰，使其变为虚析构函数
* 这样子类、父类的析构函数都能够被调用

### 3. 纯虚函数

* 纯虚函数：没有函数体，且初始化为 0 的虚函数，用来定义接口规范（相当于函数声明，实现由子类来实现）
* 抽象类（Abstract Class）
  * 含有纯虚函数的类，不可以实例化（不可以用来创建对象）
  * 抽象类也可以包含非纯虚函数、成员变量
  * 如果父类是抽象类，子类没有完全重写纯虚函数，那么这个子类依然是抽象类

## Static

* 在一个类内，使用 Static 修饰的成员变量或成员函数
* static 修饰的成员变量，必须在类的外面初始化
* 生命周期一直存在，不随类的消亡而消亡
* 其实就是全局静态变量，只不过可以给这个变量添加访问权限（public、private、protected）
* 不用实例化对象就能访问，如 car::value
* 通常在单例类中使用
* 静态成员函数不能修改类中的非静态成员变量

## Const 成员

* const 成员：被 const 修饰的成员变量、非静态成员函数
* congt 成员变量
  * 必须初始化（类内部初始化），可以在声明的时候直接初始化
  * 可在构造的参数列表中初始化
* const 成员函数（非静态）
  * const 关键字写在参数列表后面，函数的声明和实现都必须带 const
  * const 函数内部不能修改类内非 static 成员变量
  * 内部只能调用 const 成员函数、static 成员函数。因为 static 成员函数也不能修改非 static 成员变量
  * const 成员函数 与 非 const 成员函数构成重载
  * 非  const 对象优先调用非 const 成员函数
  * const 对象只能调用 const 成员函数、static 成员函数

## 引用类型成员

* 必须初始化（不考虑 static 情况）
  * 在声明时初始化
  * 参数列表中初始化

## 拷贝构造函数

* 利用已有对象，创建新的对象，新的对象初始化时调用拷贝构造函数

* 如果类中没有拷贝构造函数，默认将已有对象的内存复制到新的对象中

* 类中存在拷贝构造函数，使用已有对象创建新对象时，调用拷贝构造函数，如下：

  ```c++
  class car
  {
  public:
  
  	int m_price;
  	int m_hight;
  
  	car(int price = 0, int hight = 0) :m_price(price), m_hight(hight) {}
  	car(const car& car) :m_price(car.m_price), m_hight(car.m_hight) {}
  
  	void display()
  	{
  		cout << m_price << "  " << m_hight << endl;
  	}
  };
  
  int main()
  {
  	car aodi(300,400);
  	car dazhong(aodi);
  	dazhong.display();
  }
  ```

* 调用父类的拷贝构造函数：构造函数本来就会默认调用父类的无参构造函数，因此调用子类的拷贝构造函数时，如果在参数列表中指定了调用父类的拷贝构造函数，则父类的拷贝构造会被调用，否则，调用父类的无参构造函数

* 浅拷贝
  * 编译器其默认提供的拷贝是浅拷贝
  * 将一个对象中的所有成员变量的值拷贝到另一个对象
  * 如果某个成员变量是个指针，只会拷贝指针中存储的地址值，并不会拷贝指针指向的内存空间
* 深拷贝
  * 实现深拷贝，需要自定义拷贝构造函数
  * 将指针类型的成员变量所指的内存空间，拷贝到新的内存空间
* 使用对象类型作为函数参数或者返回值
  
  * 可能会产生一些不必要的中间对象，编译器默认会调用拷贝构造函数

## 隐式构造、explicit

## 引用

* &变量名
* 当作对象来使用

## 友元

* 友元包括友元函数和友元类
* 如果将函数 A (非成员函数) 声明为类 C 的友元函数，那么在函数 A 内部就能直接访问类 C 对象的所有成员

## 运算符重载

* 全局函数、成员函数都能运算符重载
* 传引用减少生成中间对象（构造）
* const 加在函数前面，说明返回值为常量，不能修改，const 加在函数后面，说明该函数内部不能修改成员变量，相当于限制住该函数是不能修改任何成员变量的，但是能修改类内 static 变量。
* const 类型的对象只能调用 const 类型的成员函数、static 成员函数
* 运算符重载时，重载函数前加 const 说明返回值是常量，不能当作左值，重载函数后加 const ，说明该函数只能由常量对象来调用，且内部不能修改类的成员变量
* 重载函数参数使用引用，减少中间变量产生
* c++ 中尽量传递引用，减少中间变量引起的过多函数调用，且效率高，对于函数返回，可返回一个对象，该对象并不用 malloc 出来，而是开辟在栈上，直接返回，c++ 会在函数调用者的栈中拷贝构造一个对象给函数调用者（c 中返回结构体对象是否是用样的机制？明天看看汇编码）
* 重载 ++a，直接重载即可，a++,需要在重载函数中添加 int

## 模板

* 泛型，是一种将类型参数化以达到代码复用的技术，C++ 中使用模板来实现泛型
* 模板的使用格式
  * templetc <typename\class T>
  * typename 和 class 是等价的
* 模板没有被使用时，是不会被实例化出来的
* 模板的声明和实现如果分离到 .h 和 .cpp 中，导致链接报错
* 一般将模板的声明和实现统一放到一个文件中
* 默认参数只能写在声明里面，不能写在实现里面

## 类型转换

* const_cast，将const 类型转为非 const 类型

  ```c++
  const Point* p1 = new Point();
  Point* p2 = const_cast<Point *>(p1);
  等价于 p2 = (Point *)p1;
  ```

  