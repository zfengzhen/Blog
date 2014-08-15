# 《深入探索C++对象模型》读书笔记

## 第1章 关于对象
- 1、C++在布局以及存取时间上的主要的额外负担是由**virtual**引起的，包括：
	- a、virtual function机制，引入**vptr**以及**vtbl**，支持一个有效率的"执行期绑定"
	- b、virtual base class，用以实现"多次出现在继承体系中的base class，有一个单一而被共享的实例"
	- c、多重继承下，派生类跟第二个以及后续基类之间的转换
- 2、"指针的类型"会教导编译器如何解释某个特定地址中的内存内容以及其大小（void*指针只能够持有一个地址，而不能通过它操作所指向的object）
- 3、**C++通过class的pointers和references来支持多态**，付出的代价就是额外的间接性。它们之所以支持多态是因为它们并不引发内存中任何"与类型有关的内存委托操作(type-dependent commitment)"，会受到改变的，只有他们所指向的内存的"大小和内容的解释方式"而已。

## 第2章 构造函数语意学
![](https://github.com/zfengzhen/Blog/blob/master/img/%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.png)
![](https://github.com/zfengzhen/Blog/blob/master/img/%E5%A4%8D%E5%88%B6%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0.png)

##第3章 Data语意学
- 1、类对象大小受三个因素影响
	- a、virtual base和virtual function带来的vptr影响
	- b、**EBO**（Empty Base class Optimize）空基类优化处理，**EBC**（Empty Base Class）占用一个字节，其他含有数据成员的从EBC派生的派生类，只会算自己数据成员的大小，不受EBC一字节的影响
	- c、alignment 字节对齐
- 2、Nonstatic data members在class object中的排列顺序将和其被声明顺序一样，任何中间介入的static data members都不会被放进布局之中。
- 3、静态成员变量 static data members
	- a、存放在程序的data segment之中
	- b、通过指针和对象来存取member，完全一样，不管继承或者是虚拟继承得来，全局也只存在唯一一个实例
	- c、静态常量成员可以在类定义时直接初始化，而**普通静态常量成员只能在.o编译单元的全局范围内初始化**
- 4、非静态成员变量 nonstatic data members
	- a、每一个nonstatic data member的偏移量在编译时即可获知，不管其有多么复杂的派生，都是一样的。通过对象存取一个nonstatic data member，其效率和存取一个C struct member是一样的。
	- b、从对象存取obj.x和指针存取pt->x有和差异？
		- 当继承链中有虚基类时，查找虚基类的成员变量时延迟到了执行期，根据**virtual class offset**查找到虚基类的部分，效率稍低(成员变量的数据存取并没有this指针的变化）
- 5、成员变量具体分布 （代码引用陈皓 C++对象的内存布局 http://blog.csdn.net/haoel/article/details/3081328 http://blog.csdn.net/haoel/article/details/3081385 分析环境：linux 2.6 gcc）
	- ![单一继承的一般继承](https://github.com/zfengzhen/Blog/blob/master/img/%E5%8D%95%E4%B8%80%E7%BB%A7%E6%89%BF%E7%9A%84%E4%B8%80%E8%88%AC%E7%BB%A7%E6%89%BF.png)
	- ![多重继承](https://github.com/zfengzhen/Blog/blob/master/img/%E5%A4%9A%E9%87%8D%E7%BB%A7%E6%89%BF.png)
	- ![重复继承](https://github.com/zfengzhen/Blog/blob/master/img/%E9%87%8D%E5%A4%8D%E7%BB%A7%E6%89%BF.png)
	- ![钻石型多重继承](https://github.com/zfengzhen/Blog/blob/master/img/%E9%92%BB%E7%9F%B3%E5%9E%8B%E5%A4%9A%E9%87%8D%E7%BB%A7%E6%89%BF.png)

##第4章 Function语意学
- 1、C++的设计准则之一：nostatic member function 至少必须和一般的nonmember function有相同的效率。
	- a、改写函数原型，在参数中增加this指针
	- b、对每一个"nonstatic data member的存取操作"改为由this指针来存取
	- c、将member function重写为一个外部函数，经过"mangling"处理（不许要处理的加上 extern "C"）
- 2、覆盖（override）、重写（overload）、隐藏（hide）的区别
	- **重载**是指**不同的函数使用相同的函数名，但是函数的参数个数或类型不同**。调用的时候根据函数的参数来区别不同的函数。
	- **覆盖**（也叫重写）是指**在派生类中重新对基类中的虚函数（注意是虚函数）重新实现**。即函数名和参数都一样，只是函数的实现体不一样。
	- **隐藏**是指**派生类中的函数把基类中相同名字的函数屏蔽掉了**。隐藏与另外两个概念表面上看来很像，很难区分，其实他们的关键区别就是在多态的实现上。
- 3、静态成员函数 static member functions
	- a、不能访问非静态成员
	- b、不能声明为const、volatile或virtual
  	- c、参数没有this
	- d、可以不用对象访问，直接 类名::静态成员函数 访问
- 4、**C++多态（polymorphism）**表示"以一个public base class的指针（或者reference），寻址出一个derived class object"
- 5、vtable虚函数表一定是在编译期间获知的，其函数的个数、位置和地址是固定不变的，完全由编译器掌控，执行期间不允许任何修改。
- 6、vtable的内容：
	- a、virtual class offset（有虚基类才有）
	- b、topoffset
	- c、typeinfo
	- d、继承基类所声明的虚函数实例，或者是覆盖（override）基类的虚函数
	- e、新的虚函数（或者是纯虚函数占位）
- 7、执行期虚函数调用步骤
	- a、通过vptr找到vtbl
 	- b、通过thunk技术以及topoffset调整this指针（因为成员函数里面可能调用了成员变量）
	- c、通过virtual class offset找到虚基类共享部分的成员
	- d、执行vtbl中对应slot的函数
- 8、多重继承中，一个派生类会有n-1个额外的vtbl（也可能有n或者n以上个vtbl，看是否有虚基类），它与第一父类共享vtbl，会修改其他父类的vtbl
- 9、函数性能
Inline Member > (Nonmember Friend, Static Member, Nonstatic Member) > Virtual Member > Virtual Member(多重继承) > Virtual Member(虚拟继承)
- 10、Inline对编译器只是请求，并非命令。inline中的局部变量+有表达式参数-->大量临时变量-->程序规模暴涨

##第5章 构造、析构、拷贝语意学
- 1、**析构函数不能定义为纯虚函数**，因为每个子类的析构函数都会被编译器扩展调用基类的析构函数以及再上层的基类的析构函数。因为只要缺乏任何一个基类的析构函数的定义，就会导致链接失败。（实际上纯虚函数是可以被定义以及静态调用-类名::函数，但是C++标准为纯虚函数就是代表没有定义）
- 2、继承体系下带有数据成员的构造顺序
	- a、虚基类构造函数，从左到右，从顶层到底层。它和非虚基类不同：是由最底层子类调用的。
	- b、非虚基类构造函数，按照基类被声明的顺序从左到右调用。它与虚基类不同：是由直接之类调用的。
	- c、如果类中有虚表指针，则设置vptr初值，若有新增虚函数或者是覆盖基类虚函数，修改vtbl内容（实际上vtble在编译时，类定义时就已经建立好了）
	- d、成员变量以其声明顺序进行初始化构造
	- e、构造函数中，用户自定义代码最后被执行
- 3、如果类没有定义析构函数，那么只有在类中含有成员对象或者是基类中含有析构函数的情况下，编译器才会自动合成一个出来
- 4、析构函数的顺序跟构造函数的顺序完全相反，如果是为了多态的析构函数，设置为虚析构函数
- 5、**不要在构造函数和析构函数中调用虚函数**


##第6章 执行期语意学
- 1、**尽量推迟变量定义，避免不必要的构造和析构**（虽然C++编译器会尽量保证在调用变量的时候才进行构造，推迟变量定义会使得代码好阅读）
- 2、全局类变量在编译期间被放置于data段中并被置为0
	- GOOGLE C++编程规范：禁止使用class类型的静态或全局变量，只允许使用POD型静态变量(Plain Old Data)和原生数据类型。因为它们会导致很难发现的bug和不确定的构造和析构函数调用顺序
	- 解决：改成在static函数中，产生局部static对象
- 3、如果有表达式产生了临时对象，那么应该对完整表达式求值结束之后才摧毁这些创建的临时对象。有两个例外：1）该临时对象被refer为另外一个对象的引用；2）该对象作为另一对象的一部分被使用，而另一对象还没有被释放。

##第7章 站在对象模型的尖端
- 1、对于RTTI的支持，在vtbl中增加一个**type_info**的slot
- 2、dynamic_cast比static_cast要花费更多的性能（检查RTTI释放匹配、指针offset等），但是安全性更好。
- 3、对引用施加dynamic_cast：1）成功；或2）抛出bad_cast异常；对指针施加：1）成功；2）返回0指针。
- 4、使用**typeid()**进行判断，合法之后再进行dynamic_cast，这样就能够避免对引用操作导致的bad_cast异常： if(typeid(rt) == typeid(rt2)) …。但是如果rt和rt2本身就是合法兼容的话，就会损失了一次typeid的操作性能。