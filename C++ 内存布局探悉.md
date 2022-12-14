# C++ 类的内存布局探悉

本文将深度探悉C++的类内存布局，借助VS2022开发工具可以让我们看到类的内存布局情况，操作步骤如下：

1. 打开VS2022开发人员命令提示符(Developer Command Prompt for VS 2022)，注意普通命令提示符不行
2. 运行命令：cl /d1reportSingleClassLayoutObj main.cpp（Obj是所要打印的类的名字）

### 1、最简单的类，无继承，无虚函数

```
struct Obj {
	int m_obj1;
	int m_obj2;
};
```

命令输出内容如下：

```
class Obj       size(8): ///总大小=8
        +---
 0      | m_obj1 ///第一个是m_obj1，4个字节大小
 4      | m_obj2 ///第二个是m_obj2,4个字节大小
        +---
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：**类里面的元素按照声明的顺序依次排列（考虑内存对齐，下面不再提示内存对齐问题）**

### 2、无继承，但有虚函数的类

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};
```

命令输出内容如下：

```
class Obj       size(12):///总大小=12，起始处多了一个vfptr指针指向该类的虚函数表Obj::$vftable@
        +---
 0      | {vfptr} ///虚函数表指针，4个字节大小，指向了虚函数表，以供运行时调用虚函数
 4      | m_obj1
 8      | m_obj2
        +---

Obj::$vftable@: ///虚函数表
        | &Obj_meta  ///类的相关信息，用于RTTI,运行时类型识别
        |  0         ///暂时不知为何多一个0？
 0      | &Obj::print   ///Obj::print
 1      | &Obj::{dtor}  ///Obj::~Obj 析构函数

Obj::print this adjustor: 0
Obj::{dtor} this adjustor: 0
Obj::__delDtor this adjustor: 0
Obj::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

总结：

1. 类中一旦有虚函数，会导致类的所有对象的首地址处增加一个指向虚函数表的指针vfptr
2. 虚函数表中首部保存了该类类型信息地址：&Obj_meta，用以支持运行时类型识别
3. 虚函数表中依次保存了该类的所有虚函数地址

### 3、单继承的类

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Animal:Obj {
	int m_anmal1;
	int m_anmal2;
	
	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(20):///总大小=20
        +---
 0      | +--- (base class Obj)
 0      | | {vfptr}  ///指向Animal::$vftable@ 地址
 4      | | m_obj1
 8      | | m_obj2
        | +---
12      | m_anmal1
16      | m_anmal2
        +---

Animal::$vftable@:
        | &Animal_meta ///类的相关信息，用于RTTI,运行时类型识别
        |  0
 0      | &Animal::print
 1      | &Animal::{dtor} //Animal没有重写析构函数，由此可见此时编译器自动生成了析构函数
 2      | &Animal::speak  //新增的虚函数，在末尾处增加

Animal::print this adjustor: 0
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 0
Animal::__delDtor this adjustor: 0
Animal::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：

1. 子类的虚函数表中，子类重写的虚函数会在原位置覆盖父类的虚函数
2. 子类新增的虚函数会增加在虚函数表后面
3. 父类的成员变量在前面，子类的成员变量在父类的后面
4. 父子类共用了起始处虚函数指针，指向了子类的虚函数表。由于该表中，子类的重写的虚函数地址已覆盖了父类的，所以无论在子类、父类中调用该函数，最终都会调用子类的虚函数，以实现所谓“动态联编”

### 4、单个类的虚继承

与上述唯一不同的点在于，把普通继承方式修改为虚继承:

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Animal :virtual Obj { ///虚继承,与上述的唯一不同
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(28)://总共增加了3个指针，2个虚函数表指针，一个虚基类表指针
        +---
 0      | {vfptr}//virtual function pointer，指向Animal::$vftable@Animal@，只包含新增的虚函数
 4      | {vbptr}//virtual base pointer,指向Animal::$vbtable，表中包含所有虚基类的数据的偏移量
 8      | m_anmal1
12      | m_anmal2
        +---
        +--- (virtual base Obj)
16      | {vfptr}//指向Animal::$vftable@Obj，只包含所有Obj的虚函数表（子类覆盖父类），不包含新增的
20      | m_obj1
24      | m_obj2
        +---

Animal::$vftable@Animal@://只包含类型信息、新增的虚函数表
        | &Animal_meta   //类型信息，用以支持RTTI
        |  0             //
 0      | &Animal::speak //新增的虚函数

//由于虚继承导致所有虚基类的成员信息将依次放在对象的尾部，如果子类中访问虚基类的数据成员信息，需要知道偏移量
Animal::$vbtable@:///虚基类数据偏移量表，保存类所有的虚基类的数据成员的偏移量
 0      | -4    ///可能表示vbptr的偏移量？
 1      | 12 (Animald(Animal+4)Obj) ///虚基类Obj的数据成员偏移量

Animal::$vftable@Obj@://只包含所有Obj的虚函数表（子类覆盖父类），不包含新增的
        | -16  //表示虚基类Obj的数据成员在整个对象中的偏移量
 0      | &Animal::print ///Obj有，但是被Animal覆盖了
 1      | &Animal::{dtor}///Obj有，但是被Animal覆盖了

Animal::print this adjustor: 16
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 16
Animal::__delDtor this adjustor: 16
Animal::__vecDelDtor this adjustor: 16
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      16       4       4 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：由此可知，当继承方式修改为virtual虚继承时，对象的内存布局方式会发生较大改变，主要表现为：

1. 所有虚基类的数据成员信息将不会在头部，而是依次放到尾部，其各自偏移量会保存在一个vbtable表中(virtual Base table)
2. 每个虚基类的数据成员前面都会有该类自己的虚函数表指针，表内只保存该类的所有虚函数（当然子类重写的方法会覆盖父类的）。
3. 整个对象起始处依然有一个虚函数表指针，只不过该表内只包含类型信息、新增的虚函数信息（不含父类的）
4. 整个对象起始处的虚函数表指针后面，会增加一个虚基类表指针，表内记录了所有虚基类数据地址偏移量
5. 所有基类的虚函数表，表头不再包含类型信息，只增加了本类的数据信息在整个对象中的偏移量

### 5、普通多继承

所谓普通继承，指的是非虚继承。

##### 1、两个基类都有虚函数

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Animal :Obj ,Live{
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(32)://大小=32，多增加了2个虚函数表指针
        +---
 0      | +--- (base class Obj)
 0      | | {vfptr}///虚函数表指针->Animal::$vftable@Obj，覆盖父类Obj的虚函数表的基础上，增加新增的
 4      | | m_obj1
 8      | | m_obj2
        | +---
12      | +--- (base class Live)
12      | | {vfptr} ///虚函数表指针->Animal::$vftable@Live,覆盖父类Live的虚函数表的基础上，不含新增的
16      | | m_live1
20      | | m_live2
        | +---
24      | m_anmal1
28      | m_anmal2
        +---

Animal::$vftable@Obj@://首个基类的虚函数表，在父类Obj的虚函数表（子类重写会覆盖）基础上，增加新增的虚函数
        | &Animal_meta ///类型信息，支持RTTI
        |  0
 0      | &Obj::ObjTest    //Obj::ObjTest，子类没有重写，所有保留父类的
 1      | &Animal::print   //Obj::print，子类重写了
 2      | &Animal::{dtor}  //Obj::~Obj，析构函数，子类重写了（编译器自动生成的）
 3      | &Animal::speak   //子类新增的虚函数

Animal::$vftable@Live@://Live基类的虚函数表信息，只包含Live自己的虚函数（子类重写会覆盖），不含新增的
        | -12  ///Live类的数据信息地址偏移量
 0      | &thunk: this-=12; goto Animal::print //Live::print，子类重写了，所有调用子类的
 1      | &Live::liveTest                      //Live::LiveTest，子类没有重写，所有保留父类的  
 2      | &thunk: this-=12; goto Animal::{dtor}//Live::~Live，子类重写了，所有调用子类的

Animal::print this adjustor: 0
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 0
Animal::__delDtor this adjustor: 0
Animal::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：

1. 父类数据按照声明顺序依次放在首部位置，最后才是子类的数据成员
2. 子类与第一个声明的类共享虚函数表指针（头部位置的指针），子类重写的虚函数会覆盖所有父类的虚函数表中，但是子类新增的虚函数只增加到第一个声明的类的虚函数表后面
3. 第一个虚函数表和后面的虚函数表格式不同，第一个的表头包含类型信息，而后的虚函数不包含（只包含该类数据地址偏移量）。

##### 2、两个基类：一个没有虚函数，一个有虚函数

通过上述例子可以看到，子类一般会和第一个声明的类共用一个虚函数表指针（也就是头部的指针），但倘若第一个类本身没有虚函数呢？？这时上述的内存布局明显是不合理的！我们看下编译器会怎么办？

还是用上述代码，只不过把第一个类Obj中的虚函数去掉：

```
struct Obj {
	int m_obj1;
	int m_obj2;
	/*virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}*/
};

struct Live {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Animal :Obj ,Live{
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(28):
        +---
 0      | +--- (base class Live)
 0      | | {vfptr}
 4      | | m_live1
 8      | | m_live2
        | +---
12      | +--- (base class Obj)
12      | | m_obj1
16      | | m_obj2
        | +---
20      | m_anmal1
24      | m_anmal2
        +---

Animal::$vftable@:
        | &Animal_meta
        |  0
 0      | &Animal::print
 1      | &Live::liveTest
 2      | &Animal::{dtor}
 3      | &Animal::speak

Animal::print this adjustor: 0
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 0
Animal::__delDtor this adjustor: 0
Animal::__vecDelDtor this adjustor: 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

由此看到，编译器很聪明，他巧妙的把第一个声明的不含虚函数的类放到后面了，这样子类就和第二个类共用了一个虚函数表指针，新增的虚函数也加到表的后面。

### 6、多个虚继承的类

如果类同时虚继承了多个类，会如何呢？按照上面的知识，我们可以猜想，多个虚基类自然也应该放到数据的后面，且应该多增加一个虚基类表的指针，记录下每个虚基类的数据偏移量，这样子类才能正常访问父类的数据成员。我们验证一下：

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live:Obj {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Animal :virtual Obj, virtual Live {
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(48):
        +---
 0      | {vfptr}
 4      | {vbptr}
 8      | m_anmal1
12      | m_anmal2
        +---
        +--- (virtual base Obj)
16      | {vfptr}
20      | m_obj1
24      | m_obj2
        +---
        +--- (virtual base Live)
28      | +--- (base class Obj)
28      | | {vfptr}
32      | | m_obj1
36      | | m_obj2
        | +---
40      | m_live1
44      | m_live2
        +---

Animal::$vftable@Animal@:
        | &Animal_meta
        |  0
 0      | &Animal::speak

Animal::$vbtable@:
 0      | -4
 1      | 12 (Animald(Animal+4)Obj)
 2      | 24 (Animald(Animal+4)Live)

Animal::$vftable@Obj@:
        | -16
 0      | &Obj::ObjTest
 1      | &Animal::print
 2      | &Animal::{dtor}

Animal::$vftable@Live@:
        | -28
 0      | &Obj::ObjTest
 1      | &thunk: this-=12; goto Animal::print
 2      | &thunk: this-=12; goto Animal::{dtor}
 3      | &Live::liveTest

Animal::print this adjustor: 16
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 16
Animal::__delDtor this adjustor: 16
Animal::__vecDelDtor this adjustor: 16
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      16       4       4 0
            Live      28       4       8 0
```

可以看到，这与我们的猜想基本一致，但是问题来了，里面Obj对象的数据好像有两份，我们怎么样才能使得相同的类只有一份数据呢？等等....不是说好了虚继承本质就是为了解决继承同个类多次导致类数据有多份的问题吗？为什么我们用了virtual继承却还是有2份数据呢？

我们可以尝试把上面的代码稍作修改：让Live 也虚继承自Obj

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live:virtual Obj {///修改点，此次也修改为虚继承
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Animal :virtual Obj, virtual Live {
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(48):
        +---
 0      | {vfptr}
 4      | {vbptr}
 8      | m_anmal1
12      | m_anmal2
        +---
16      | (vtordisp for vbase Obj) ///这个字段用来干嘛的呢？？
        +--- (virtual base Obj)
20      | {vfptr}
24      | m_obj1
28      | m_obj2
        +---
        +--- (virtual base Live)
32      | {vfptr}
36      | {vbptr} ///指向自己所有的虚基类数据地址偏移量
40      | m_live1
44      | m_live2
        +---

Animal::$vftable@Animal@:
        | &Animal_meta
        |  0
 0      | &Animal::speak

Animal::$vbtable@Animal@:
 0      | -4
 1      | 16 (Animald(Animal+4)Obj)
 2      | 28 (Animald(Animal+4)Live)

Animal::$vftable@Obj@:
        | -20
 0      | &Obj::ObjTest
 1      | &(vtordisp) Animal::print
 2      | &(vtordisp) Animal::{dtor}

Animal::$vftable@Live@:
        | -32
 0      | &Live::liveTest

Animal::$vbtable@Live@:
 0      | -4
 1      | -16 (Animald(Live+4)Obj)

Animal::print this adjustor: 20
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 20
Animal::__delDtor this adjustor: 20
Animal::__vecDelDtor this adjustor: 20
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      20       4       4 1
            Live      32       4       8 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

如此一来，Obj数据确实只有一份了。我们也大概清楚编译器是如何让多个相同类只保留一份数据了。

### 7、菱形虚继承问题

##### 1、Animal 普通继承自两个父类（非虚继承）

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live :virtual Obj {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Dead :virtual Obj {
	int m_dead1;
	int m_dead2;
	virtual void print() {
	}
	virtual void DeadTest() {
	}
	virtual ~Dead() {
	}
};

struct Animal : Dead, Live {///与 struct Animal :virtual Dead, virtual Live 完全不一样的内存布局
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出：

```
class Animal    size(56):
        +---
 0      | +--- (base class Dead)
 0      | | {vfptr}
 4      | | {vbptr}
 8      | | m_dead1
12      | | m_dead2
        | +---
16      | +--- (base class Live)
16      | | {vfptr}
20      | | {vbptr}
24      | | m_live1
28      | | m_live2
        | +---
32      | m_anmal1
36      | m_anmal2
        +---
40      | (vtordisp for vbase Obj)
        +--- (virtual base Obj)
44      | {vfptr}
48      | m_obj1
52      | m_obj2
        +---

Animal::$vftable@Dead@:
        | &Animal_meta
        |  0
 0      | &Dead::DeadTest
 1      | &Animal::speak

Animal::$vftable@Live@:
        | -16
 0      | &Live::liveTest

Animal::$vbtable@Dead@:
 0      | -4
 1      | 40 (Animald(Dead+4)Obj)

Animal::$vbtable@Live@:
 0      | -4
 1      | 24 (Animald(Live+4)Obj)

Animal::$vftable@Obj@:
        | -44
 0      | &Obj::ObjTest
 1      | &(vtordisp) Animal::print
 2      | &(vtordisp) Animal::{dtor}

Animal::print this adjustor: 44
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 44
Animal::__delDtor this adjustor: 44
Animal::__vecDelDtor this adjustor: 44
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      44       4       4 1
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

结论：

1. Dead与Live类的相同部分被提取出来，放到了整个对象的最后，且被Dead与Live共享。
2. 整个内存布局顺序：Dead->Live->Animal->Obj
3. 声明类时，如果使用virtual，则会将该基类依次放到尾部，否则依次放首部

##### 2、修改Animal继承方式为虚继承

```
struct Obj {
	int m_obj1;
	int m_obj2;
	virtual void ObjTest() {
	}
	virtual void print() {
	}
	virtual ~Obj() {
	}
};

struct Live :virtual Obj {
	int m_live1;
	int m_live2;
	virtual void print() {
	}
	virtual void liveTest() {
	}
	virtual ~Live() {
	}
};

struct Dead :virtual Obj {
	int m_dead1;
	int m_dead2;
	virtual void print() {
	}
	virtual void DeadTest() {
	}
	virtual ~Dead() {
	}
};

struct Animal :virtual Dead,virtual Live {///修改为虚继承
	int m_anmal1;
	int m_anmal2;

	virtual void print() override {///重写了父类的虚函数
	}

	virtual void speak() {///新增的虚函数
	}
};
```

输出如下：

```
class Animal    size(64):
        +---
 0      | {vfptr}
 4      | {vbptr}
 8      | m_anmal1
12      | m_anmal2
        +---
16      | (vtordisp for vbase Obj)
        +--- (virtual base Obj)
20      | {vfptr}
24      | m_obj1
28      | m_obj2
        +---
        +--- (virtual base Dead)
32      | {vfptr}
36      | {vbptr}
40      | m_dead1
44      | m_dead2
        +---
        +--- (virtual base Live)
48      | {vfptr}
52      | {vbptr}
56      | m_live1
60      | m_live2
        +---

Animal::$vftable@:
        | &Animal_meta
        |  0
 0      | &Animal::speak

Animal::$vbtable@Animal@:
 0      | -4
 1      | 16 (Animald(Animal+4)Obj)
 2      | 28 (Animald(Animal+4)Dead)
 3      | 44 (Animald(Animal+4)Live)

Animal::$vftable@Obj@:
        | -20
 0      | &Obj::ObjTest
 1      | &(vtordisp) Animal::print
 2      | &(vtordisp) Animal::{dtor}

Animal::$vftable@Dead@:
        | -32
 0      | &Dead::DeadTest

Animal::$vbtable@Dead@:
 0      | -4
 1      | -16 (Animald(Dead+4)Obj)

Animal::$vftable@Live@:
        | -48
 0      | &Live::liveTest

Animal::$vbtable@Live@:
 0      | -4
 1      | -32 (Animald(Live+4)Obj)

Animal::print this adjustor: 20
Animal::speak this adjustor: 0
Animal::{dtor} this adjustor: 20
Animal::__delDtor this adjustor: 20
Animal::__vecDelDtor this adjustor: 20
vbi:       class  offset o.vbptr  o.vbte fVtorDisp
             Obj      20       4       4 1
            Dead      32       4       8 0
            Live      48       4      12 0
Microsoft (R) Incremental Linker Version 14.30.30709.0
Copyright (C) Microsoft Corporation.  All rights reserved.
```

由于Animal也采用了虚继承，因此编译器必须将Dead、Live两个基类数据都放到了尾部，因而Animal自己的头部的虚函数表指针和虚基类指针就无法与Dead共享，必须再次添加，导致多了2个指针，故而比上面的方式多了8个字节大小。而这样带来的好处是，Dead、Live这两个类在Animal的所有子类中都只会存在一份，而上面的情况则不然。



Note:如果想让类A在内存中只有一个拷贝，则所有继承自A的类都应该声明为virtual，不声明为virtual的必然会增加一个A的拷贝。