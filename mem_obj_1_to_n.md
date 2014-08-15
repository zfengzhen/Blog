# 内存对象池一对多关系处理   

游戏服务器经常会涉及到各种对象, 这些对象一般都由对象内存池分配, 为了使得重启数据仍然存在, 内存对象池在服务器中采用共享内存实现.  

经常会碰到一个问题, 一个Player对象, 可能拥有好多其他对象. 比如一个玩家可能有多个建筑物对象, 有多个物品对象. 在程序中如何去实现一个Player对象拥有多个其他的对象, 且使得程序在重启后能够直接恢复这种拥有关系.  
![image](https://github.com/zfengzhen/Blog/blob/master/img/one_obj_has_many_other_obj.jpg)   

### 两个问题:  
### **问题1**   
如何使得一个Player对象拥有多个其他对象, 比如Building对象, Item对象?   
### **问题2**
这进程coredump后, 重启后加载, 仍然能够恢复这层拥有关系.

### 假设:  
ShmPool<Player>为Player对象的对象内存池, 每次都是分配一个大于0的内存池索引, 用来标识该Player对象.  
同样, ShmPool<Building>和ShmPool<Item>分别为Building对象和Item对象的对象内存池, 且都拥有一个字段标识是属于哪个Player对象, 这里采用Player对象的内存池索引.  

### 方法1:  
Player对象中不去描述这种拥有关系, 而采用一个类似于全局map的动态内存方式, 比如std::map<player_index, std::list<building_index>>这种方式, key为Player对象的内存池索引, 而value为Player对象拥有的所有Building对象. 这样可以解决**问题1**.  
因为map是动态内存, 进程重启后, 数据消失, 这样只能在重启的时候, 遍历Building对象内存池, 将这层关系恢复出来, **问题2**也得到解决.  

### 方法2:  
Player对象中分配一个所拥有对象的数组, 比如m_building_list[MAX_BUILDING_NUM+1]用来存储建筑物对象的内存池索引, 0为未使用, 其他为存储的建筑物对象内存池索引. 通过m_building_list我们可以查找到Player对象所拥有的所有建筑物对象, 因为通过建筑物对象的内存索引可以到建筑物对象的内存池中获取到具体数据. **问题1**解决.  
因为拥有对象关系存在Player对象中, Player对象本身保存在共享内存池中, 所以**问题2**也解决, 而且不需要额外操作.  

在项目中, 采用了第二种做法. 

### 遇到的后续问题:   
###1  
Player对象拥有许多其他的对象, 每个对象都需要有以下的方法.

```c      
get_free_inst_id() // 在m_building_list中获取空位
add_inst()  // 添加一个新的对象
get_inst()  // 根据inst_id获取对象指针
del_inst()  // 根据inst_id删除对象
```	
	
对于每个对象都得写一堆几乎一样的方法, 目前想着的一个办法是: 上述方法只各写一个, 但是传入参数都加一个枚举值, 比如get_free_inst_id(E_BUILDING)是获取建筑物对象的空闲的实例ID, get_inst(E_BUILDING, inst_id)根据inst_id的值, 获取建筑物对象.  

###2  
对于每一个对象的管理都有一个类去完成. 比如Player对象有PlayerMgrModule去管理, Building有BuildingMgrModule去管理, Item有ItemMgrModule去管理. 实际上这些管理类中的方法也几乎相同. **我们可以把所有的管理类合并, 统一由一个ObjMgrModule去管理**.
   


