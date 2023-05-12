---
title: c++复习
date: 2019-06-09 23:19:51
description: C++作为一门高级语言, 往往大家对其又爱又恨. 但对于真正想学习编程的人来说, 第一门语言我首推C++, 它真的太严格了, 远不及其他语言这么灵活确实大大增加了难道, 但这也对于初学者来说培养编程思维十分重要. 这里是一个对C++的复习总结, 参照C++ prime学习过程而写. 设计基础语法, 类, STL, 编译等等. 其中会又作者本人对难点的深刻思考, 帮助更加深刻的理解与使用. 其中指针, 引用, 类, 继承与多态尤为重要.
tags: 
    c++
categories:
    语言
mathjax:
    true
---

<center/><font size=12> C++复习</font></center>
# 基础部分

## 变量申明与定义

​		申明: 使得名字为程序所知． extern int i;  //申明(前面有extern)

​		定义: 创建与名字关联实体.    int  j;  //申明并定义j

​		重点: 多个文件之间使用同一个变量, 只能在一个文件下对变量定义, 在其他文件下要申明.(extern: 外部的).

## 引用(&)

​		引用是一个别名,，引用类型引用另一种类型。 使用&d的形式定义。 d为申明的变量名。 **引用不可重新绑定到另一个对象上, 同时引用必须初始化**. 这时对引用和对原来对象的操作一样。 引用类型初始必须是一个变量，不能是常量。

```
int a;

int &d = a;
```



## 指针(*)

​		指针本身是一个对象(引用不是)，允许对指针赋值和拷贝， 在生命周期内可以指向不同对象。

​		指针无需定义时赋值。

​		**引用不是对象(别名), 没有实际地址, 不能定义指向引用的指针。**

​		**所以指针类型都要与其指向对象类型严格匹配(不存在默认类型转化)**

​		void* 用来存放任意对象地址(只能存放地址，不能直接操作所指对象，由于不知道所指对象类型)。

​		指向指针的指针(**)

​		指向指针的引用(*&): 指针本身的引用，与对象无关。

​		指针也是迭代器， 可以进行相减以及和整数加减操作。

### 字符串指针

​		字符串指针比较特殊, 不能使用string对字符串指针进行初始化, 但可以使用.c_str()进行。

```c++
string s("uiguy");

const char *a = s.c_str();
```

​		对字符串指针输出会是字符串本身, 响要输出原始地址则要求进行void*的转化。

```C++
char a[] = "jdgdtyvds";

cout<<a<<endl<<(void *)a<<endl;

output:

jdgdtyvds
0x7ffc5ce91d22
```



## const限定符

​		const类型可以使用一个普通对象进行初始化(不要求是常量)。

```
int n=20;

const int w = n;
```

​		由于const要求定义时赋常量值当要求几个文件之间共享同一个const值时对申明和定义都采用extern表示,这样只需用定义一次。

```
// a.cpp

extern int bufsize = 128;

//b.cpp

extern int bufsze;
```

### const引用(对常量的引用)

​		常量的引用必须是常量引用，普通变量可以使用常量引用，不过不能通过常量引用改变变量的值，只能对变量进行更改。常量引用可以直接引用数值常量。

```
const int ci = 1024;

const int &r1 = ci; //对常量的引用

const int &r2 = 42; // 对数值常量的引用.

int c1 = 12;

const int &r3 = c1; //对普通变量的引用.

int &r4 = ci; //错误, ci为常量, 必须使用常量引用.
```

​		普通引用要求引用与原始对象类型完全一致，而const引用不必如此. 

```
double a = 3.14;

int & b = a; //错误, 非法操作

const int &b = a; //正确, 合法.
```

​		这是由于定义一个引用肯定是希望该引用与原始对象完全一致，要求两个中有一个更改另一个也会更改，但是当类型不一致时，正常操作是会定义一个临时变量来保证引用匹配，这将导致引用与原对象不一致，无法做到同步更改，也不符合引用的目的，因此是非法的。而对于常量引用来说就不存在这个问题，常量引用本来就不希望通过引用对原始对象进行更改，但需注意，这里同样是存在临时变量，因此改变原始值时，常量引用不会改变。

即:

```
double a=3.14; const int & b = a;

==

double a=3.14; const int temp = a; const int &b = temp;
```

此时更改a不会影响b的值.

### 指针与const

​		指针(的)常量：是一个常量，并且属于指针，即指针本身是常量，`int * const`

​		常量(的)指针：是一个指针，并且指向的对象是常量，即指针本身不是常量，指向的对象是常量： `const int *`。

**记忆方式, 从右向左度表达式,**

​		指针常量:`int * const`:  最左边为const表示，该对象本身是一个常量，其次才属于指针。

​		常量指针: `const int *`: 可以理解为\*是一个取指针元素的含义, 即\*表示指向的对象, 外面在加上const, 表示指向对象是常量.

​		**常量指针与常量引用类似, 可以使用非常量对象对常量指针初始化, 其目的是为了防止指针对原来对象进行更改.**

```C++
int c = 50;

const int* p = &c;

c =50;

cout<<*p<<endl;

output:
50

```



### 顶层const与底层const

​		顶层const表示任意对象本身是常量，对任意数据类型都试用。

​		底层const表示指针或引用等复合类型所指对象是常量。

## decltype类型指示符

​		`decltype(f()) sum = x;`

​		`decltype(f())`返回表达式f()结果的类型. 用此类型对sum定义;

​		`int *p;`

​		decltype(*p) = int & 而不是int 这是由于解引用可以访问对象还能够对对象赋值。

​		对变量加上一层或多层括号，编译器会将其当为一个表达式, 变量是一种可以作为赋值语句左值的特殊表达式，所以这时decltype会将其当做引用。即 `int i; decltype((i)) = int &`;

## 显式类型转换

​		形式: `cast_name<type>(expression);`

​		`cast_name`为 `static_cast`、 `dynamic_cast`、`const_cast`、`reinterpret_cast`中的一种.

`static_cast:`

​		任何具有明确定义的类型转换, 只要不包含底层const, 都可以使用static_cast.

`const_cast:`

​		只能改变底层const.

ex:

```c
const char *pc;
char *p = const_cast<char *>(pc); //正确
```



## 数组

​		数组下标[]比解指针优先级高.

​		因此指针数组：

```c++
int *p[10]; //表示的是指针的数组(是数组, 元素是指针)
```

​		不存在引用的数组。数组之间不能直接赋值。

​		数组之间不能直接赋值.

​		数组(的)指针：

```c++
 int (*p)[10] = &arr;//指向元素为10个整数数组的指针.
```

​		数组的指针相当于构建了一个二维指针，其中第一维始终是0。`(*p)`可以看做`p[]` (但是却不能直接写`p[0][10]` ，这对应的是一个二维数组元素). 其中arr本来就是数组首地址, 再对其取址其实是没有用的(或许有?), 返回依然是原来地址。下个例子说明:

```C++
int w[] = {2, 5, 8};
cout<<w<<endl<<&w<<endl;
int (*q)[3] = &w;
cout<<endl;
for(int i=0;i<3;i++)
{
    cout<<(*q)[i]<<endl;
    cout<<&q[0][i]<<endl;
    cout<<q[0][i]<<endl;
    cout<<endl;
}

output:

0x7ffd5cd9ca88
0x7ffd5cd9ca88

2
0x7ffd5cd9ca88
2
5
0x7ffd5cd9ca94
5
8
0x7ffd5cd9caa0
8
```

​		可以看到w和&w输出一致，(*q)[i]与q\[0][i]输出一致。

​		数组与指针类似， 当一个指针指向数组中某个元素是，在拿这个指针使用索引访问数组元素等价于将该地址按照索引进行移动后解地址。

​		`begin()`与`end()`函数: 分别返回数组中第一个元素与最后一个元素下一个位置的地址。

### 字符串数组

```C++
char a[] = "ajjh0"; //长度为6, 因为后面默认存在一个空字符'\0'作为结尾.
```

​		对于char型数组来说，输出某个位置的地址时并不是直接输出地址，而是输出该地址及之后位置的字符串，同时对于字符串数组赋值来说，可以使用在cin时赋值，但不能使用string对其赋值。

```C++
char a[100];

cin>>a;

cout<<a<<endl;

a = "digusgy0"; // 错误

string w = "ugyu";

a = w; // 错误

```



### 多维数组

不存在多维数组, 实质是数组的数组.

## C分格字符串

​		cstring文件中

```C++
strlen(p); //计算长度, 不计算最后的空字符

strcmp(p1, p2); //比较两个字符串大小, > 取1, <取0

strcat(p1, p2);// 拼接两个字符串, 返回p1.

strcpy(p1, p2);// 拷贝字符串从p2到p1, 返回p1.

```

​		上述操作传入的参数必须是以空字符'\0'作为结尾的数组。

```C++
char a[] = {'a', 'b', 'c'}; 

strlen(a); //false

char a[] = {'a', 'b', 'c', '\0'};

stlen(a); //true. 输出3

```

## 运算符

sizeof 运算符: 计算表达式结果类型的大小，满足右结合率，对于数组，sizeof返回数组大小，而不是指针：

```C++
sizeof(type);

sizeof expr;

sizeof *p;
==
sizeof (*p);//由于sizeof不会计算实际表达式, 使用p指向空地址也是合法的, 依然会返回对应类型大小(字节)

```

### 结构体的sizeof

​		结构体的sizeof涉及对齐的问题。为什么需要字节对齐？计算机组成原理教导我们这样有助于加快计算机的取数速度，否则就得多花指令周期了。为此，编译器默认会对结构体进行处理（实际上其它地方的数据变量也是如此），让宽度为2的基本数据类型（short等）都位于能被2整除的地址上，让宽度为4的基本数据类型（int等）都位于能被4整除的地址上，依次类推。这样，两个数中间就可能需要加入填充字节，所以整个结构体的sizeof值就增长了。

​		字节对齐的细节和编译器的实现相关，但一般而言，满足三个准则：

1. 结构体变量的首地址能够被其最宽基本类型成员的大小所整除。

2. 结构体的每个成员相对于结构体首地址的偏移量（offset）都是成员大小的整数倍，如有需要，编译器会在成员之间加上填充字节（internal adding）。

3. 结构体的总大小为结构体最宽基本类型成员大小的整数倍，如有需要，编译器会在最末一个成员后加上填充字节（trailing padding）。

    注意：空结构体（不含数据成员）的sizeof值为1。试想一个“不占空间“的变量如何被取地址、两个不同的“空结构体”变量又如何得以区分呢，于是，“空结构体”变量也得被存储，这样编译器也就只能为其分配一个字节的空间用于占位了。

对于存在`virtual`的结构体，情况较为复杂，将来学习后再说。

### 联合体sizeof

​		结构体在内存组织上是顺序式的，联合体则是重叠式，各成员共享一段内存；所以整个联合体的sizeof也就是每个成员sizeof的最大值。

### 数组的sizeof

​		数组的sizeof值等于数组所占用的内存字节数。

​		注意：1）当字符数组表示字符串时，其sizeof值将’/0’计算进去。

​            2）当数组为形参时，其sizeof值相当于指针的sizeof值。 

例1：

```
char a[10];  
char n[] = "abc";   
  
cout<<"char a[10]                 "<<sizeof(a)<<endl;//数组，值为10  
  
cout<<"char n[] = /"abc/"           "<<sizeof(n)<<endl;//字符串数组，将'/0'计算进去，值为4
```

例2：

```
void func(char a[3])  
{  
    int c = sizeof(a); //c = 4，因为这里a不在是数组类型，而是指针，相当于char *a。  
}  
  
void funcN(char b[])  
{  
    int cN = sizeof(b); //cN = 4，理由同上。  
} 
```

### 函数的sizeof

​		sizeof也可对一个函数调用求值，其结果是函数返回值类型的大小，函数并不会被调用。

​		对函数求值的形式：sizeof(函数名(实参表))

注意：1）不可以对返回值类型为空的函数求值。 

​            2）不可以对函数名求值。

​           3）对有参数的函数，在用sizeof时，须写上实参表。

### 一些重要的运算符优先级:

{::}(作用域) > {',', '->', '[]'} > {'++', '--'}(后置) > {'++', '--'(后置), '*'(接地址), '&'(取地址)}

## try语块与异常处理

​		`throw`表达式，引发异常。即当检测到异常使用`throw`触发(抛出)异常，这时将终止当前函数。并将控制权交给能够处理该错误的代码。

```C++
int a[] = {2, 5, 6, 8};

int n;

cin>>n;

if(n>3)

{

   throw runtime_error("number over the length");

}

cout<<a[n]<<endl;

```

try语句块

```
try {

.........

}

catch (exception-declaration){.....}

catch (exception-declaration){.....}

....

```

catch捕获异常类型并相应做出处理.

# 函数

## 局部静态对象

​		在某个函数内定义一个变量,希望其生命周期为整个程序执行时间, 这时可以在前面添加static，表示是一个局部静态对象. 此时每次调用该函数时, 均共享该参数, 不会重复初始化。

```C++
int get_num()

{

​    static int num = 0;

​    return num++;

}

int main()

{

​    for(int i=0;i<10; i++)

​    {

​        cout<<get_num()<<" ";

​    }

​    return 0;

}
output: 0 1 2 3 4 5 6 7 8 9

```

​		与类类似, 一般期望函数申明在头文件(.h), 定义在源文件(.cpp).

## 传递参数

​		传值参数:初始化非引用类型变量时，是将初始值拷贝给变量．此时对变量改动不会影响原来变量。

​		传引用参数：传引用参数是定义一个别名，对引用的更改会作用到原来对象上。

​		当定义参数为引用时应该尽量定义为常量引用，这是由于当传递参数为一个常数是，不能使用普通的引用来作用于常量．例如：

```Ｃ＋＋
int add(int &a, int &b)

{

​    return a+b;

}

add(5, 6) //错误，５，６均为常值，这等于：int &a = 5; int &b = 6.

int add(const int &a, const int &b){....}　//这样就不会出错．

```

​		但有时就是要使用引用来更改源对象值时则另当别论，但要注意使用条件．

## main: 处理命令行选项

​		main函数包含于可执行文件prog之内, 可以向程序传递下面的可选项.

```C++
prog -d -o ofile data0

int main(int argc, char *agry[]) {.....}

==

int main(int argc, char **agrc) {......}

```

argc表示数组中字符串数量, argc是一个数组, 元素是指向C风格字符串的指针, 第一个元素指向程序的名字或者一个空字符串, 接下来的元素依次传递命令行的实参.

对于上面第一行, argv为5.

例如:

```C++
#include<iostream>
using namespace std;
int main(int argc, char **argv)
{
    for(int i=0;i<argc;i++)
        cout<<argv[i]<<endl;
    return 0;
}

input:

./main hhj jkh jkj jkjnj

output:

./main
hhj
jkh
jkj
jkjnj

```

## 不应该返回局部对象的引用或指针

​		局部对象会在函数调用完成后被注销, 这是其引用与指针将会失效(不再有效).

## 引用返回左值(即可当做变量被赋值)

​		调用一个返回引用的函数将得到左值, 其他返回类型将得到右值.我们可以为返回结果为非常量引用的函数的结果赋值.

ex:

```
char &get_str(string &str, int index)

{

​    return str[index];

}

int main()

{

​    string a = "hellow world";

​    get_str(a, 0) = 'H';

​    cout<<a<<endl;

​    return 0;

}

output:

Hellow world

```

## 函数重载

​		要保证不同的函数之间存在差异, 存在默认类型转换的参数会被认为是一样的, 形参一致(或存在默认转换)返回类型不同一样会被认为重复定义.

```C++
int add(double a, double b)

{

​    return a+b;

}

double add(double a, double b)

{

​    return a+b;

}

```

​		这会被认为重复定义了add.

​		一个拥有顶层const的形参无法和另一个没有顶层const的形参区分出来. 由于顶层const可以使用非const进行赋值,

```C++
int add(int a, int b)

int add(const int a, const int b)//重复定义add
int process(int *w)
int process(int * const w)//重复定义process

```

​		对于底层const来说是可以区分的(虽然const引用可以指向普通常量, 编译器会更具实参是否为常量进行选择)

```C++
int process(int &);

int process(const int &);

int process(int *);

int process(const int *);
//均不会被认为是重复定义

```

实例:

```C++
# 成员函数该函数希望通过引用对成员变量赋值, 隐含的参数为this指针
string& isbn()
{
    return bookId;
}
# 该函数只是希望读取类中对于数据, 隐含的参数是 const this, 这两个类内成员函数并不会被认为重复定义.
string isbn() const
{
    return bookId;
}

```



## 默认实参的默认参数放到后面, 且要避免函数重复定义

## 内联函数(online)(通常定义在头文件中)

## 函数指针

函数指针指向的是函数而不是对象.函数类型由其返回类型和形参共同决定而与函数名无关.

```C++
bool lengthCompare(const string &, const string &);

bool (*pf)(const string &, const string &);

```

其中(*pf)表示是一个指针, 右侧是形参列表, 表示其是指向的函数, 因此是函数指针.

```C++
pf = lengthCompare;

==

pf = &lengthCompare;

bool b1 = pf("hello", "goodbye");

==

bool b2 = (*pf)("hello", "goodbye")

==

bool b3 = lenthCompare("hello", "goodbye")

```

ex:

```C++
int *get_first(int *a)

{

​    return a;

}

int main()

{

​    int* (*w)(int *);

​    w = get_first;

​    int a[] = {2, 3, 5};

​    cout<<*w(a)<<endl;

​    return 0;

}
output:
2

```

## 返回函数的指针

虽然函数不能直接返回函数, 但是能够返回指向函数的指针.(花里胡哨,有点难理解)

# 类

类定义完成后一定要加分号, 这是由于类体后面可以紧跟着变量名(实例名).

```
struct Sales_item { /*........*/} accum, trans *salesptr;

==

struct Sales_item { /*........ */} ;

Scales_item accum, trans *salesptr;

```

编译器会先处理完全部申明再去处理成员函数定义, 因此数据成员定义的位置并不要求在成员函数之前.

## 类别名

1. typedef double wages, *p; wages表示double, *p表示double *.
2. using SI = Sales_item;

类还可以自己定义某种类型在类中的别名, 并且具有访问权限的属性.

```C++
public:

​    typedef string::size_type pos;

```

对此定义要放在使用pos之前.

类的成员必须是完全类型.

```
class Bar {

public:

​    ........

private:

​    static Bar meml; // 正确, static不受必须是完整成员的限制.

​    Bar *mem2; // 正确, 指针成员可以是不完整的(指针本身就是一个成员, 与指向无关)

​    Bar mem3;//错误, 数据成员必须是完整类型.(Bar本身就在定义整体, 不完整)

}

```



## 成员函数

成员函数申明必须在类内, 定义可以在类内也可以在类外.

### this指针

成员函数通过名为this的隐式参数来访问调用它的那个对象. 当调用成员函数时, 用请求该函数的对象的地址初始化this. 例如: total.isbn(), 编译器首先会把total的地址传递给isbn的隐式形参this, 可以认为调用该函数的实际过程为:

```
// 原始函数, 类是sales_data

string  Sales_data::isbn() const

{

​    return bookNO;

}

// 调用代码, 实例为total:

total.isbn();

// 实际执行过程理解代码(伪代码):

Sales_data::isbn(&total)

{

​    return this->bookNO

}

```

当我们只需要对对象内元素进行访问而不进行更改时, 应该将this变换为一个常量指针. 这就是函数参数后const的含义.

### const(常量)成员函数

在上述代码中, 我们看到isbn的参数列表后紧跟着一个const关键字, 这是由于isbn成员函数只是为了进行访问对象元素, 而我们不希望对其内容进行更改, 因此对this指针进行赋值是我们希望其是常量指针.  然而默认情况下, this的类型是指针常量(即this本身不会变, 详见const与指针)而非常量指针. 以上述代码来看, this 类型是 .

```
Sales_data * const

```

而我们希望的类型是:

```
const Sales_data * const

```

因此C++对这一步的转化方式是在成员函数参数列表后面完成转化. 此时该成员函数只能访问对象, 无法更改对象.

特别的, 常量对象或者常量对象的引用和指针都只能调用常量成员函数.

### 可变数据成员

在上述const成员函数, 我们不允许对类内成员进行更改, 但有时我们又希望能够对类内部分元素进行更改, 此时我们可以设置可变数据成员, 只需要在普通的变量申明中加上mutable关键字.

```
mutable int print_num = 0;

```

这时, 即使是const成员函数也可以对其进行更改.

### <font color=red>返回this指针</font>

当我们定义的成员函数类似于每个内置运算符时, 我们应该尽可能的去模仿这个运算符.(为了使得当我们在使用时像内置运算符一样便捷高效, 对于之后的运算符重载是一样的).  内置的赋值运算符将左侧运算对象当做左值返回, 因此为了与其保持一致, 当定义的函数类似于内置运算符并且将对类中某个元素进行修改时我们应该返回引用类型(引用类型可以作为左值, 同时可以利用返回的引用对对应元素直接赋值). 这是由于当对类内元素进行变更时, 必然存在语句value = exper;(其中vale为类内元素). 而利用上文提到的this指针,我们知道, 实际等于this->value = expr; 这就等于a.value = expre; 因此对于更改类中对象的操作等于赋值语句(高端赋值). 因此要返回引用类型, 而对于类似内置运算符的函数一般返回应该就是其本身, 因此应该返回的是*this;

```C++
Sales_data& Sales_data::combine(const Sales_data &a)

{

​    units_sold += a.units_sold;

​    revenue += a.revenue;

​    return *this;

}

```

为了说明这个操作类似内置运算符, 我们考虑:

```
a.combine(b.combine(c));

```

操作, 这可以实现a+b+c, 而仅仅只是为了实现元素相加时, 我们可能往往让其返回void, 这时就不能完全上述操作.

### 构造函数

构造函数名字与类名一致, 没有返回类型, 其他与成员函数没有不同,

当未手动定义构造函数是, 编译器会自动生成默认构造函数, 当自己定义了构造函数后依然希望存在默认构造函数时只需要在加上一个=default的构造函数即可.

构造函数初始化列表: 在构造函数参数之后使用类内成员(参数值)可以直接对类成员赋值. 且可以选择部分赋值不要求全部对象.

```C++
Sales_data() = default;

Sales_data(const string &s, const unsigned &num, const double &p): 

bookNO(s), units_sold(num),price(p), revenue(p*num){}

```

**成员变量初始化的顺序取决于其定义的顺序, 因此有时要格外小心, 不能使用未初始的变量对另外的变量初始化**.

例如:

```
class a

{

int i;

int j;

public:

​    a(int val) j(val), i(j) {};

}

```

这应该是有问题的代码, 正常初始化顺序是先i, 后j, 与构造函数后的初始化列表无关, 则这时我们会用未定义的j对i进行初始化, 这显然不是我们所期望的.

使用默认构造函数实例化时应该是classname instancename; 不能有括号, 不然会被认为是函数申明.

### 委托构造函数

委托构造函数使用所属类的其他构造函数, 执行它自己的初始化过程, 把自己一些或全部责任委托给其他构造函数.

```C++
// 非委托
Screen(pos a, pos b, char c): cursor(c),height(a), width(b) {} ;
// 委托上面的
Screen(pos a):Screen(a, 0, ' ') {};

```

### 拷贝构造函数

拷贝初始化通常由拷贝构造函数完成

无返回值, 名字与类名一致, 参数为常量类的引用:

参数由引用完成是因为如果传递一般的参数, 就会调用拷贝构造函数, 即调用直接, 这样就会一直调用下去直到溢出.

```
Sales_data(const Sales_data &w)
{
    bookNO = w.bookNO;
    units_sold = w.units_sold;
    revenue = w.revenue;
    price = w.price;
}

```

拷贝初始化发生条件:

1. 使用"="或使用s(s1)进行定义变量时发生.
2. 将一个对象作为非引用参数时.
3. 从一个函数返回非引用类型时
4. 用花括号列表初始化一个数组中的元素或一个聚合类的成员时.

### 拷贝赋值运算符

用来对已存在对象赋值.

使用重载赋值运算符. 返回为类型引用

```
Sales_data& operator=(const Sales_data &w)
{
    bookNO = w.bookNO;
    units_sold = w.units_sold;
    revenue = w.revenue;
    price = w.price;
    return *this;
}

```

拷贝赋值运算符与拷贝构造函数区别为:**调用的是拷贝构造函数还是赋值运算符，主要是看是否有新的对象实例产生。如果产生了新的对象实例，那调用的就是拷贝构造函数；如果没有，那就是对已有的对象赋值，调用的是赋值运算符**

```
Sales_data s("dsjdbs", 20, 3.14); 
Sales_data s2 = s;//调用拷贝构造函数
Sales_data s3; 
s3 = s;//调用拷贝赋值函数

```

### 析构函数

析构函数执行与构造函数相反的操作, 释放对象使用资源, 销毁对象的非static成员.使用~classname();构成, 无参数和返回值.

### 使用=default和=delete

在上述的构造,赋值, 析构函数中, 定义函数是使用=default表示使用编译器自定义的函数, 使用=delete表示组织编译器自定义的默认函数.(析构函数不能使用=delete)

```
Sales_data() = default;
Sale_data() = delete;
Sales_data(const Sales_data&) = defaule(delete);//使用默认拷贝,或阻止拷贝
Sales_data& operate=(const Sales_data&) = default(delete);//使用默认赋值, 或阻止赋值
~Sales_data() = default();

```



### 类数据成员初始化

可以使用类名加上构造函数的参数实现对类的初始化.

```C++
Sales_data w = Sales_data("jygu", 20, 25);// 初始化+赋值操作

// window_mgr类中数据成员
vector<Screen> screens{Screen(24, 80, ' ')}; // 使用上述方式进行初始化并对vector进行赋值

```

### 类的静态成员

成员与类本身无太大关系而与类的各个对象保持关联(例如统计类实例化的次数). 为此引入静态成员变量.

申明静态成员变量: 静态成员变量可以是public或private, 可以是常量, 引用. 指针, 类类型等等.

```C++
class Account{

public:

​    '''''''

​    static double rate() {return interestRate;}
     static void rate(double);
private:

​    ......

​    static double interestRate;
     static double initRate();

}

```

使用静态变量: 利用作用域运算符直接访问

```C++
double r;

r = Account::rate();

```

同时, 我们也可以使用类的实例来进行访问.

```
Account a;

a.rate()

```

成员函数可以直接对其进行修改, 任何实例均可.

静态成员初始化;

由于静态成员不属于任何一个对象, 因此定义应该在类外, 同时成员的static关键字只能出现在类内. 

```
double Account::interestRate = expre;//初始化static

```

对于静态成员属于常量表达式来说, 可以在类内进行定义:

```
static constexpr int period = 30;

```

## 拷贝控制与资源管理

### 类的行为像一个值

类的行为像一个值意味着它有自己的状态. 当我们拷贝一个像值的对象时, 副本和原来的对象完全独立. 改变副本不会改变原对象. **标准库容器与string类的行为均像一个值.**

### 类的行为像一个指针

像指针的类共享状态, 拷贝一个这种类型时, 副本与原对象使用相同的底层数据. 改变其中一个另一个也会改变. 这种情况比较常见于shared_ptr与new. 这时应该注意析构函数的使用. 应该使用一个整型指针指向一个区域来记录有多少个对象正在共享当前区域, 当为0时才能将对应的底层空间释放. 

### 对象移动

#### 右值引用(&&)

右值引用只能绑定到将要销毁的对象上. 一个左值表达式表示的是一个对象的身份, 而一个右值表达式表示的是对象的值. 我们不能将左值引用绑定到要求转换的表达式. 字面值常量或者返回右值的表达式上. **左值持久, 右值短暂.** 左值有持久的状态, 而右值要么是字面值常量, 要么是在表达式求值过程中创建的临时对象.

ex:

```c++
int i = 42;
int &r = i; //true 
int &&rr = i; //false. 右值引用不能绑定到左值上.
int &r2 = i*42; //false. 左值引用不能绑定到右值上.
const int &r3 = i*42;//true; 
int &&rr2 = i*42;//true. 右值引用绑定到右值上.

```

变量表达式都是左值(即变量就是左值. 右值引用也是一个变量). 因此我们不能将右值引用绑定到右值引用上. 即:

```c++
int %%r1 = 42; //true
int &&r2 = r1; //false. r1是一个左值

```

#### 标准库move函数

我们可以显示的将一个左值转换为对应的右值引用类型. move可以获得绑定到左值上的右值的引用.(即将一个左值转换为右值).定义在utility中.

```c++
int &&r3 = std::move(rr1);

```

#### 移动构造函数与移动赋值运算符

移动构造函数的第一个参数是该类型的右值引用. 与构造函数不同, 移动构造函数不分配新内存, 而是接管原对象的内存. 在移动拷贝构造函数和移动赋值运算符运行结束之后, 会调用原来对象的析构函数, 此时如果存在动态内存时应该注意, 不能将动态内存delete, 由于新生成的类(或是新更改的类)与原来的对象使用的是同一个动态内存空间. 如果delete了, 新生成的类将失效. ex:

```c++
StrVec::StrVec(StrVec &&s) noexcept : elements(s.elements), first_free(s.first_free), cap(s.cap) {s.elements = s.firet_free = s.cap = nullptr;}

StrVec w1(/* args*/);
StrVec w2(std::move(w1));

```

上例中, elements与first_free以及cap均是指向动态内存的指针. 对于非动态内存, 我们一样使用原始的方式进行赋值. 随后将指向动态内存的指针全部置位空指针. 使得在结束拷贝后执行w1的析构函数是保证动态内存不会被delete. 这就保证了w2的正常使用. 

#### 右值引用与成员函数

类的成员函数也可定义移动版本. 一般是正常版本接受参数为const的左引用, 移动版本接受的是非const的右引用.

ex:

```c++
void push_back(const string &s)
{
    w.push_back(s);
}
void push_back(string &&s)
{
    w.push_back(std::move(s));
}

string s1 = "dddd";
push_back(s1); //调用正常版本.
push_back(std::move(s1)); //调用移动版本.

```



## 访问控制与封装

### 访问说明符

public: 程序内均可访问, 是类接口.

private: 只能被类内成员函数访问. 隐藏了类的实现细节.

一般构造函数和部分成员函数在public之后, 数据成员在private之后.

class与struct的区别只在默认的访问权限上, class是private, struct是public,  还有就是在继承关系上的不同, 如果为指明继承关系, 则class定义派生类为private继承, struct定义的派生类为public继承.

### 友元

#### 友元函数

由于一般将数据类型设置为private的权限, 因此,对于非成员函数来说则无法直接访问, 为了可以使得非成员函数可以对类进行访问, 我们可以设置其为友元函数. 只需在类内添加一条以friend为开头的函数申明语句即可.(不要求其权限, 在public和private均可)

```C++
friend istream& read(istream &, Sales_data &);

friend ostream& print(ostream&, const Sales_data &);

```

**注意: 类内申明的友元并非通常意义下对函数的申明, 只是定义了访问权限,要想进行函数使用, 类外依然要再次申明.**

#### 友元类

当类A中有数据成员为类B, 而类B中包含私有成员, 这将使得类A对其访问十分不便, 为了解决这个问题. 引入友元类概念. 在类B中添加类A为其友元类: friend class A;

```
friend class window_mgr;

```

**注意: 友元关系不具有传递性, 即A是B的友元类, B是C的友元类, C也不能够直接访问A的私有变量, 对于友元函数来说同样试用.**

**每个类负责控制自己的友元类和友元函数**

有时我们并不希望类B完全拥有对类A所以的访问权, 只希望个别成员函数可以访问A, 此时我们可以令**成员函数作为友元**. friend void B::function();

```
friend void window_mgr::clear();

```

## 面向对象编程(OOP)

核心思想为: 数据抽象, 继承和动态绑定. 多态

### 继承

派生类: 使用派生类列表, 在类后面使用冒号, 后面跟着以逗号分割的基类列表, 其中每个基类前面添加访问说明符.

```C++
class bulk_quoto: public quoto{
    public:
    double net_price(size_t n) const;
};

```



### 虚函数

基类将与类型相关的函数(不同派生类对应函数不同)与派生类不做改变直接继承的函数(各个派生类函数一致,均试用基类中定义的)区分对待.当基类希望它的派生类各自定义适合自己版本的函数, 此时基类可以将函数申明为虚函数. 只需要在函数名前加上virtual关键字即可. 继承类重新定义的虚函数会将基类的虚函数覆盖.  继承中使用虚函数时, 一定要进行申明.

```C++
class quoto{
    public:
    string isbn() const;
    virtual double net_price(size_t n);
};

class quoto{
    public:
    double net_price(size_t n); //一定要申明才能使用.
}

```

对虚函数的调用可能会在运行时才被解析.(当且仅当调用虚函数的是指针与引用才会在运行时被解析)

当我们使用基类的引用或指针调用虚函数时会执行动态绑定. 因为直到运行时才能知道到底调用了那个版本的虚函数, 因此所以虚函数都必须有定义. **虚函数调用的选择为动态类型相匹配的那个.**(动态类型详见下文动态类型与静态类型)

### 动态绑定(运行时绑定)

使用动态绑定我们可以使用同一段代码处理基类与派生类.

```C++
double printf(ostream &os, const quoto &w, size_t n)
{
    double ret = w.net_price(n);
    os<<"ISBN: "<<w.isbn()<<" # sold:"<<n<<" total due: "<<ret<<endl;
    return ret;
}

```

我们即可以使用quote也可以使用bulk_quote执行上述代码.

<font color = red>动态绑定只有在我们通过指针或引用调用虚函数时才会发生</font>

### protected访问运算符

protected表示受保护的类型, 意思是该类型只希望该类本身和其友元以及派生类可以访问, 而其他不允许访问. 注意: **private类型即使是派生类也没有权限访问**, 这一点比较难以理解, 虽然类拥有该数据成员但却无权限访问. 具体解释将在之后的部分解释.

### 定义派生类

使用**派生类列表**指出该派生类继承于那个类或那些基类, 同时可以指出希望继承的方式(public, protected, private).其中继承方式说明符只是用来说明派生类原本具有访问权限的基类部分成员(即public和protected)继承到派生类后其访问权限.其中protected最低权限也是protected, 不可能成为public.

```C++
class sphere: public graphy

```

派生类必须将继承而来的成员函数中需要覆盖(虚函数)的那些部分重新定义. 负责在动态绑定时将找不到该要调用哪个函数了.

### <font color = red>派生类向基类的类型转换</font>

派生类含义基类的对应组成部分, 所以我们能够把派生类当做基类使用. 在这个过程中我们将基类的指针或引用绑定到派生类对象中基类部分. 

**只能通过基类的指针或者指针实现.** 

个人理解: 使用基类指针或引用时并未执行拷贝操作, 只是相当与将相对与基类中多加的部分添加一个限制(类似于删除). 此时我们访问派生类中基类部分时访问的其实还是派生类的元素, 只是看起来似乎我们在使用基类. 

注意: 不存在对象之间的类型转换.(基类与派生类之间转换要通过基类指针或引用, 也不能是对象之间直接转换)

```C++
sphere w1;
graphy w2 = w1;
graphy &w3 = w1;

```

按照上述所述第二行按道理应该是错误的. 但实际往往不会如此, 这是由于无论拷贝构造函数还是重载的拷贝赋值运算符参数都是对于常量的引用:

```C++
graphy(const graphy&);
graphy& operate = (const graphy&);

```

此时执行w2 = w1;操作时首先执行上述的两个函数中的一个(具体是哪个参考拷贝赋值运算符节),此时已经完成了类似于第三行的基类引用到派生类的绑定, 传入函数中的已经是基类了, 因此可以完成. 当拷贝赋值运算符参数不是引用时, 依然是正确的, 这是由于如果不是引用作为参数时就会首先运行拷贝构造函数, 依然完成了转化. 而拷贝构造函数参数必须是引用, 因此在代码中我们会发现: **基类 = 派生类总是不会报错, 但我们应该看到其实际是不合理的, 之所以不报错是因为在执行这一步过程中包含一步基类的引用 = 派生类的操作, 这将造成额外的花销, 因此应该尽量避免这个操作, 尽量使用基类的引用或指针 = 派生类.** 想要证明这一点很简单,只需要单步调试即可, 就能发现.

同时应该注意, 不存在从基类向派生类的隐式类型转换.

### 派生类构造函数

派生类构造函数应该使用基类的构造函数对基类部分进行初始化. 这能够避免考虑访问权限问题.

```C++
sphere(string s, double r): graphy(s), rito(r){} // 其中graphy使用了基类的构造函数

```

<font color =red>继承体系中默认初始化顺序</font>

在派生类的构造函数中一般会使用构造函数初始化列表, 初始化的顺序是哪个数据成员定义在前先初始化哪个, 因此对于派生类来说, 基类的成员被最先定义, 因此会先初始化基类部分的成员数据, 同样, 在初始化基类部分的成员时又会先初始化基类的基类部分, 因此顺序是**从基类到派生类的顺序**.

派生类构造函数还可以简单的使用基类的构造函数, 只需要使用using语句即可. 此时具有特殊性, 此时使用using语句不会改变构造函数的访问级别, 在基类中的访问级别和派生类的一致.

```
A是基类, B是派生类:
using A:A;
等价于:
B(rps):A(rps){}

```



### 派生类赋值运算符

派生类赋值运算符与拷贝构造函数一样, 必须显示的为其基类部分赋值.

```c++
D & D::operate=(const D&rhs)
{
    Base::operate=(rhs); //显示的调用基类的赋值运算符为基类部分成员赋值.
    /''''/
    return *this;
}

```



### 静态成员的继承

基类中定义了一个静态成员, 则整个继承体系中只存在改成员的唯一定义, 对派生类来说共享一个静态成员.

### 派生类申明

派生类申明包含类名但不包含派生列表.

```C++
class sphere; // 正确
class sphere: public graphy; // 错误

```

### 防止继承的发生

当我们不希望某个类成为基类时, 我们只需要在类名后加上final就表示该类不能作为别的类的基类:

```C++
class sphere final{}; 
class sphere final : public graphy{};

```

### 静态类型与动态类型

对于继承体系来说, 静态类型与动态类型应该区分开. 静态类型表示我们定义的类型, 而动态类型是指, 实际给我们的类型. 

<font color = red>动态类型与静态类型是相对于继承体系中派生类向基类的转换而言的, 对于非指针与引用类型来说动态类型与静态类型一致.</font>

例如:

```C++
sphere w1;
graphy &w2 = w1; // 对w2来说, 静态类型是graphy, 动态类型是sphere
graphy w3 = w2; // w3的静态类型与动态类型均是graphy
graphy w4 = w1; // w4的静态类型与动态类型也是一样的. 其实w4 = w1;这个操作等于上面两个操作之和.

```



### 派生类的虚函数

派生类覆盖了某个虚函数时, 可以再次使用virtual关键字指出该函数的性质(为了以次派生类作为基类进行继承时使用). 然而这是非必须的. 一个函数被申明为虚函数, 则在所以派生类中均是虚函数(包括派生类的派生类).

派生类覆盖虚函数要求参数必须与基类中一致, 否则被认为是隐藏了虚函数并重新定义了一个函数.

派生类中虚函数的返回值也应该与基类中一致. 唯一的例外是当返回值为类本身的指针或引用时, 允许返回类型不同. 但这要求从派生类到基类的类型转换是可访问的.(派生类到基类可访问性详见下文)

### final与override说明符

在派生类中, 如果没有指明是对虚函数的覆盖而定义了一个错误的虚函数的覆盖(实际希望覆盖虚函数但只达到了隐藏的效果), 这种情况会在参数列表不一致或者返回值不一致时发生, 编译器并不会报错, 但实际是严重的错误. 为了避免这种情况, 我们在派生类中对虚函数的定义后(参数列表后)加上override, 此时如果没有真正实现对虚函数的覆盖则编译器报错. override只对虚函数有用.

有时我们希望定义的虚函数在接下来的派生类中不要再被覆盖了, 保持使用当前类中的, 此时我们可以在虚函数的参数后加上final, 表示以此类为基类的派生类不能覆盖当前的虚函数.

ex:

```C++
struct B
{
    virtual void f1(int) const;
    virtual void f2();
    void f3();
};
struct D1 : B
{
    void f1(int) const override final; //正确, override检验也正确, 且申明为final
    void f2(int) override; // 错误, override使用正确, 但是并为真正实现覆盖, 报错
    void f3() override; // 错误, f3不是虚函数
    void f4() override; // 错误, 不存在f4函数
};
struct D3 : D1
{
    void f1(int) const; //错误, D1中已经定义f1为final ,这里不能覆盖.
};

```

### 虚函数与默认实参

使用指针或引用调用虚函数, 如果含有默认实参则实参值由静态类型决定.

### 回避虚函数机制

当我们使用指针或引用调用虚函数时, 我们可以回避动态绑定, 使用域运算符即可:

```C++
double count = baseP->quoto::net_price(42); //其中baseP为quoto的派生类实例的指针

```

### 抽象基类

当我们定义一个虚函数成员函数, 其没有实际意义, 只是为了以此为基类的成员能够使用动态绑定统一使用该虚函数接口, 各个派生类都重新定义自己的虚函数时. 我们可以将这个函数定义为**纯虚函数**, 该类成为抽象基类. 对于纯虚函数定义, 我们只需要在一般成员函数定义中加上=0;同样需要用关键字virtual. 

**我们不能创建一个抽象基类的对象, 只能创建引用或指针.**

为了方便理解, 我们可以举个例子:

```C++
struct graphy
{
    double get_area() const = 0;
};
struct sphere : graphy
{
    double get_area() const;
};
struct square: graphy
{
    double get_area() const;
};
.....

```

这里我们定义了一个抽象基类图像, 由于图像的面积不好计算, 因此我们只希望能够计算规则的图像, 所以我们将图片类作为一个抽象基类, 只是提供一个统一的接口. 而后我们对各个图像定义对应的面积计算函数. 而当我们使用时, 我们可以将任意类赋值到基类的指针或引用上, 此时利用动态绑定, 无论是那个图像, 我们都能够使用get_area函数获得图像的面积.

### 继承中的类作用域

派生类的作用域嵌套在基类的作用域里, 如果一个名字在派生类中找不到, 就会在外层作用域查找, 也就是其基类中查找.

<font color = red>名字查找优先于类型检查:</font>

即申明在内层作用域的函数并不会重载申明在外层作用域的函数, 即使参数和返回值不同. 编译器只会优先查找函数名, 当在内层作用域查到函数名就会终止查找了, 如果参数和返回值不匹配则会报错.

### <font color = red>名字查找与继承</font>

理解成员调用时的解析过程对于理解C++继承至关重要, 总共分为四步:

1. 首先确定静态类型.
2. 在静态类型的类中查找对应成员, 如果找不到, 就在其基类中查找(只能查找派生类运行访问的成员, 即public和protected), 如果找不到就继续往下找, 终止条件是找到成员名字一致(只要求名字一致, 参数可以不一致, 原因: 名字查找优先于类型检测)或者找到最终的基类也没找到(报错).
3. 找到对应成员后再进行类型检测(对于函数而已), 判断调用是否合法.
4. 调用合法时, 编译器会根据是否为虚函数进而产生不同代码, 如果不是虚函数, 生成常规函数调用, 如果是虚函数且是通过引用或指针调用的, 则将在运行时选择执行哪段代码.

### <font color =red>隐藏与覆盖的区别</font>

覆盖是单对于虚函数而言的, 当我们在派生类中对虚函数重新定义时, 基类中的虚函数就会被隐藏. 需要注意, 当派生类中存在与基类中虚函数名字一致但参数列表不一致时, 并不会覆盖基类中的虚函数, 这相当于重新定义了另一个函数, 实际会将基类中的虚函数隐藏.

对于一般的成员函数, 在派生类中只有出现与基类中名字一样的普通成员函数, 测根据名字查找优先于与类型检查, 当调用该函数时, 由于派生类中存在一个该名字的函数, 就不会再向基类中查找了, 这时基类中的同名函数就会被隐藏. 

个人理解: <font color = red>隐藏阻止了查找时向基类的查找, 覆盖允许调用时从基类向派生类中的调用.</font>

### 覆盖重载的函数

当基类中存在成员函数, 我们希望也被派生类使用, 但我们又希望定义一个或多个对其重载, 此时, 由于名字查找优先级大于参数匹配, 我们一旦定义同名函数, 基类中的就会被隐藏.  所以, 一个策略是将基类中的函数在派生类中在写一遍, 再添加新的重载, 但是这显然比较麻烦, 还有一个方式则是使用using申明语句. 申明我们希望使用基类中的成员函数, 而后我们只需定义派生类中特有的重载即可. 而且此时如果存在派生类的函数参数与基类的一致, 基类的将会被隐藏, 依然使用派生类中自定义的函数.

```C++
struct A
{
    double get_m();
    double get_m(int);
    double get_m(int, int);
};
struct B : A
{
    using A::get_m; //申明B要使用A内的get_m函数
    double get_m(int, int); //自定义了一个get_m重载函数, 此时A中get_m(int, int)将被隐藏.
}

```

### 访问控制与继承

对于继承来说, 存在三种方式, 即public, protected, 以及private. 这只是说明其继承中将基类成员继承为何种成员. 某个类对其继承而来的成员访问权限受两个方面的影响, 一个是在基类中的访问说明符, 一个是派生列表中的访问说明符.具体如下:



| 基类中访问说明符 | 派生列表说明符 | 派生类中访问权限 |
| ---------------- | -------------- | ---------------- |
| public           | public         | public           |
| public           | protected      | protected        |
| public           | private        | private          |
| protected        | public         | protected        |
| protected        | protected      | protected        |
| protected        | private        | private          |
| private          | --             | 无访问权限       |



解释: 对于基类中是private的成员, 派生类访问权限完全取决于派生列表中希望其拥有的访问权限. 对于基类中protected, 基类本身就是希望其只能在基类与派生类对其进行访问, 因此至少其访问权限是protected, 如果继承列表使用private,则其访问权限会被加强为private. 对于基类中的private成员, 基类本身就不希望派生类能够访问, 因此, 无论如何其都没有对private的访问权限, 但派生类又确实存在该成员. 因此, 如果希望访问private的话只能通过基类中定义的接口函数.

对于派生类访问基类的private数据成员的解释:

当派生类想要调用基类的private数据成员时, 首先会在自己的作用域中查找, 当然是找不到的, 而后会在基类中查找, 但是, 在基类中查找时, 其并没有查找private的权限, 因此也是找不到的,此时就会报错. 而当使用基类对于的接口函数时则是可以的. 同样先在派生类中查找, 找不到回去基类中查找, 找到后, 基类的接口还是有权限访问private数据成员, 此时就可以访问到派生类无法访问到的基类中private成员.

### 派生类向基类转换的可访问性

派生类向基类的转换是否可访问(是否允许)由使用该转换的代码决定, 同时派生列表的访问说明符也有影响.

(1)只有派生类是public继承基类时, 用户代码才能使用派生类向基类的转换:

```C++
struct A
{
    ...
};
struct B: public A
{
    ...
};
struct c: private A
{
    ...
};

A w1;
B &w2 = w1; //正确, public继承, 用户代码可访问从派生类到基类的转换
C &w3 = w1; //错误, private和protected继承,用户代码不可访问从派生类到基类的转换

```

(2)不论派生类以何种方式继承基类, 派生类的成员函数和友元均可以使用派生类到基类的转换.

```C++
struct A
{
    ...
};
struct B : public/private/protected A
{
    void change()
    {
        B w1;
        A &w2 = w1; //正确, 作为B的成员函数, 从B到A的类型转换始终可以访问
    }
    friend void ex_change();
};
void ex_change()
{
    B w1;
    A &w2 = w1; //正确, 作为B的友元函数, 从B到A的类型转换始终可以访问
}

```

(3)如果派生类继承基类的方式是public或protected的, 则派生类的派生类的成员函数或者友元可以使用从派生类向基类的转换, 而如果是private继承则不行.

```c++
struct A;
struct B :public/protected A;
struct C: private/protected/public: B
{
    void change()
    {
        B w1;
        A &w2 = w1;
    } //正确
}

```

### 友元与继承

友元关系不能继承. 基类的友元可以访问基类的成员, 这种访问包括了派生类中内嵌的基类部分, 即**派生类的基类部分对于基类的友元来说也是可以访问的**. 友元与类本身对类中所以成员具有相同的访问权限.

```c++
class A
{
    friend class C;
    protected:
    int m;
};

class B : public A
{
    int j;
};

class C
{
    int f(B b)
    {
        return b.m; 
    }// 正确, B是A的派生类, C是A的友元, C可以访问B中A的成员部分.
}



```

### 改变个别成员可访问性

有时我们需要改变基类中某个成员在派生类中的访问级别, 我们可以使用using申明(对于数据成员以及函数成员均有效).

### 虚析构函数

由于我们在使用在使用动态绑定时需要使用派生类的引用或指针, 则当我们delete一个new出来的指针所指向区域时, 如果派生类没有一个虚析构函数则会产生未定义的行为. 因此,在继承体系中, 我们应该将析构函数定义成虚函数. 

对于析构函数来说, 其销毁顺序是从派生类成员到基类成员. 与构造顺序相反.

### 虚继承

在多重继承中, 有可能某个派生类继承两个基类, 并且这两个基类来自于同一个基类. 即A是B,C的基类, B,C又是D的基类, 此时D就存在两套A的成员, 显然这是不希望存在的. 此时可以使用虚继承. 

虚继承virtual说明符表示了一种愿望, 即在后续的派生类中共享一份虚基类的实例.

上述A,B,C,D可以使用如下方式解决.

```c++
class A;
class B:public virtual A
{};
class C:public virtual A
{};
class D: public B, public C
{};

```

此时D只存在A的一份成员. 

虚继承中构造函数与析构函数比较特殊. 

构造函数:

首先构造虚基类子部分, 而后按照直接继承中出现的顺序依次构造, 不过在构造直接继承时不会对虚基类子部分进行构造, 最后构造自己的部分. 即:

```c++
D::D(args): A(args_a), B(args_b), C(args_c){};

```

在构造B, C时不会构造A的子部分, 虽然会传递A子部分的参数. 当没有显示调用虚基类A时, 会先执行A的默认构造函数.

析构函数与构造函数顺序相反.

### 在构造函数与析构函数中调用虚函数

派生类构造函数被执行时, 先从基类部分构建, 当构造函数中要使用虚函数, 按照正常操作应该使用动态类型也就是说应该执行派生类中的虚函数, 但是派生类虚函数并未被构造, 处于未完成状态. 而派生类析构函数被执行时, 从派生类成员到基类成员被析构, 当执行到基类的析构函数时, 如果基类的析构函数调用虚函数, 同样根据动态绑定应该执行派生类的虚函数, 但不说了的虚函数已经被析构. 上述两个问题都会导致错误. C++处理方式是, 编译器认为对象在执行构造函数和析构函数时, 类型仿佛发生了变话. 即当执行基类部分的构造或析构函数时, 认为对象就是基类, 只调用基类的虚函数.

## 运算符重载

一元运算符有一个参数, 二元运算符有两个. 对于二元运算符来说, 左侧运算对象传递给第一个参数, 右侧运算对象传递给第二个参数. 如果运算符是成员函数, 则左侧运算符对象隐式的绑定到this指针上.  对于一个重载的运算符来说, 其优先级和结合率不变.

### 递增递减运算符

为了区分前置后置递增递减运算符, 在函数申明时在参数中添加一个未使用的形参int, 这个形参唯一作用就是区分前置后置, 而不是参与运算.

```c++
T& operate++(); //前置
T& operate--();

T& operate++(int);//后置
T& operate--(int);

```

### 函数调用运算符

类中可以重载函数调用运算符. 此时, 我们可以像使用函数一样使用该类. 

ex:

```c++
struct absInt
{
    int operate()(int val) const
    {
        return val>0? val:-val;
    }
}

```

如果类定义了调用运算符, 则该类的对象称作函数对象. 行为像函数一样. **函数对象也是可调用的**

除了重载函数调用运算符, 类中也可以存在数据成员和其余的函数成员. 

### lambda是一个函数对象

使用lambda表达式时, 编译器将该表达式翻译成一个未命名类的未命名对象. 类中包含一个重载的函数调用运算符. 对于捕获的参数, 会变成类中成员变量. (对于lambda的基本介绍看下面STL中泛型算法部分)

ex:

```c++
[sz](const string &s)->bool{return s.size()>sz;};
==
class no_name
{
    public:
    no_name(int n) sz(n);
    bool operate(const string&s)
    {
        return s.size()>sz;
    }
    private:
    int sz;
};

```



# STL标准库

## IO类

IO对象无拷贝或赋值操作.

```C++
ofstream out1, out2;

out1 = out2; //错误, 不能进行拷贝.

ofstream print(ofstream)//错误, 不能初始化IO类参数

out2 = print(out2); // 错误, 不能拷贝流对象

```

由于不能拷贝流对象, 因此不能将形参与返回值设置为流类型, 通常以引用类型进行传参与返回. 读写一个IO对象会改变其状态, 因此引用不能是const的.

### IO类型条件状态

IO类最大的问题就是会发生错误, 因此,我们定义其条件状态来帮助我们访问和操作流. 下面是IO类定义的函数和标注.

```C++
strm::iostate //strm是IO类型, iostate是机器相关类型, 表示条件状态
strm::badbit;//指出流已崩溃
strm::failbit;//指出IO操作失败
strm::eofbit;//指出流达到文件末尾
strm::goodbit;//流处于正常状态. 正常是为0
s.eof();//s处于eofbit状态返回true
s.fail();//s的failbit或badbit处于置位状态, 即为1, 返回true
s.bad();//s的badbit处于1返回true
s.good();//若s处于有效状态, 返回true
s.clear();//将流s中所有条件状态复位, 将流的状态设置为有效. 返回void
s.clear(flags);// 给定指定flag位置, 将流中对应条件状态位复位, flag为str::iostate类型, 返回void
s.setstate(flags);//给定指定flag位置, 将流中对应条件状态位置位, flag为str::iostate类型, 返回void
s.rdstate();// 返回s流当前状态, 返回类型为strm::iostate

```

### iostream

标准输入((w)istream)输出((w)ostream), 由于不能拷贝流对象, 因此is>>和os<<返回值应该是对应的引用. 对于类内定义输入输出函数, 返回应该是对应引用.

```
istream &read(istream &is, Sales_data &a)

{

​    is>>a.bookNO>>a.units_sold>>a.price;

​    a.revenue = a.price*a.units_sold;

​    return is;

}

ostream &print(ostream &os, const Sales_data &a)

{

​    os<<a.bookNO<<" "<<a.price<<" "<<a.units_sold<<" "<<a.revenue<<endl;

​    return os;

}

```

### fstream:文件输入输出

操作:

```C++
fstream fstrm; //创建fstream实例, 未绑定

fstream fstrm(s); //创建fstream实例,打开s并与fstrm绑定

fstream fstrm(s, mode);// 创建fstream实例, 以mode模式打开s并与fstrm绑定

fstrm.open(s); //打开文件s并与fstrm关联

fstrm.close(); //关闭fstrm关联的文件

fstrm.is_open()//指出fstrm关联的文件是否打开

```

文件打开方式:

```C++
in; //读方式
out; // 写方式
app;//写操作前定义于文件末尾, 当使用app时默认添加out
ate;//打开文件后立即定位到文件末尾.
trunc; //截断文件, 只有在out被设置下才可以使用
binary; //以二进制进行IO

```

注意: 对文件的操作我们要将文件看做正常程序的控制台, 要从文件中读取数据时应该是in(类比于从控制台输入), 要向文件中写东西时应该是out(类比于向控制台输出).

```C++
    fstream s("test.cpp", ifstream::in);
    if(s.is_open())
    {
        string line;
        while (getline(s, line))
        {
            cout<<line<<endl;
        }
    }
    else
    {
        cout<<"cant'n open file"<<endl;
    }
    return 0;

```

### sstring

sstring是将string当做一个IO对象来处理(上面一个是文件, 这个是string).

istringstream是向string中读数据, ostringstream是向string中写数据.

stringstream特有操作:

```
sstream strm;// 申明stringstream实例, 未绑定

sstream strm(s);// 申明stringstream实例, 保存string的一个拷贝. 是explicit的(不存在默认转换)

strm.str(); // 返回strm所保存的string的拷贝

strm.str(s);// 将string s拷贝到strm中.

```

## 迭代器(iterator)

标准库容器均为迭代器.

迭代器类型不仅包含元素本身, 同时还包含返回迭代器的成员(即指向成员的迭代器), 这与指针不同, 指针需要取地址符, 而迭代器本身还有指向地址的元素.

迭代器包含的指向成员的迭代器包含 begin()和end(), 其返回的是指向指向第一个元素和最后一个元素的后一个位置的迭代器(类似于指向对应位置的指针).

由于成员选择".", "->"的优先级高于解引用"*", 因此要获得对应元素要先获得迭代器再解引用.

```
*a.begin()

==

*(a.begin())

```

迭代器类型:

iterator和const_iterator表示迭代器类型.

迭代器运算:

迭代器加减常数表示迭代器前后移动. 两个迭代器相减表示两个迭代器距离.

比较, >, < ,==, ... 均表示迭代器所指向元素位置的前后关系.

**迭代器类型不但支持++, --操作, 还能直接加常数.**

## 顺序容器

```
vector; 可变大小数组, 随机访问, 除尾部以为位置插入很慢
deque;  双端队列.
list; 双向链表
forward_list; 单向链表
array: 固定大小数组. 不能添加删除元素
string; 与vector类似, 不过专门用来存储字符.

```

### 容器操作

#### 常规操作

```
类型别名:
iterator: 此容器类型的迭代器类型.
const_iterator:可以读取元素，不能修改元素的迭代器类型(类比于常量指针.)
size_type: 无符号整数类型, 足够保存此种容器类型的最大可能容器的大小.
difference_type: 带符号整数类型, 表示两个迭代器距离
value_type: 元素类型
reference: 元素左值类型, 等价于value_type &
const_reference: 元素的const左值类型, 等价于const value_type &

构造函数:
C c:
C c1(c2):要求容器类型与元素类型均匹配
C c(b, e): 构造c,使用迭代器b和e之间指定范围的拷贝到c, array不支持, 容器类型与元素类型均可不同, 只要求            元素类型可以实现转换即可.
C c{a, b, c, ....}: 列表初始化
C c(n, t); n个元素, 均使用t进行赋值.指定容器大小.
// array具有固定大小.
array<int 10> w; //正确
array<int> w; //错误, 必须指明大小.

赋值与swap: 要求c1, c2类型一致, 即容器和元素类型均一致
c1 = c2;
c1 = {a, b, c, ...};
a.swap(b); 交换
swap(a, b);

大小:
c.size(); 不支持foreard_list
c.max_size(); c中可保存的最大元素数量.
c.empty();

添加/删除元素(不适用array)(不同容器中不同):
c.insert(args);
c.emplace(args); 替换
c.erase(args); 删除
c.clear(); 删除所以元素.

关系运算符:
==, !=: 所有容器均支持
<, >, <=, >=; 无序关联容器不支持.

获取迭代器:
c.begin(), c.end(); 返回首尾位置迭代器
c.cbegin(), c.cend(); 返回首, 尾(之后)位置const 迭代器.

反向容器的额外操作(不支持forward_list):
reverse_iterator: 按逆序寻址元素的迭代器(即++往前走, --往后走)
const_reverse_iterator: 不能修改元素的逆序迭代器
c.rbegin(), c.rend(); 返回指向尾, 首(之前)的元素位置的迭代器.
c.crbegin(), c.crend(); 返回const_reverse_iterator

```

举例:

```C++
    list<int> w({2, 3, 4, 5, 6, 7});

​    auto index = w.cbegin();

​    for(int i=0;i<w.size();i++)

​    {

​        cout<<*index<<" ";

​        index++;

​    }

​    cout<<endl;

​    auto index2 = w.crbegin();

​    for(int i=0;i<w.size();i++)

​    {

​        cout<<*index2<<" ";

​        index2++;

​    }

output:
2 3 4 5 6 7 
7 6 5 4 3 2 

```

**assign函数(仅适用于顺序容器)**

assign允许我们使用迭代器对容器和元素(可以实现转化)均不同的的对象实现赋值操作. 类似于使用迭代器初始化对象, 不过这个操作是对已经存在的对象.(之前的内容将被丢弃)

```C++
list<int> w({2, 3, 4, 5, 6, 7});

deque<float> w2;

w2.assign(w.rbegin(), w.rend());

```

第二个版本的assign还是与初始化时一样, 使用一个整数和一个元素表示对其赋值操作.

```
w2.assign(10, 1);

```

**swap交换操作**

swap操作除了对于array容器外, 均是元素本身未被交换, 只是交换了两个容器内部数据结构. 这意味着原本指向容器的迭代器, 引用, 指针均不会失效. 它们将指向交换之前的那些元素, 但是这些元素已经不属于原本的容器了.(string除外, 对string调用swap将导致上述内容失效, array也除外, 交换array会交换元素).  ex:

```C++
 list<int> w1({2, 3, 4, 5, 6, 7});

​    list<int> w2 = {5, 7, 9, 10};

​    auto index = w1.begin()++;

​    cout<<*index<<endl;

​    swap(w1, w2);

​    cout<<" w1: "<<endl;

​    for(auto i = w1.begin(); i!=w1.end();i++)

​    {

​        cout<<*i<<" ";

​    }

​    cout<<endl<<"w2:"<<endl;;

​    for(auto i = w2.begin(); i!=w2.end();i++)

​    {

​        cout<<*i<<" ";

​    }

​    cout<<endl<<*(++index)<<endl;

output:
2
w1: 
5 7 9 10 
w2:
2 3 4 5 6 7 
3

```



**关系运算符**

对于比较来说必须要求两边类型完全相等(容器元素均相等).

当两个容器大小不同, 且较小的与较大的对应元素一致, 则较小的容器小.(这里较大较小指元素数量)

当两个相互都不是对方的前缀子序列则比较第一个不同元素,

​    

```
string a = "asdfg";

​    string b = "asdfghjk";

​    if(a>b)

​    {

​        cout<<a<<">"<<b<<endl;

​    }

​    else

​    {

​        cout<<b<<">"<<a<<endl;

​    }
output:
asdfghjk>asdfg

```



#### 向顺序容器添加元素

```C++
forward_list有自己版本的insert和emplace;
forward_list不支持push_back和emplace_back;
vector和string不支持push_front和emplace_front;

c.push_back(t):c的尾部创建值为t或由args创建的元素. void返回.t为元素, args为可以转化成元素的参数
c.emplace_back(args):

c.push_front(t):c的头部添加元素 vector和string不支持.
c.emplace_front(args):

c.insert(p, t):在迭代器p之前添加元素, t为元素 , args为可以转化成元素的参数, n为常数, b,e为迭代器, il                为元素值列表. 返回均为p
c.emplace(p, args):
c.insert(p, n, t):
c.insert(p, b, e):
c.insert(p, il):

```

#### 访问元素

front和back成员函数, 返回首元素和尾元素的引用.

访问元素的成员函数(front, back, at, 下标)返回均为引用.

string, vector, deque, array提供了下标操作进行快速访问. 使用下标可能会发生越界, 编译器不会检测是否越界, 这将导致程序错误.

at成员函数, 为了防止发生越界的错误, 可以使用at(index)成员函数访问元素, 此时当越界时将会报out_of_range异常.

#### 删除元素

```C++
forward_list有自己的erase
froward_list不支持pop_back
vector, string不支持pop_front

c.pop_back() 删除尾元素, 返回void
c.pop_front() 删除首元素, 返回void
c.erase(p) p ,b, e均为迭代器, 返回均是删除元素的后一个元素
c.erase(b, e)
c.clear() // 返回void

```

#### forward_list特殊操作

由于forward_list是单向链表, 无法访问当前元素前一个元素, 因此操作较为特殊

```C++
lst.before_begin()返回首元素之前的不存在的元素迭代器(链表头结点, 无实际元素, 仅仅提供入口)
lst.cbefore_begin() 与上面一样,不过返回const_iterator

lst.insert_after(p, t) 在p之后插入元素,  返回插入的最后一个元素的迭代器
lst.insert_after(p, n, t)
lst.insert_after(p, b,e)
lst.insert(p, b, e)
lst.emplace_after(p, args)

lst.erase_after(p)删除p之后的元素, 返回被删元素之后的迭代器
lst.erase_after(b, e) 删除b(不包括)到e, 返回e之后的迭代器.

```

#### 改变容器大小

```C++
c.resize(n) 更改大小, 当n<c.size(), 多余被删除, n>size(), 新加入的部分被默认初始化

c.resize(n, t) 更改大小, 当n<c.size(), 多余被删除, n>size(), 新加入的部分使用t初始化

```

#### 容器操作可能使迭代器失效

这里主要涉及容器内部实现, 以及对应操作的实现, 当容器的存储空间是否被重新分配, 重新分配则会失效.

插入元素时:

当vector和string存储空间被重新分配时, 迭代器,指针, 引用失效.

deque除插入位置为收尾,其它均会失效.

list和forward_list均有效

删除元素时:

被删除的位置均会失效.

list和forward, 剩余部分均有效

deque如果删除位置处于收尾元素之外其它也均会失效, 如果是尾元素,尾后迭代器也会失效.

vector和string被删元素之后会失效.

### string额外的操作

#### 构造string的其他方法

```
string s(cp, n);拷贝cp中前n个作为s

string s(s2, pos2); 从s2下标pos2开始到末尾的元素拷贝初始化s

string s(s2, pos2, len2); 上一个再加上长度限制.

```

其中接受的参数是string或char*(不必以空字符结尾).

s.substr()取子字符串操作, 两种形式

s.substr(pos); 取从pos到维的子字符串

s.substr(pos1, pos2); 取pos1, 到pos2的子字符串[pos1, pos2). pos均为整数, pos最小为0.



#### 改变string的其他方式

在一般删除, 添加, assign, 等操作中使用的迭代器可以完全换成索引, 依然可以对string进行更改

s1.append(args), 在s1后面添加args

s1.replace(pos, length, args), 删除pos为起点长度为length的元素, 用args替换. 

```
string s = "l don't think so!!!";
s.replace(2, 5, "do");
cout<<s<<endl;
output:
I do think so!!!

```

#### string搜索操作

```
s.find(args) 查找s中args第一次出现位置

s.rfind(args) 查找s中最后一次出现args的位置

s.find_first_of(args) 在s中查找args中任意一个字符第一次出现的位置

s.find_last_of(args) 在s中查找args中任意一个字符最后一次出现的位置

s.find_first_not_of(args) 在s中查找第一个不在args中的字符

s.find_last_not_of(args) 在s中查找最后一个不在args中的字符

args参数要求: c是字符, pos是起始搜索的位置(下标), 默认是0, s2为字符串, cp为字符串指针(以空字符结尾),
c, pos 
s2, pos
cp, pos
cp, pos, n: 从s的pos位置查找cp指向的数组的前n个字符, pos和n无默认值.

```

#### 数值转换

```
to_string(val)
stoi(s, p, b)
stol(s, p, b)
stoul(s, p, b)
stoll(s, p, b)
stouul(s, p, b)
stof(s, p)
stod(s, p)
stold(s, p)

```

sto表示string to 后面部分是要转换到的类型.

### vector如何增长

vector和string会分配比需求大的空间作为备用, 当空间不够时再进行扩张, 此时就要移动所以元素了, 会导致迭代器,指针和引用失效. 

容器大小操作:

```C++
shrink_to_fit只适用于vector, string,deque
capacity和reserve只适用vector和string
c.shrink_to_fit() 将capacity减少至size大小
c.capacity()不重新分配空间时, c能容纳多少元素
c.reserve(n) 分配至少可以容纳n个元素的空间

```



### 容器适配器

一个适配器是一种机制, 使得某些事物的行动看起来像另外一件事一样.

所有适配器均支持的操作:

```
size_type: 足以保存当前类型最大对象的大小

value_type: 元素类型

container_type :实现适配器的底层容器类型

A a; 创建空的适配器

A a(c); 创建一个a的适配器, 带有容器c的一个拷贝

关系运算符:
==, !=, >, >=, <, <=

a.empty()

a.size()

swap(a,b)

a.swap(b)

```

创建适配器是可以指定使用何种容器实现

```
     vector<int> W = {2, 5, 7, 5, 3, 10};

​    priority_queue<int, deque<int>> p; // <>中第一个是元素类型, 第二个是容器类型

​    for(int i=0;i<W.size();i++)

​    {

​        p.push(W[i]);

​    }

​    cout<<p.top()<<endl;

```



#### 栈适配器stack

```
栈默认基于deque, 也可以在list或vector上
s.pop() 删除栈顶元素, 返回void
s.push(item) 创建一个元素压入栈
s.emplace(args)
s.top() 返回栈顶元素但不弹出栈.

```

#### 队列适配器(queue/priority_queue)

```
queue默认使用deque实现, 也可以使用list或vector实现, priority_queue默认使用vector, 也可以使用deque
q.pop() 删除queue队首或priority_queue中拥有最高优先级的元素, 无返回
q.front() 返回首元素或尾元素 只适用于queue
q.back() 只适用于queue
q,top() 返回最高优先级元素但不删除, 只适用于priority_queue
q.push()
q.emplace()

```



## 泛型算法

标准库为顺序容器定义了一组泛型算法, 泛型算法不止实验与标准库类型, 还适用于内置的数组类型. 一般这些算法不直接操作容器, 而是遍历两个迭代器指定的一个元素范围. 大多数泛型算法都定义在algorithm头文件中, numeric中也定义了一组数值泛型算法.

算法不会执行容器的操作, 只会运行与迭代器之上, 执行迭代器的操作. 算法永远不会改变底层容器大小, 可以改变容器中保存的元素, 或者移动元素, 不可直接添加删除元素. 个人理解: 泛函算法类似于批处理, 对指定区域元素进行某一相同操作(甚至可以将区域本身也当做一种操作).

### 只读算法

#### find

寻找子元素. find(begin, end, val); 返回指向元素的迭代器.

#### accumulate

序列求和算法. 只能用于可以加和的元素, accumulate(begin, end, val), val为求和的初值, 范围是[begin, end).

#### equal

比较两个序列是否保存相同的值, equal(a1.begin(), a1.end(), a2.begin() ); 三个元素前两个与之前一样, 第三个表示第二个元素的首元素. 这个算法有一个假设, 即a2长度不小于a1.

### 写容器算法

由于算法不能直接更改迭代器底层大小, 因此写操作要求写入的位置本身就具有元素.

#### fill

填充, 语法 fill(a.begin(), a.end(), val); 将指定范围内元素使用val赋值.

#### fill_n

填充, 第二个参数指明长度. fill_n(a.begin, n, val). 以a.begin()为起点的n个元素使用val填充. 要求n为合法数值, 即begin()后有n个元素.

#### back_inserter

向容器中添加插入迭代器. 定义在头文件iterator中.  iterator = back_inserter(a), 向a中尾部添加迭代器iterator. 

```C++
vector<int> vec;
fill_n(back_inserter(a), 10, 0);// 向vec添加10个元素, 并用0赋值.

```

其中fill_n语句执行类似一个批处理命令, 10表示执行10遍, 每次执行back_inserter操作添加迭代器, 而后使用0赋值.

#### copy

拷贝. copy(a1.begin(), a1.end(), a2), 将指定区域元素拷贝到以a2为起点的序列中..

```c++
int a[] = {2, 3, 4, 5};
int b[sizeof(a)/sizeof(*a)];
copy(begin(a), end(a), b);

```

#### replace

替换, replace(begin(a), end(a), val1, val2), 将指定区域中值为val1的变量赋值为val2.

### 重排算法

#### sort

重排, 默认使用"<", 即由大到小, 比较方式可以指定. sort(begin, end, compare); cmpare应该是一个二元谓词(下面详细介绍),

```c++
int a[] = {5, 4, 8, 6, 7};
bool compare(int a, int b)
{
    return a>b;
}
sort(begin(a), end(a), compare);

```

#### unique

重排标准库, 使得不重复的元素出现在容器前面, 返回排序后重复的第一个位置. unique(begin, end); 其中要求重复元素是连续的, 即重复元素是在一起的, 并且, 对于后面的重复元素其值并不是重复元素,而只是站位而已, 其内的元素是不定的.

```c++
int a[] = {5, 7, 8, 9, 1, 5, 7, 11};
sort(begin(a), end(a));
unique(begin(a), end(a));
for(auto i : a)
{
    cout<<i<<" ";
}
OUTPUT:
1 5 7 8 9 11 9 11

```

### 定制操作

#### 向算法传递函数

谓词是一个可调用的表达式, 返回结果是一个能用作条件的值. 标准库算法使用的谓词包含一元谓词有二元谓词, 代表接受的参数个数. 

#### lambda表达式

我们可以向一个算法传递任意类别的可调用对象. lambda表达式也是可调用对象. 一个lambda表达式表示一个可调用的代码单元. 形式:

```
[capture list] (parameter list)->return type {function body};

```

capture list(捕获列表)是一个lambda所在函数中定义的局部变量列表. 捕获列表可以为空, 其他与正常函数没有太大不同, 返回类型必须为尾置返回. 我们可以忽略参数列表和返回类型. 但必须包含捕获列表和函数体.

ex:

```c
auto f = []{return 42;};
//调用
f();

```

lambda忽略括号和参数列表等价于指定一个空参数列表. 忽略返回值类型, lambda会根据函数体推断返回类型. 如果函数体只有一个返回语句, 返回类型通过return语句推断, 否则返回void.

向lambda传递参数:

```c++
[](const string &s1, const string &s2){return s1.siez()<s2.size();};//compare function.

stable_sort(a.begin(), a.end(), [](const string &s1, const string &s2){return s1.siez()<s2.size();}); //sort vector of string

```

使用捕获列表:

```c++
int sz = 42;
[sz](const string &s1){return s1.size()>=sz};

```

值捕获:

值捕获是变量拷贝, 捕获发生在lambda创建时而非调用时. 因此, 之后变量的改变不会影响lambda函数.

```c++
size_t v1 = 42;
auto f = [v1]{return v1;};
v1 = 0;
auto j=f(); //j = 42;

```

引用捕获

引用捕获变量的值是会随着变量本身变化而变化的.

```c++
size_t v1 = 42;
auto f = [&v1]{return v1;};
v1 = 0;
auto j=f(); //j = 0;

```

隐式捕获:

我们可以让编译器根据代码自动选择捕获变量. 这被称为隐式捕获. 隐式捕获要求注明捕获方式是值捕获还是引用捕获, 分别使用"=", 和"&"表示.  同时,我们可以混合使用隐式捕获与显示捕获.但这要求捕获列表第一个必须是"=", 或者"&", 并且之后的显示捕获的方式要与隐式捕获的不同(不然就应该全部使用一种方式啊).

```c++
int a=1, b=2, c=3;
auto f = [&, c](int d){return a+b+c+d;}; //对a,b调用引用的隐式捕获, 对c调用显示的值捕获.

```

可变lambda

此前所述的值捕获是不能更改lambda中参数大小的, 如果想要更改大小, 则应该加上mutable关键字. 此时lambda中值捕获就可以更改大小了.

```C++
int a=42;
auto f = [a]()mutable{return ++a;};
v1 = 0;
int j = f() // j=43, 更改了值捕获中拷贝的大小

```

指定lambda返回类型:

对于只有一个return 的简单函数, 编译器会自己判断返回类型, 而对于较为复杂(多个return)的lambda表达式, 应该指明返回类型.

```c++
auto f = [](int i )->int {if(i<0) return -i; else return i;};

```



#### find_if

查找容器中满足条件的元素.find_if(begin, end, callabe), callabe表示可调用对象.  返回为第一个满足条件的元素的迭代器.

```c++
auto wn = find_if(a.begin(), a,end(), [sz](const string &s){return s.size()>sz;});

```

#### for_each

为容器中满足条件的运算执行操作. for_each(a.begin, a.end, callabe);

```c++
for_each(a.begin(), a.end(), [](const string &s){cout<<s<<" ";});

```

### 参数绑定(bind)

bind函数可以看做一个通用函数适配器(类似于python中装饰器), 它接受一个可调用对象, 生成一个新的可调用对象来"适应"原对象的参数列表. bind被定义在functional中, 一般形式:

```c++
auto newCallable = bind(callable, arg_list);

```

arg_list是一个逗号分割的参数列表, 对应于callable本身的参数. 当我们调用newCallable时, newCallable会调用callable并传递arg_list中的参数. arg_list参数可能包含_n的名字, n是一个整数, 作为占位符, 表示newCallable的参数. 它们占据了传递给newCallable的参数的位置. n表示原来的可调用函数的第n个位置的参数, _1表示原来的callable的第一个参数位置被占据. _n被定义在placeholder命名空间, 使用时要添加using namespace std::placeholder;

```c++
bool com(const string &s, const int &sz)
{
    return s.size()<sz;
}
auto new_com = bind(com, _1, 32);

```

有时我们希望传递的参数是引用类型(比如io无法赋值), 然而bind拷贝其参数,此时我们需要ref或cref函数. 该函数返回一个对象, 包含给定的引用, 次对象可以拷贝.

```
ostream &print(ostream &os, const string &s)
{
    return os<<s<<endl;
}
auto new_print = bind(print, ref(os), _1);

```

## 关联容器

关联容器支持高效的关键字查找与访问. 两个主要的关联容器是set和map(均使用红黑树实现). 关联容器支持一般容器的常规操作.  STL主要定义了八种关联容器. 为set, map, multimap, multiset, 以及对应与前四者的无序版本: unordered_(), map中的元素是关键字-值(key-value)对, pair类型. map被称为关联数组. set是关键字的集合.  关联容器必须定义关键字比较的方式. 为了使用自己定义的操作类型, 我们需要提供比较操作--一种函数指针类型.

### map

```c++
map<string int> word_count;
word_count["ui"] = 1; //当"ui"存在时, 会对其value赋值, 当不存在时, 会先创建"ui"再赋值.

bool compare(const string &. const string &);
map<string, int, bool (*)(const string &, const string&)> word_count(compare); //指定比较类型
==
map<string, int, decltype(compare)*> word_count(compare); //指定比较类型

```

map中存储元素为pair类型. 包含两个共有数据成员first和second分别对应key和value.

使用下标访问map是有一个问题的, 有时我们不知道是否存在要访问的元素, 如果不存在, 则使用下标时会完成创建. 使用at相对更好一些, 当不存在时, at会报错.

```c++
map<string, int> w;
w["ann"];
int j = w["ann"];//j = 0;
w.at("annn"); //throw out of range

```



### set

```c++
set<string> word;

```

### pair类型

pair为一种标准库类型. 定义在utility中.

pair操作:

```c++
pair<T1, T2> p;
pair<T1, T2> p(v1, v2);
pair<T1, T2> p = {v1, v2};
make_pair(v1,v2); //返回一个pair.
p.first;
p.second;
p1 relop p2;

```

### 关联容器操作

```c++
key_type    容器类型的关键字类型
mapped_type 关键字关联的类型, 只适用于map
value_type  对于set, 与可以\_type相同, 对于map为pait<key_type, mapped_type>;

```

```c++
 set<string>::value_type v1; //v1是一个string类型
set<string>::key_type v2; //v2是一个string类型
map<string, int>::value_type v3;//v3是一个<string, int>的pair

```

### 关联容器迭代器

当解关联容器的迭代器时, 得到的是value_type, map中first保存的是const类型, set本身的value_type就是const类型. 即关键字都不能更改.

### 添加元素

```c++
c.insert(v);// v is value_type
c.emplace(args);
c.insert(b, e); // b, e is iterator
c.insert(il); // il is a list of value_type
c.insert(p, v); // p is a iterator which identity the local of begining to search where to insert the v.
c.emplace(p, v);

```

ex:

```
word_count.insert({"word", 1});
word_count.insert(make_pair("ward", 1));
set<int> set2;
vector<int> ivec = {1, 2, 3, 4, 8};
set2.insert(ivec.begin(), ivec.end());
set2.insert({5, 8, 7, 1,3});

```

#### insert的返回值

对于不能包含重复元素的容器, 插入**单一元素**返回为pair类型, 告诉我们是否插入成功. pair的first是迭代器, 指向具有给定关键字的元素, second是一个bool值, 表示是否插入成功.

#### 向multiset和multimap添加元素

对于插入单一元素, 返回是指向插入后的新元素的迭代器.

### 删除元素

```c++
c.erase(k);//k is key
c.erase(p);//p, b, e is iterator 
c.erase(b, e);

```

### 访问元素

```c++
c.find(k); // return a iterator which identify the element whose key is k, if k isn't exist, return c.end();
c.count(k); // return the number of element whose key is k;
c.lower_bound(k); //return a iterator whose key is the first key>=k;
c.upper_bound(k); // return a iterator whose key is the first key>k;
c.equal_range(k); // return a pair of iterator which identify the range of key = k, if k isn't exist, return (c.end(), c.end()).

```



## 动态内存与智能指针

### shared_ptr类

shared_ptr类一种智能指针, 允许多个指针指向同一对象. 智能指针均定义在memory头文件中. 智能指针也是模板, 当我们创建一个智能指针时,必须提供指向的类型. 智能指针能够自动释放内存不用用户自己手动释放. 智能指针可以看做对普通内置指针的封装.

ex:

```c++
shared_ptr<string> p1;
shared_ptr<string> p2; //p1, p2均是智能指针, 访问成员需要->

```

unique_ptr与shared_ptr均支持的操作:

```c++
shared_ptr<T> sp;
unique_ptr<T> up;
p;// 将p作为一个条件判断, 若p指向一个对象则为true.
*p;// 解引用p, 获取指向元素
p->mem; // 等价于(*p).mem;
p.get(); // 返回p中保存指针.
swap(p, q); // 交换p, q指向的指针.
p.swap(q);

```

shared_ptr支持的操作:

```c++
make_shared<T>(args);
shared_ptr<T> p(q); //递增q中计数器.
p = q; // 递减p中计数器, 递增q的计数器. 当p的引用计数变成0, 则将其原本管理的内存释放.
p.unique(); // 如果p.use_count() == 1 retuen true , else return false.
p.use_count(); // 返回与p共享对象的智能指针数量.
ex:
shared_ptr<vector<string>> p = make_shared<vector<string>({"a", "the", "an"});

```

#### shared_ptr的拷贝与赋值

```
auto p = make_shared<int>(42);
auto q(p);

```

每个shared_ptr都有一个关联的计数器, 称为引用计数. 无论我们什么时候拷贝一个shared_ptr对象, 计数器就会增加. 包括, 使用一个shared_ptr初始另一个, 作为参数传递, 作为返回值. 我们给shanre_ptr对象赋予一个新值或者shared_ptr被销毁(离开作用域时), 计数器就会递减. 一旦引用计数对于0, 就会自动释放管理的内存.

ex:

```c++
auto r = make_shared<int>(42);
r = q; //r的引用计数达到0, 原本管理的地址被释放.

```

### 直接管理内存

new分配内存, delete释放new出来的内存. delete只是释放指针指向的内存区域, delete之后原本的指针依然存在, 此时相当于一个空指针.

```c++
vector<string> *pv = new vector<string>();
delete pv;
int *p(new int(42));
使用new动态分配const对象:
const int *pci = new const int(2014);

```

### shared_ptr 与 new结合使用

我们可以使用new返回的指针来初始化智能指针. 必须使用直接初始化方式, 不能将内置指针转换为智能指针.

```c++
shared_ptr<double> p1(new double(14)); // true;

shared_ptr<double> p2 = new double(14);// false, 不能将内置指针转换为智能指针.

shared_ptr<int> clone(int p)
{
    return shared_ptr<int>(new int(p)); //true
    return new int(p); // false;
}

```

定义和改变shared_ptr的方法:

```c++
shared_ptr<T> p(q); // p管理内置指针q所指对象. 要求q所指对象要是new分配的内存(即堆内存).
shared_ptr<T> p(u); // p从unique_ptr接管对象所有权, 将u置空.
shared_ptr<T> p(q, d); //  p管理内置指针q所指对象. 要求q所指对象要是new分配的内存(即堆内存).同时接受可调用对象(函数, 函数指针, lambda)类代替shared_ptr默认的delete.
shared_ptr<T> p(p2, d); // p是p2的拷贝, d与上述d相同.
// p是唯一指向其对象的shared_ptr, reset将会释放此对象.如果传递了可选的参数内置指针q, 会令p指向q, 否则会将p置空. 其中d与上述d一致.
p.reset();
p.reset(q);
p.reset(q,d);

```



### unique_ptr

unique_ptr拥有它所指的对象, 某个时刻只能有一个unique_ptr指向一个给定对象. unique_ptr不支持拷贝即p1(p2), p1 = p2均是非法的. unique_ptr支持的操作:

```c
unique_ptr<T> u1;
unique_ptr<T, D> u2; //D为可调用类型, 用来代替delete.
unique_ptr<T, D> u(d); // D与上面一样, d为D的实例对象
u = nullptr; // 释放u指向的对象, 将u置空.
u.release(); // u放弃对指针的控制权, 返回指针, 并将u置空.由于release并未将指向的空间释放,而是切断了u与对于空间的连续, 因此应该使用返回的指针去对另一个智能指针赋值.
 // 释放u指向的对象. 如果提供了内置指针q, 令u指向这个指针. 
u.reset();
u.reset(q);
u.reset(nullptr);

```

ex:

```c
unique<string> p2(p1.release()); //所有权从p1转换到p2;

p2.release(); //错误, p2不会释放内存, 而且我们丢失了指针.
auto p3 = p2.release(); //正确.

unique_ptr不能赋值, 但有一种情况例外, 就是编译器知道赋值后原来的unique_ptr对象将被销毁:
unique_ptr<int> clone(int a)
{
    return unique_ptr<int>(new int(a));
}

向unique_ptr传递销毁函数:
unique_ptr<objectT, delT> p(new objetT, fcn);

```

### weak_ptr

weak_ptr是一种不控制所指对象生命周期的智能指针, 由一个shared_ptr管理的对象. 将一个weak_ptr绑定到一个shared_ptr上不会增加shared_ptr的引用计数. 当shared_ptr被释放, weak_ptr就会被释放.

操作:

```c
weak_ptr<T> w;
weak_ptr<T> w(sp);
w = p;
w.reset(); //w置空
w.use_count(); // 统计与shared_ptr共享空间的引用计数
w.expired(); // w.use_count == 0 return true, else return false;
w.lock(); // w.expired() = true 返回一个空的shared_ptr, 否则返回一个指向w的shared_ptr;

```

### 动态数组

#### new和数组

```c
int *w = new int[10];
delete [] w;
// 多维数组.w[30][50][10];
int ***w = new int **[30]; //可以理解为*(w[][]) = [30][][];
for(int i=0;i<30;i++)
{
    w[i] = new int *[50]; // w[i]为二维数组指针. 指向 [50][];
    for(int j = 0;j<50;j++)
    {
        w[i][j] = new int [10]; //w[i][j]为一维数组指针, 指向[10];
    }
}
// delete
for(int i = 0;i<30;i++)
{
    for(int j=0;j<50;j++)
    {
        delete [] w[i][j];
    }
    delete [] w[i];
}
delete [] w;

```

注意:当用new分配数组时, 我们并未得到一个数组类型对象, 而是得到了一个数组元素类型指针. 由于得到的不是数组类型, 因此不能使用begin和end函数. 

### allocator类

allocator将内存分配与对象构造分离开. 其分配的内存是原始的, 未构造的. 操作:

```c++
allocator<T> a; // 定义一个名为a的allocator对象. 可以为类型为T的对象分配内存.
a.allocate(n); // 分配内存, 大小为n个T的大小.
a.deallocate(p, n); // 释放从T* 指针p开始的内存, 保存了n个对象. p是allocate返回的地址. n是创建时大小. 且在调用之前, 必须要对每一个对象调用destory
a.construct(p, args); // 使用args对p指向的内存构造一个T对象.
a.destroy(p); // 销毁p所指向的对象. 并不释放内存.

```



## 标准库特殊设施

### tuple类型

tuple是特殊类型模板, tuple类型的成员可以不相同, 可以有任意数量的成员, 每个确定的tuple类型的成员数目是固定的. tuple支持的操作:

```c++
tuple<T1, T2, ..., Tn> t;
tuple<T1, T2, ..., Tn>t(v1, v2, ..., vn);
make_tuple<v1, v2, ..., vn>;
t1 == t2;
t1!=t2;
t1 relop t2;
get<i>(t); //返回t中第i个数据成员的引用.
tuple_size<tupletype>::value; // 类模板, 通过tuple类型初始化. 表示tuple类型中元素数量.
tuple_element<i, tupleType>::type; // tuple中第i个元素类型.
```

tuple常被用来函数返回多个值.



# 模板与泛型编程

模板是泛型编程的基础, 模板是一个创建类或函数的公式.

## 函数模板

定义通用函数模板而不用为每个类型定义一个新函数.

```c++
template <typename T>
int compre(const T& v1, const T& v2)
{
    if(v1<v2) return -1;
    if(v1>v2) return 1;
    return 0;
}

compare(4, 5); // 正确, T为int
compare(4, 0.5);//错误, T不能有两种类型.
compare<double>(4, 0.5); //正确, 显示表明使用double类型实例, 4被认为为double

```

在使用函数模板时, 编译器会根据输入参数进行选择相应的函数实例.  也可以显示的指明使用哪个模板实例.

类型参数前必须使用class或者typename:

```c++
template <class T, typename U> //true;
template <class T,U> //false;

```

## 非类型模板参数

一个非类型参数表示一个值而不是一个类型. 通过一个特定的类型名而非关键字class或typename来指定非类型参数. 当模板被实例化时, 非类型参数被一个用户提供的或者编译器推断的值代替.

ex1:

```c++
template <int n, int m>
int compare_length(const char (&p1)[n], const char (&p2)[m])
{
    cout<<n<<" "<<m<<endl;
    if(n>m) return -1;
    if(n<m) return 1;
    return 0;
}

compare_length("qwer", "qwert");// n=5, m=6, 编译器会在字符串常量后添加空字符'\0'作为结尾.

```

ex2:

```c++
template <int m, int n>
int compare_lenth(const char (&p1)[n])
{
    cout<<n<<" "<<m<<endl;
    if(n>m) return -1;
    if(n<m) return 1;
    return 0;
}

compare_lenth<8>("hvgfcfg"); //对m赋值为8.

int p = 8;
compare_lenth<p>("hvgfcfg"); //错误, 非类型整型参数实参必须是一个常量.

```

一个非类型参数可以是一个整型, 或者是一个对象或函数类型的指针或(左值)引用. 绑定到非类型整型参数实参必须是一个常量.

## 模板编译

一般我们将类定义和申明放在头文件中, 而普通函数和成员函数的定义放在源文件中. 但对于模板则不同, 为了生成一个实例化版本, 编译器需要掌握函数模板或类模板定义. 因此<font color=red> 模板头文件即包含申明也包含定义</font>

## 类模板

编译器不能为类模板推断参数类型. 使用类模板, 我们必须在模板名后的尖括号中提供额外信息. 

```c++
template<class T> class class_name
{
    /* .....*/
};

class_name<typename> args;

```

### 类模板与友元

当在模板类中申明一个友元函数, 则该友元函数可以访问所以类模板的实例化类. 

在类模板中申明友元类时有两种形式, 一种是将特定的模板实例申明为友元, 另一种是将通用模板类申明为友元.

ex:

```c++
template <class T> class1;
template <class T> class2;
template <class U> class3
{
friend class class1<C>; // 特定C实例化作为友元
template <class T> friend class class2; // 将所以class2的模板实例均作为友元类.
};

```

### 类模板的static成员

模板类的每一个实例均存在一套static成员.

### 模板默认实参与类模板

无论何时使用类模板都要使用<>, 但有时我们可以不用指定使用类型, 而是使用默认类型(如果有):

ex:

```c++
template <class T = int> class_name;
class_name<> w; // usw int

```

### 类模板的成员模板

类模板的成员函数依然可以是模板函数.

ex:

```c++
template <class T> class class_name
{
    template <typename It> class_name(It b, It e);
}

template<class T> //类的类型参数
template<class It> // 成员函数的类型参数
class_name::class_name(It a, It, b)
{
    /*'''''*/
}

```

## 控制实例化

当模板被使用时才会被实例化. 当多个独立编译的源文件都使用了相同的模板, 并提供了相同的模板参数时, 每个文件就会有该模板的实例. 这会导致花销的增加. 通过显示实例化来避免这种开销.

形式:

```c++
extern template declaration; //实例化申明
template declaration; //实例化定义

```

当编译器遇到extern模板申明时, 它不会在本文件生成实例化代码. extern申明必须在使用次模板实例化之前. 

```c++
extern template int compare(const int&, const int&);
extern template class Blob<string>; //使用string实例化Blob模板.

```

## 模板实参推断

顶层const无论是在形参中还是实参中都会被忽略. 这是由于\<class T>中, 当使用const int作为参数时, T会被推断为const int, 当使用int作为参数时, T被推断为int. 

参数调用时不会进行参数转换. 如算数转换, 派生类向基类的转换, 以及用户自定义的转换都不会在模板参数中出现.

### 函数模板显式实参

在某些情况下, 编译器无法推断模板实参类型. 这时我们需要指定显式模板实参. 

ex:

```c++
template<class T1, class T2, class T3>
T1 sum(T2, T3);

int a;
double b;
sum<int> sum(a, b); //此时指定返回类型为int, T2, T3根据参数推断.

```

### 尾置返回类型与类型转换

在显式的实参中存在一个问题, 当我们知道返回类型时, 我们可以利用显式实参进行模板实例化, 但有时我们不知道要返回的类型时, 这就不好使了. 例如, 我们输入是class It, 返回值可能是&It, 此时, 当It是int或double或者其他类型时, 返回值是不同的, 同时我们在调用该模板时可能不清楚真正使用的是哪个It, 此时, 就无法使用显式实参了. 为解决此问题, c++有两种策略, 尾置返回类型和类型转换.

使用尾置返回类型时, 可以使用decltype()进行返回类型的确定, 例如输入迭代器, 返回对应元素引用.

ex:

```c++
template <class It>
auto fcn(It beg, It end)->decltype(*beg) // 返回元素引用, 具体参见decltype
{
    return *beg;
}

```

为了获得元素类型, 我们可以使用标准库中的类型转换模板. 这些模板定义在type_traits头文件中.

常用类型模板为remove_reference

<table>
    <tr>
        <th>对Mod &lt T &gt, 其中Mod为:</th>
        <th>若T为:</th>
        <th>则Mod&lt T &gt::type为:</th>
    </tr>
    <tr>
        <th>remove_reference</th>
        <th>X& or X&&</th>
        <th>X</th>
    </tr>
    <tr>
        <th>remove_reference</th>
        <th>否则</th>
        <th>T</th>
    </tr>
</table>

使用实例:

```c++
template<class It>
auto fcn2(It beg, It end)-> typename remove_reference<decltype(*beg)>::type //
{
    return *beg;
}

```

上述代码中, 当使用迭代器进行实例化时, decltype(*beg)是对应参数的引用, 即X&, 根据上述表格, remove_reference(X&)::type为X, 因此该函数返回为对于元素.

### 模板实参推断与引用

当模板参数为T&&即右值引用时, 比较特殊. 在此c++定义了两个例外, 其均是std::move()的基础.

#### 引用的引用(& &)

我们不能定义一个引用的引用, 但是, 通过类型别名或者模板类型参数间接定义是可以的.

类型别名使用引用的引用:

```c++
using int_yy = int&;
int w = 5;
int_yy &p1 = w; //正确, 使用类型别名可以定义引用的引用
int & &p2 = w; // 错误, 不能直接定义引用的引用.

```

这在模板引用中,  发生情况为, 当定义的模板为T&&时, 即右值引用时, 当我们传递参数为左值时, T就会被推断为对应类型的引用, 在加上之后的引用, 就变成了右值引用的引用. ex:

```c++
template <class It>
bool compare(It&& a, It&& b);
double w1 = 15, w2 = 50;
compare(w1, w2); // 此时, It被推断为double&, 所以传递参数为double& &&;

```

#### 引用折叠

当出现引用的引用时, 为了解决这一问题, c++定义了引用折叠. 对于给定类型X;

X&, X& &&, X&& &均被折叠成X&,  X&& &&被折叠次X&&.

所以上述double& &&被折叠成double&.

由于引用折叠, 有时会出现比较严重的错误, 当使用T&&, 我们显然是希望使用临时变量, 但当我们传递左值时, 会被折叠错误引用, 此时就会对原本的左值进行更改, 但这往往是我们所不希望看到的. 为了解决这个问题, 我们应该定义const的重载函数.ex:

```c++
template<class It> void f(T&&);
template<class It> void f(const T&);

```

此时参数为左值时会调用下面的函数. 参数为右值引用时调用上面函数.

### 理解std::move()的实现

```c++
template <class T>
typename remove_reference<T>::type&& move(T&& t)
{
    return static_cast<typename remove_reference<T>::type&&>(t);
}

```

以int为例解释, 当传入为右值时, T为int,  remove_reference<T>::type&&为int&&, 因此, 返回值return为int&&, 因此类型转换什么也不做.

当传入的是右值时, 根据第一条例外, T为int&, remove_reference<T>::type&&为int&&, static_cast将t变为右值, 最终返回值依然是右值.

# 编译调试

编译:

```
$ cc project.cpp

```

$为系统提示符, cc为编译器程序名字(g++), project.cpp为要编译的程序. unix下默认生成a.out可执行文件, windows下生成project.exe可执行文件. 也可以指定生成可执行文件名字. 

unix下实例

```
$ g++ -o demo project.cpp

```

-o demo是编译器参数, 指定生成可执行文件名字.

## 分离式编译

分离式编译允许将程序分割到几个文件中, 每个文件分别编译.

例如存在三个文件: funcation.h, funcation.cpp, main.cpp.分别包含函数申明, 函数定义, 函数使用.编译如下:

```
$ cc main.cpp funcation.cpp -o main
==
$ cc funcation.cpp main.cpp -o main

```

## 调试帮助

程序可以包含一些用于调试的代码, 这些代码只在开发程序时使用, 当程序编写完成准备发布时, 要先屏蔽掉调试代码. 主要用到两项预处理功能: assert和NDEBUG.

### assert预处理宏

预处理宏: 预处理变量, 使用一个表达式作为条件: 

```C++
assert(expr);

```

首先对expr求值, 如果expr为, assert输出信息并终止程序, 否则assert什么也不做.

assert定义在cassert文件中.预处理名字由预处理而非编译器管理.

当assert的表达式expr为假时将执行#ifndef NDEBUG与#endif之间的代码.

### NDEBUG预处理变量

assert的行为依赖NDEBUG预处理变量的状态. 如果定义了NDEBUG则assrt什么也不做(等于没有), 默认没有定义NDEBUG, 此时assert将执行运行时检查.

我们使用一个#define语句定义NDEBUG, 从而关闭调试状态. 同时, 编译器都提供一个cmd选项使我们可以定义预处理变量:

```
$ cc -D NDEBUG main.cpp  #use /D with the windows

```

这条指令表示在main.cpp文件的一开始写#define NDEBUG

### 举例

```C++
#include<iostream>
#include<cassert>
using namespace std;
int main(int argc, char **argv)
{
    for(int i=0;i<argc;i++)
        cout<<argv[i]<<endl;
    assert(argc>1);
#ifndef NDEBUG
    cout<<"未定义NDUBGET"<<endl;
#endif
    return 0;
}
编译: g++ main.cpp -o main
运行:./main skjj suu ii
输出:
./main
skjj
suu
ii
未定义NDUBGET


编译:g++ -D NDEBUG main.cpp -o main
运行:./main skjj suu ii
输出:
./main
skjj
suu
ii

```

## unix下使用g++编译过程及实例

### g++编译器选项解读

1. 基本选项

```
-E 是只进行预处理选项，不进行编译、汇编、以及连接

-S 编译后停止，不进行汇编和连接

-c 编译或会汇编文件，但不进行连接

-o file 指定输出文件名

```

2. 警告选项

```
-Wall 启用所有警告信息

-Werror 在发生警告时取消编译操作，即将警报看作是错误

-w 禁用所有警告信息

```

3. 优化选项

```
-O0：不进行优化处理

-O或-O1：进行基本的优化，这些优化在大所属情况下都会使程序执行的更快

-O2：除了完成-O1级别的优化外，还需要一些其他的调整工作，如处理器指令调度等，只是GNU发布软件的默认优化

-O3：除了完成-O2级别的优化外，还进行循环的展开（这往往会提高执行速度）以及其他的一些预处理器相关的优化工作。

-Os：生成最小的可执行文件，主要用在嵌入式领域。

```

4. 连接器选项

```
-Idirectory 向G++的头文件搜索路径中添加新的目录

-Ldirectory 向G++的库文件搜索路径中添加一个行的目录

-llibrary 提示连接程序在创建可执行文件时包含指定的库文件

-static 强制使用静态库

-shared 强制使用共享库

```

5. 其他选项

```
-xlanguage 指定输入文件的编程语言

-v 显示编译器的版本号

-g 获得有关调试程序的详细信息

-ansi 支持符合ansi彼岸准的c程序

```

实例:

包含三个文件, add.h, add.cpp, test.cpp.

### 第一步: 预处理

预处理阶段是进行处理代码中的宏和include指令，并作语法检查。这一过程的命令为：g++ -E test.cpp -o test.i 执行这一步生成了一个test.i文件(预处理文件)

### 第二步:汇编程序生成汇编码

将生成的预处理文件进行汇编. 命令为:　g++ -S test.i  -o test.s

生成汇编程序test.s

### 第三步:由汇编程序转换为中间目标文件

这一步是将汇编的代码进一步进行处理，每一个源程序都会生成相应的目标文件，是以.o为扩展名的文件

命令为: g++ -c test.s -o test.o

​              g++ -c add.cpp -o add.o

### 第四步: 连接目标文件，生成最终目标文件(可执行文件或静态库或动态库)

将上一步生成的中间文件进行链接, 生成最终的可执行文件.

```
g++ test.o add.o -o test

==

g++ add.o test.o -o test

```

这里不要求先后顺序

上述四步可以简化为一步: g++ test.cpp add.cpp -o test 同样不强调先后.

连接opencv库 \`pkg-config --cflags --libs opencv`

## 库

库就是一组已经写好了的函数和变量、是经过编译了的代码，为了提高开发的效率和运行的效率而设计的。库可以分为静态库和动态库（共享库）两类，在linux系统中静态库的扩展名为.a，动态库的扩展名是.so

**静态库**是在每个程序进行链接的时候将库在目标程序中进行一次拷贝，当目标程序生成的时候，程序可以脱离库文件单独运行，换言之原来的文件即使删除程序还是会正常工作。

**共享库**可以被多个应用程序共享，实在程序运行的时候进行动态的加载，因此对于每个应用程序来说，即使不再使用某个共享库，也不应该将其删除，因为其他的引用程序可能需要这个库。

生成静态库的过程是先将每个每个原文件进行编译生成中间目标文件，然后利用打包程序，将程序进行一次打包，最后生成静态库文件.

## CMake使用

使用流程:

1. 编写CMake配置文件CMakeLists.txt
2. 执行CMake PATH. PATH为含有CMakeLists.txt文件的路径, 生成makefile
3. 使用make进行编译

### 实例一: 单文件

只含有一个test.cpp.

编写CMakeLists:

```cmake
`# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)
# 项目信息
project (Demo1)
# 指定生成目标
add_executable(Demo main.cc)`
```

### 实例二: 同一目录, 多个文件

test.cpp, add.h, add.cpp

```
./Demo2

​    |

​    +--- test.cpp

​    |

​    +--- add.cpp

​    |

​    +--- add.h



```

CMakeList.txt:

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo2)

# 指定生成目标
add_executable(Demo test.cpp add.cpp)

```

上述中add_executable(Demo test.cpp add.cpp)是将要编译的文件全部加入,但文件很多时将十分不便,因此使用aux_source_directory(\<dir> \<variable>)代替, 表示将目录dir下的所有源文件均添加进去.

于是更改为

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)
# 项目信息
project (Demo2)
# 查找当前目录下的所有源文件,并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)
# 指定生成目标
add_executable(Demo ${DIR_SRCS})

```

### 实例三: 多个目录, 多个文件

```
./Demo3

​    |

​    +--- test.cpp

​    |

​    +--- add/

​          |

​          +--- add.h

​          |

​          +--- add.cpp

```

需要分别在项目根目录 Demo3 和add目录里各编写一个 CMakeLists.txt 文件.为了方便，我们可以先将 math 目录里的文件编译成静态库再由 main 函数调用。

根目录中的 CMakeLists.txt:

```cmake
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)
# 项目信息
project (Demo3)
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)
# 添加 math 子目录
add_subdirectory(add)
# 指定生成目标 
add_executable(Demo main)
# 添加链接库
target_link_libraries(Demo add)

```

使用add_subdirectory(add)表明项目包含一个子目录 add, 这样 add 目录下的 CMakeLists.txt 文件和源代码也会被处理. target_link_libraries(Demo add)指明可执行文件 main 需要连接一个名为 add 的链接库.

子目录中的 CMakeLists.txt:

```cmake
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_LIB_SRCS 变量
aux_source_directory(. DIR_LIB_SRCS)
# 生成链接库
add_library (add ${DIR_LIB_SRCS})

```

add_library (add ${DIR_LIB_SRCS}) 将 add 目录中的源文件编译为静态链接库。

## 使用VS code进行C++编码

1. 创建文件,并编写好代码.
2. 编写CMakeLists.txt文件
3. 调用命令台工具(Ctrl + Shift + P),选择Cmake Config. (每当文件结构发生变换就应当执行该步骤.)
4. 运行build(点击build按钮)
5. 测试成功 转到build目录.运行生成的目标可执行文件.
6. 创建tasks.json(Ctrl + Shift + P 选择Tassk:configure tasks)更改"command"参数为"build/可执行文件" 如"build/example-app"
7. 在Terminal中选择Run Task,选择tasks.json中"label"对应的参数, 再选择第一个
8. F5 创建launch.json, 将"program"参数改为"${command:cmake.launchTargetPath}" 即设置为Cmake插件的debug模式
9. 再次点击build(Debug模式的Cmake)
10. 点击debug按钮即可debug