# tolua++实现分析   
2014-08-10 11:50:04

## 前言
cocos2dx-lua以及quick都大规模采用tolua++进行绑定,  对于大规模使用tolua++信心不足, 最近花了几天时间把tolua++源码读了一遍, 了解了tolua++绑定的实现, 明白了tolua++对内存的管理, 增强对这种大规模使用tolua++代码的把控制, 以及未来服务器也同步采用lua5.1和tolua++的可能.   

## tolua++使用流程   
1 tolua++工具通过tag或者pkg方式生成要绑定函数的目标文件, 把类成员函数翻译成lua能够注册的C函数   
参见官方文档: http://www.codenix.com/~tolua/   
2 得到tolua++自动生成的绑定文件, 该代码提供了从lua层访问C/C++层的绑定   
3 与项目一起编译tolua++自动生成的绑定文件, 在项目文件中调用tolua的API进行使用  

##tolua++注册类   
1 在项目文件中tolua++自动生成的绑定文件中的tolua_xxx_open(L)函数注册类绑定(其中xxx为pkg文件名)   
2 tolua_xxx_open(L)函数调用了tolua_open(L), tolua_reg_types, tolua_cclass, tolua_beginmodule, tolua_function, tolua_variable, tolua_endmodule等函数进行类的绑定, 其中tolua_reg_types是自动绑定文件生成的函数, 其他都是tolua++库函数   
3 tolua_open(L) 建立相应的全局注册表, 包括tolua_opened, tolua_peers, tolua_ubox,    tolua_super, tolua_gc   
4 tolua_reg_types主要调用tolua_usertype注册该pkg文件下的所有C++类型的metatabl  
5 tolua_usertype调用tolua_newmetatable 注册C++类型metatable和const C++类型metatable   
6 tolua_usertype调用tolua_classevents注册C++类型metatable中的__add, __call, __div, __eq, __gc, __index, __le, __lt, __mul, __newindex, __sub方法   
7 tolua_cclass设置父子关系, 以及对象回收函数(为tolua++自动生成的一个调用delete的C函数)    
8  tolua_cclass调用mapinheritance设置父子关系metatable, 指定父类为子类的metatable, 通过指定metatable的方式进行继承, 其中const_type—继承—>type—继承—>base_type    
9 tolua_cclass调用set_ubox设置父子关系共享同一tolua_ubox, tolua_ubox暂时为空表, 且值为弱引用的弱表    
10 tolua_cclass调用mapsuper设置全局注册表中tolua_super表的父子关系    
11 绑定类成员函数: tolua_function   
12 绑定类成员变量: tolua_variable, 成员变量的绑定通过建立.set 和.get两张表, 通过__index绑定tolua++生成的get方法, __newindex绑定tolua++绑定的set方法    
13 如果有namespace的话, 通过tolua_beginmodule和tolua_endmodule进行定义    
14 绑定new, 通过绑定函数创建的对象, lua不会自动调用析构, 得手动调用delete进行析构   
15 绑定new_local,  通过绑定的函数创建对象, 该函数中比new的绑定函数多了一个lua_register_gc调用,  在全局注册表中的tolua_gc表中, 以C++指针为键, 类型的metatable为值, 通过class_gc_event进行自动释放   
16 绑定delete,  如果有析构函数的一定要注册析构函数,   否则释放时会调用默认的释放函数,  tolua_default_collect, 调用free进行释放, 会发生很多未定义的行为    
![image](https://github.com/zfengzhen/Blog/blob/master/img/tolua_class_relation.png)    

## tolua++中全局注册表解析   
1 tolua_opened 通过该字段的存在与否标识释放已经调用过tolua_open   
2 tolua_peers 用来存储C++对象在lua中的扩展, 键为弱引用的弱表. lua5.1没用到这张表, 直接通过对C++对象指针进行lua_setfenv环境变量设置和获取该C++对象的tolua_peers表  
3 tolua_ubox  用来存储以C++对象指针为键, 值为lua建立的fulluserdata的键值对. 看tolua++源码中有两套ubox, 一套在全局注册表, 一套在C++类型的metatable中, 实际在看代码的过程中, 只用其中一套就OK, 先判断在C++类型的metatable中是否有ubox, 没有再建立全局注册表的ubox, 而在注册C++类型的metatable中就会在该metatable中建立ubox. 优先使用C++类型的metatable的ubox. ubox为值为弱引用的弱表.   
4 tolua_super 用来标识C++父类和子类关系, 该表中以C++类型的metatable的为键, 值为一个表格. 这个表格中以父类类型名为键,  0和1为值, 用来记录对象的父子关系    
5 tolua_gc 用来标识是否由lua来进行垃圾回收的表. 以C++对象指针为键, 值为传入C++类型的metatable, 传入C++类型只能为C++对象的同类或者父类, 如果传入的是父类类型, gc的时候只会调用父类的析构.   
![image](https://github.com/zfengzhen/Blog/blob/master/img/tolua_register_table.png)   

## tolua++在lua中的工具函数   
1 tolua.cast(var, type)   
修改var的元表使它成为type类型, type是一个完整的C++类型对象(包括命名空间等)字符串   
2 tolua.getpeer()   
获取peer表, C++对象在lua中的扩展   
3 tolua.setpeer()   
设置peer表, C++对象在lua中的扩展   
4 tolua.inherit(table, var)   
tolua++会将table作为对象和var具有一样的类型,  必须自己去实现table的方法.   
5 tolua.type(var)    
返回一个代表对象类型的字符串    
6 tolua.takeownership(var)     
接管对象引用变量的所有权, 意味着当对象的所有引用都没有的话, 对象本身将会被lua删除    
7 tolua.releaseownership(var)    
释放对象引用变量的所有权    

## tolua++的一些细节   
1 在lua中对C++对象引用的实质, 比如ClassA* pa = new ClassA();tolua_pushusertype(L, pa, "ClassA");lua_setglobal(L, “classa_fromc”); classa_fromc是C++对象lightuserdata的引用嘛?   
tolua_pushusertype通过压入C++指针pa, 搜寻到ClassA的metatable, 从该metatable中找寻到tolua_ubox表, 以pa的C++ lightuserdata为键, 看是否有元素存在, 如果存在则表示该指针已经压入过lua中, 如果不存在, 则建立C++ lightuserdata为键, fulluserdata为值, fulluserdata存储了该C++指针, 且设置ClassA的metatable为fulluserdata的metatable, 栈上留下的是ubox[u], 也就是说classa_fromc实质是存储C++对象指针的一个lua中的fulluserdata.    

2 tolua_ubox表为值为弱引用的弱表    
因为lightuserdata在lua中只表示一个C++内存的地址, 所有的lightuserdata都共享一张metatable(http://lua-users.org/wiki/LightUserData), 不能直接通过C++传入的内存地址进行绑定具体的C++类型的metatable, 所以引入了tolua_ubox表. 假如tolua_ubox表不是值为弱引用的弱表, tolua_ubox[u] = newu, 不管怎么样newu在tolua_ubox的引用计数都至少为1, 所以gc不会释放, 引起不必要的内存消耗. 设置为值为弱引用的弱表, gc的时候newu不占用一个引用计数, gc后该表的键值对消失.   

3 C++类如果有析构函数, 则一定要注册进tolua++   
C++一定要注册析构函数, 否则释放时会调用默认的释放函数 tolua_default_collect, 调用free进行释放, 会发生很多未定义的行为    

4 如果设置了tolua_peers表, 调用的时候要小心   
在lua脚本中通过tolua.setpeer设置tolua_peers表时, 如果表里面有跟C++类中相同的函数名, 则在类的使用中tolua_peers表会优先使用    

