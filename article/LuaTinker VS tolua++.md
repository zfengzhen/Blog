# LuaTinker VS tolua++  
**作者: fergus (zfengzhen@gmail.com)**    

lua原生提供注册C函数的方式对C中的函数进行调用.   
C++面向对象的项目都需要一套lua OOD的封装.  
如果直接使用注册C函数的方式的话, 对于面向对象的所有方法都得重写成C的形式, 工作量太大,还好目前有许多第三方库对lua的C++面向对象的调用进行封装, 主要有**LuaPlus**, **LuaBind**, **tolua++**, **LuaTinker**.  
最近想在后台使用LuaTinker调用lua进行逻辑处理, 也学习了下quick-cocos2d-x, quick-cocos2d-x使用的时tolua++, 也阅读了下源码, 查看了下两者对调用C++面向对象的实现, 进行了下比较.

---  

## LuaTinker  
采用模板实现的小巧精悍的封装, 不到2000行代码, 基本的功能都有, 使用起来非常方便.  
使用LuaTinker时, 类定义, 继承关系只能在C++中定义好, 并不能在Lua中定义类, 以及继承某个类, 也就是说类原型是在C++中声明好的, 只能传入Lua中, Lua进行调用, 如果想在Lua中创建新的类的话, LuaTinker是支持不了的, 所以在选择LuaTinker时, 要注意自己有没有这些需求.  
其中源代码中有几个坑:  
`1 table的某个构造函数中没有增加table的引用, 在使用table的时候会遇到问题`  
`2 var_base作为基类, 没有添加虚析构函数`  
`3 class_def的时候不能添加静态成员函数, 如果时静态成员函数的时候需奥通过def去定义`  
`4 call函数调用时候不能传入引用`  
`5 class_con 构造函数只能注册一个, 并以最后一个注册的为准 (2014.5.22添加)`   
修改下, 并注意下就好了, 毕竟这么小巧精悍的代码, 以后遇到问题也很快能够找出来.  

### LuaTinker C++类实现  
![](https://github.com/zfengzhen/Blog/blob/master/img/lua_tinker_cpp_impl.jpg)

---

## tolua++
quick_cocos2d-x中使用的是tolua++  
tolua++是通过修改头文件, 并保存为.pkg文件(quick-cocos2d-x是.tolua文件), 通过tolua++工具转换为lua可调用的cpp文件, 把这些cpp文件编译链接到你的app中, 并通过相应的tolua++ API进行初始化, 在lua脚本中就可以调用C++面向对象的东西了.  
tolua++实现比LuaTinker复杂很多, 它*`可以在Lua中进行类定义, 类继承的行为`*.  
  
### tolua++实现 
![](https://github.com/zfengzhen/Blog/blob/master/img/toluapp_impl.jpg)