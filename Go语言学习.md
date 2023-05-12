---
title: Go语言学习
date: 2021-09-30 15:09:42
tags: Go
categories: 语言
mathjax:
    true
description: go语言学习记录
---

# 入门

## 例子

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, 世界")
}
```

执行程序

```
go run helloworld.go
```

编译重新

```
go build helloworld.go
```

会生成编译产物，`helloworld`，执行编译产物

```
./helloworld
```

下载线上代码：

```
go get gopl.io/ch1/helloworld
```

要求环境中存在版本管理工具。下载的代码会放到目录`$GOPATH/src/gopl.io/chl/helloworld`中。`$GOPATH`为环境变量。

使用

```
export GOPATH=$HOME/gobook
```

设置环境变量。

设置国内镜像：

```
# 配置 GOPROXY 环境变量，以下三选一

# 1. 七牛 CDN
go env -w  GOPROXY=https://goproxy.cn,direct

# 2. 阿里云
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct

# 3. 官方
go env -w  GOPROXY=https://goproxy.io,direct

```



## 简述

Go语言的代码通过包（packet）来组织，包类似于其他语言的库或模块。一个包由位于单个目录下的一个或多个.go源代码文件组成, 目录定义包的作用域。

每个源文件都以一条`package`声明语句开始，这个例子里就是`package main`, 表示该文件属于哪个包，紧跟着一系列导入（import）的包，之后是存储在这个文件里的程序语句。

fmt包包含格式化输入输出、接受输入的函数。

`import`声明必须跟在文件的`package`声明之后。随后，则是组成程序的函数、变量、常量、类型的声明语句（分别由关键字`func`, `var`, `const`, `type`定义）

一个函数的声明由`func`关键字、函数名、参数列表、返回值列表（这个例子里的`main`函数参数列表和返回值都是空的）以及包含在大括号里的函数体组成。

Go语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句。实际上，编译器会主动把特定符号后的换行符转换为分号, 因此换行符添加的位置会影响Go代码的正确解析。举个例子, 函数的左括号`{`必须和`func`函数声明在同一行上, 且位于末尾，不能独占一行，而在表达式`x + y`中，可在`+`后换行，不能在`+`前换行（译注：以+结尾的话不会被插入分号分隔符，但是以x结尾的话则会被分号分隔符，从而导致编译错误）。

`gofmt`工具把代码格式化为标准格式。

## 命令行参数

`os`包以跨平台的方式，提供了一些与操作系统交互的函数和变量。程序的命令行参数可从os包的Args变量获取；os包外部使用os.Args访问该变量。

os.Args变量是一个字符串（string）的*切片*（slice）,与python切片类似。序列的元素数目为len(s)。os.Args的第一个元素，os.Args[0], 是命令本身的名字；其它的元素则是程序启动时传给它的参数。

实现echo

```
package main

import (
    "fmt"
    "os"
)

func main() {
    var s, sep string
    for i := 1; i < len(os.Args); i++ {
        s += sep + os.Args[i]
        sep = " "
    }
    fmt.Println(s)
}
```

注释使用`//`开头。

for循环只有一种形式：

```
for initialization; condition; post {
    // zero or more statements
}
```

for循环的这三个部分每个都可以省略，如果省略`initialization`和`post`，分号也可以省略：

```
for condition {
    // ...
}
```

如果连condition也省略，则为无限循环。

`for`循环的另一种形式, 在某种数据类型的区间（range）上遍历，如字符串或切片。`echo`的第二版本展示了这种形式：

```
package main

import (
    "fmt"
    "os"
)

func main() {
    s, sep := "", ""
    for _, arg := range os.Args[1:] {
        s += sep + arg
        sep = " "
    }
    fmt.Println(s)
}

```

每次循环迭代，`range`产生一对值；索引以及在该索引处的元素值。该处不需要索引，但Go语言不允许使用无用的局部变量（local variables），这种情况的解决方法是用`空标识符`（blank identifier），即`_`。

声明变量可以使用下面几种方式

```
s := ""
var s string
var s = ""
var s string = ""
```

第一种形式，是一条短变量声明，最简洁，但只能用在函数内部，而不能用于包变量。第二种形式依赖于字符串的默认初始化零值机制，被初始化为""。实践中一般使用前两种形式中的某个，初始值重要的话就显式地指定变量的类型，否则使用隐式初始化。

如果连接涉及的数据量很大，这种方式代价高昂。一种简单且高效的解决方案是使用`strings`包的`Join`函数：

```
func main() {
    fmt.Println(strings.Join(os.Args[1:], " "))
}
```



## 查找重复的行

```go
package main

import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    counts := make(map[string]int)
    input := bufio.NewScanner(os.Stdin)
    for input.Scan() {
        counts[input.Text()]++
    }
    // NOTE: ignoring potential errors from input.Err()
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}
```

**map**存储了键/值（key/value）的集合，对集合元素，提供常数时间的存、取或测试操作。map中顺序是随机的。

`bufio`包，它使处理输入和输出方便又高效。`Scanner`类型是该包最有用的特性之一，它读取输入并将其拆成行或单词；通常是处理行形式的输入最简单的方法。

```
input := bufio.NewScanner(os.Stdin)
```

该变量从程序的标准输入中读取内容。每次调用`input.Scan()`，即读入下一行，并移除行末的换行符；读取的内容可以调用`input.Text()`得到。`Scan`函数在读到一行时返回`true`，不再有输入时返回`false`。

### 格式化输出

`fmt.Printf`函数对一些表达式产生格式化输出。`Printf`有一大堆这种转换，Go程序员称之为*动词（verb）*

```
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

Printf函数的%b参数打印二进制格式的数字；其中%08b中08表示打印至少8个字符宽度，不足的前缀部分用0填充。通常Printf格式化字符串包含多个%参数时将会包含对应相同数量的额外操作数，但是%之后的`[1]`副词告诉Printf函数再次使用第一个操作数。第二，%后的`#`副词告诉Printf在用%o、%x或%X输出时生成0、0x或0X前缀。

从文件中获取输入：

```go
import (
    "bufio"
    "fmt"
    "os"
)

func main() {
    counts := make(map[string]int)
    files := os.Args[1:]
    if len(files) == 0 {
        countLines(os.Stdin, counts)
    } else {
        for _, arg := range files {
            f, err := os.Open(arg)
            if err != nil {
                fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
                continue
            }
            countLines(f, counts)
            f.Close()
        }
    }
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}

func countLines(f *os.File, counts map[string]int) {
    input := bufio.NewScanner(f)
    for input.Scan() {
        counts[input.Text()]++
    }
    // NOTE: ignoring potential errors from input.Err()
}
```

`os.Open`函数返回两个值。第一个值是被打开的文件(`*os.File`），其后被`Scanner`读取。`os.Open`返回的第二个值是内置`error`类型的值。如果`err`等于内置值`nil`（译注：相当于其它语言里的NULL），那么文件被成功打开。

`map`是一个由`make`函数创建的数据结构的引用。`map`作为参数传递给某函数时，该函数接收这个引用的一份拷贝（copy，或译为副本），被调用函数对`map`底层数据结构的任何修改，调用者函数都可以通过持有的`map`引用看到。在我们的例子中，`countLines`函数向`counts`插入的值，也会被`main`函数看到。（译注：类似于C++里的引用传递，实际上指针是另一个指针了，但内部存的值指向同一块内存）。

可以使用`io/ioutil`包中的ReadFile函数来直接读取整个文件。

```
package main

import (
    "fmt"
    "io/ioutil"
    "os"
    "strings"
)

func main() {
    counts := make(map[string]int)
    for _, filename := range os.Args[1:] {
        data, err := ioutil.ReadFile(filename)
        if err != nil {
            fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
            continue
        }
        for _, line := range strings.Split(string(data), "\n") {
            counts[line]++
        }
    }
    for line, n := range counts {
        if n > 1 {
            fmt.Printf("%d\t%s\n", n, line)
        }
    }
}
```

## 并发获取多个URL

Go语言里的goroutine和channel来实现对并发编程的支持。

```go
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"os"
	"time"
)

func main() {
	start := time.Now()
	ch := make(chan string)
	for _, url := range os.Args[1:] {
		go fetch(url, ch)
	}
	for range os.Args[1:] {
		fmt.Println(<-ch)
	}
	fmt.Printf("$.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
	start := time.Now()
	resp, err := http.Get(url)
	if err != nil {
		ch <- fmt.Sprint(err)
		return
	}

	nbytes, err := io.Copy(ioutil.Discard, resp.Body)
	resp.Body.Close()
	if err != nil {
		ch <- fmt.Sprintf("while reading %s: %v", url, err)
	}
	secs := time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}

```

goroutine是一种函数的并发执行方式，而channel是用来在goroutine之间进行参数传递。main函数本身也运行在一个goroutine中，而go function则表示创建一个新的goroutine，并在这个新的goroutine中执行这个函数。

main函数中用make函数创建了一个传递string类型参数的channel，对每一个命令行参数，我们都用go这个关键字来创建一个goroutine。

每当请求返回内容时，fetch函数都会往ch这个channel里写入一个字符串，由main函数里的第二个for循环来处理并打印channel里的这个字符串。

当一个goroutine尝试在一个channel上做send或者receive操作时，这个goroutine会阻塞在调用处，直到另一个goroutine从这个channel里接收或者写入值，这样两个goroutine才会继续执行channel操作之后的逻辑。类似python的线程join函数。



# 工具

可以使用go doc 指令来查看阅读标准库的文档，例如

```
go doc http.ListenAndServe
```

其展示的是每一个函数之前写一个说明函数行为的注释。

搭建本地环境https://xmanyou.com/mac-vscode-go-environment/

go导入自己的包 https://www.jianshu.com/p/4fba6ce388b2

# 程序结构

## 名字

如果一个名字是在函数内部定义，那么它就只在函数内部有效。如果是在函数外部定义，那么将在当前包的所有文件中都可以访问。名字的开头字母的大小写决定了名字在包外的可见性。如果一个名字是大写字母开头的（译注：必须是在函数外部定义的包级名字；包级函数名本身也是包级名字），那么它将是导出的，也就是说可以被外部的包访问，例如fmt包的Printf函数就是导出的，可以在fmt包外部访问。包本身的名字一般总是用小写字母。

## 声明

声明定义了程序各种实体对象以及部分或全部的属性。go语言主要有四种类型的声明语句：var、const、type和func，分别对应变量、常量、类型和函数实体。

一个go程序对应一个或多个以.go为后缀的源文件。每个源文件以包的声明语句开始，说明该源文件属于哪个包。包语句后面是import语句，用于导入其他包，然后是包级别的类型、变量、常量、函数的声明语句。包级别的各种类型的声明语句的顺序没有关系（而函数内部的名字必须先声明再使用）。

例如

```
package main

import "fmt"

var secondValue = firstValue * 2.0
const firstValue = 3.5

func main() {
	fmt.Printf("secondValue is %g\n", secondValue)
	var mSecondValue = mFirstValue * 2.0
	var mFirstValue = 3.5
	fmt.Printf("mSecondValue is %g\n", mSecondValue)
}
```

此时程序不能正常执行，更改main函数内变量顺序即可正常。

注意：在包一级声明语句声明的名字可在整个包对应的每个源文件中访问，而不是仅仅在其声明语句所在的源文件中访问。相比之下，局部声明的名字就只能在函数内部很小的范围被访问。

一个函数的声明由一个函数名字、参数列表（由函数的调用者提供参数变量的具体值）、一个可选的返回值列表和包含函数定义的函数体组成。如果函数没有返回值，那么返回值列表是省略的。执行函数从函数的第一个语句开始，依次顺序执行直到遇到return返回语句，如果没有返回语句则是执行到函数末尾，然后返回到函数调用者。

一个完整的函数例子

```
func fToC(f float64) float64 {
	return (f-32) * 5 / 6
}
```

## 变量

var语句用来创建指定类型的变量，并设置初始值：

```
var 变量名 类型 = 表达式
```

其中类型和`=表达式`可以忽略一个，如果忽略类型，则会根据表达式推倒类型，如果省略表达式，则会使用默认初始值对变量初始化。其中接口或引用类型（包括slice、指针、map、chan和函数）变量对应的零值是nil。数组或结构体等聚合类型对应的零值是每个元素或字段都是对应该类型的零值。

**go语言不存在未初始化的变量。**

也可以在一个声明语句中声明多个变量，在缺省每个变量的类型，也可以声明多个类型不同的变量：

```
var i, j, k int; // true

var i, j, k int = 1,2,3 // true

var i, j, k = 1, 2.3, "test" // true

var i, j, k int float64, string = 1, 2.3, "test" // false.只有在缺省类型时，才可以声明多个不同类型变量。
```

在包级别声明的变量会在main入口函数执行前完成初始化，局部变量将在声明语句被执行到的时候完成初始化。

### 简短变量声明

**函数内部**，可以使用简短变量声明，其语法为

```
名字 := 表达式
```

变量类型将自动推导。例如

```
i := "test"
```

简明变量声明也可以声明和初始化一组变量：

```
i, j := 1,2
```

注意`:=`是变量声明语句，不是变量赋值操作。例如

```
i, j = j, i
```

这是一个赋值语句，是交换变量i，j的值。

对于简短变量声明来说，声明的一组名字可能不全都是进行声明的，对于同级作用域下有之前存在的名字，则对于该名字执行的是赋值语句。但对于简短声明语句，必须至少有声明一个新的变量，例如

```
var i, j int = 1, 2
i, j := 2,3 // false，没有新的变量
i, j, k := 2,3,4 //true， k是新声明的变量
```



### 指针

与C++类似，指针指向变量的地址。`&`为取指符，`*`表示直接在地址上操作变量值。int指针对应的类型是`*int`，其他也是如此。

例如

```
var str string = "test"
var p *string = &str
*p = "hello"
```

go中string类型是只读的，不可改变其里面的字符值，但可以改变整个值。

任何类型的指针零值均为nil。

与C++不同，返回局部变量的指针是安全的，例如：

```go
func main() {

	fmt.Printf("%t\n", returnValue() == returnValue()) // false
}


func returnValue() *int {
	var i int = 3
	return &i
}
```

这里我理解应该是使用了C++中智能指针的实现逻辑。

函数生成的i值，每次都会重新分配地址，因此比较值时，不相等。

指针是实现标注库中flag包的关键技术，其使用命令行参数来设置对应变量的值，而这些对应命令行标志参数的变量会零散分布在这个程序中。例如如下重新，实现的echo，包含两个可选参数：-n 用于忽略行尾换行符， -s sep用于指定分隔符（默认为空格），代码

```
import (
	"flag"
	"fmt"
	"strings"
)

// flag参数分别是，字符串：表示命令行指定的参数， 默认值， 和参数描述
var n = flag.Bool("n", false, "忽略行尾换行符") // 返回是*bool,表示如果出现了该参数，则使用默认值，否则是非默认值，
var seq = flag.String("seq", " ", "分割符") // 返回*string， 如果未出现该参数，*string值为" ",否则，使用指定值，最后一个参数是版本描述

func main() {
	flag.Parse() // 解析命令行输入，并更新每个标志参数对应变量的值,使用flag前必须先执行该函数
	var str = strings.Join(flag.Args(), *seq)
	fmt.Print(str)
	if !*n {
		fmt.Println()
	}

}
```

执行

```
$ ./test2_2 a b c
a b c
$ ./test2_2 -n a b c
a b c$ ./test2_2 -h
Usage of ./test2_2:
  -n    忽略行尾换行符
  -seq string
        分割符 (default " ")
$ ./test2_2 -seq / a b c
a/b/c
$
```

当程序运行时，必须在使用标志参数对应的变量之前先调用flag.Parse函数，用于更新每个标志参数对应变量的值（之前是默认值）。对于非标志参数的普通命令行参数可以通过调用flag.Args()函数来访问，返回值对应一个字符串类型的slice。如果在flag.Parse函数解析命令行参数时遇到错误，默认将打印相关的提示信息，然后调用os.Exit(2)终止程序。

### new函数

new函数用于创建指针(也是智能指针)

```
var name *T = new(T)
```

### 变量的生命周期

对于在包一级声明的变量来说，它们的生命周期和整个程序的运行周期是一致的。而相比之下，局部变量的生命周期则是动态的：每次从创建一个新变量的声明语句开始，直到该变量不再被引用为止，然后变量的存储空间可能被回收。即对于在函数中声明的变量，并函数完成后，继续引用其地址，其声明周期则不单单是在函数内，对于函数内声明的变量，并且未在函数后继续使用其引用，则其生命周期为函数内部。例如

```
var global *int
func returnValue() *int {
	var i, b, c int = 3, 4, 5
	fmt.Printf("%d\n", b)
	global = &c
	return &i
}
```

这里只有b的生命周期是在函数内，i，c均不是。

编译器会自动选择在栈上还是在堆上分配局部变量的存储空间，但可能令人惊讶的是，这个选择并不是由用var还是new声明变量的方式决定的。

对于函数返回后依然存在的变量，其必须分配到堆上，这些变量使用Go语言的术语说，从函数中逃逸了。其实在任何时候，你并不需为了编写正确的代码而要考虑变量的逃逸行为，要记住的是，逃逸的变量需要额外分配内存，同时对性能的优化可能会产生细微的影响。

## 赋值

可以使用`=`或`++`,`--`进行赋值。也可以使用二元表达式赋值，入`+=`, `*=`等等。

注意：自增和自减是语句，而不是表达式，因此`x = i++`之类的表达式是错误的。

### 元组赋值

元组赋值是另一种形式的赋值语句，它允许同时更新多个变量的值。在赋值之前，赋值语句右边的所有表达式将会先进行求值，然后再统一更新左边对应变量的值。例如交换元素

```
x, y = y, x

a[i], a[j] = a[j], a[i]
```

有些表达式会产生多个值，比如调用一个有多个返回值的函数。当这样一个函数调用出现在元组赋值右边的表达式中时（译注：右边不能再有其它表达式），**左边变量的数目必须和右边一致**。

例如os.Open是用额外的返回值返回一个error类型的错误，还有一些是用来返回布尔值，通常被称为ok。在稍后我们将看到的三个操作都是类似的用法。如果map查找、类型断言或通道接收出现在赋值语句的右边，它们都可能会产生两个结果，有一个额外的布尔结果表示操作是否成功：

```
v, ok = m[key]             // map lookup
v, ok = x.(T)              // type assertion
v, ok = <-ch               // channel receive
```

map查找（§4.3）、类型断言（§7.10）或通道接收（§8.4.2）出现在赋值语句的右边时，并不一定是产生两个结果，也可能只产生一个结果。对于只产生一个结果的情形，map查找失败时会返回零值，类型断言失败时会发生运行时panic异常，通道接收失败时会返回零值（阻塞不算是失败）。例如下面的例子：

```
v = m[key]                // map查找，失败时返回零值
v = x.(T)                 // type断言，失败时panic异常
v = <-ch                  // 管道接收，失败时返回零值（阻塞不算是失败）

_, ok = m[key]            // map返回2个值
_, ok = mm[""], false     // map返回1个值
_ = mm[""]                // map返回1个值
```

和变量声明一样，我们可以用下划线空白标识符`_`来丢弃不需要的值。

```
_, err = io.Copy(dst, src) // 丢弃字节数
_, ok = x.(T)              // 只检测类型，忽略具体值
```

对于这种可以返回一个值，也可能返回两个值的函数来说，尽量还是使用两个值。

例如

```
func test24() {
	var value map[string]int = make(map[string]int)
	value["key"] = 7
	var v int
	var err bool
	v, err = value["test"]
	if !err {
		fmt.Print("don't exist test\n")
	} else {
		fmt.Printf("%d\n", v)
	}
}
```

此时可以正常运行。

```
func test24() {

	var value map[string]int = make(map[string]int)
	value["key"] = 7
	var v int
	v = value["test"]
	if v == 0 {
		fmt.Print("don't exist test\n")
	} else {
		fmt.Printf("%d\n", v)
	}
}
```

这样也可以正常运行，v为0是不存在还是是0

### 可赋值性

赋值语句是显示赋值，还有很多隐式赋值：函数调用会隐式地将调用参数的值赋值给函数的参数变量，一个返回语句会隐式地将返回操作的值赋值给结果变量，一个复合类型的字面量也会产生赋值行为。

不管是隐式还是显式地赋值，在赋值语句左边的变量和右边最终的求到的值**必须有相同的数据类型**。更直白地说，只有右边的值对于左边的变量是可赋值的，赋值语句才是允许的。

对于普通的数据类型，可赋值规则为：类型必须完全匹配，nil可以赋值给任何指针或引用类型的变量。

对于两个值是否可以用`==`或`!=`进行相等比较的能力也和可赋值能力有关系：对于任何类型的值的相等比较，第二个值必须是对第一个值类型对应的变量是可赋值的，

## 类型

变量或表达式的类型定义了对应存储值的属性特征，例如数值在内存的存储大小（或者是元素的bit个数），它们在内部是如何表达的，是否支持一些操作符，以及它们自己关联的方法集等。

一些变量有着相同的内部结构，但是却表示完全不同的概念。一个类型声明语句创建了一个新的类型名称，和现有类型具有相同的底层结构。新命名的类型提供了一个方法，用来分隔不同概念的类型，这样即使它们**底层类型相同也是不兼容的**。

```
type 类型名字 底层类型
```

类型声明语句一般出现在包一级，因此如果新创建的类型名字的首字符大写，则在包外部也可以使用。

例如

```
// Celsius is 摄氏温度
type Celsius float64

// Fahrenheit is 华氏温度
type Fahrenheit float64
```

Celsius和Fahrenheit分别对应不同的温度单位。它们虽然有着相同的底层类型float64，但是它们是不同的数据类型，因此它们不可以被相互比较或混在一个表达式运算。刻意区分类型，可以避免一些像无意中使用不同单位的温度混合计算导致的错误；因此需要一个类似Celsius(t)或Fahrenheit(t)形式的显式转型操作才能将float64转为对应的类型。

Celsius(t)和Fahrenheit(t)是类型转换操作，它们并不是函数调用。类型转换不会改变值本身，但是会使它们的语义发生变化。

**对于每一个类型T，都有一个对应的类型转换操作T(x)，用于将x转为T类型。只有当两个类型的底层基础类型相同时，才允许这种转型操作，或者是两者都是指向相同底层结构的指针类型，这些转换只改变类型而不会影响值本身。**

底层数据类型决定了内部结构和表达方式，也决定是否可以像底层类型一样，但是不能和flaot64一起计算，不同类型不能直接计算。

比较运算符`==`和`<`也可以用来比较一个命名类型的变量和另一个有相同类型的变量，或有着相同底层类型的未命名类型的值之间做比较。但是如果两个值有着不同的类型，则不能直接进行比较：

```
var c Celsius
var f Fahrenheit
fmt.Println(c == 0)          // "true"
fmt.Println(f >= 0)          // "true"
fmt.Println(c == f)          // compile error: type mismatch
fmt.Println(c == Celsius(f)) // "true"!
```

命名类型还可以为该类型的值定义新的行为。这些行为表示为一组关联到该类型的函数集合，我们称为类型的方法集。类型的参数出现在了函数名的前面，表示声明的是类型的一个方法，例如

```go
package main

import "fmt"

// Celsius is 摄氏温度
type Celsius float64

// Fahrenheit is 华氏温度
type Fahrenheit float64

func test25() {
	var c Celsius = 36.5
	// var f Fahrenheit
	fmt.Printf("c is %g, f is %g\n", c, c.cTof())
}

// CToF is 摄氏温度转华氏温度
func CToF(c Celsius) Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}

// FToC is 华氏温度转摄氏温度
func FToC(f Fahrenheit) Celsius {
	return Celsius((f - 32) * 5 / 9)
}

// CToF is Celsius类的摄氏温度转华氏温度方法
func (c Celsius) cTof() Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}

// CToF is Celsius类的摄氏温度转华氏温度方法
func (f Fahrenheit) fToc() Celsius {
	return Celsius((f - 32) * 5 / 9)
}

```



## 包和文件

Go语言中的包和其他语言的库或模块的概念类似，目的都是为了支持模块化、封装、单独编译和代码重用。一个包的源代码保存在一个或多个以.go为文件后缀名的源文件中，通常一个包所在目录路径的后缀是包的导入路径。

每个包都对应一个独立的名字空间。对于自己的包使用go mad对空间进行初始化。例如`go mod init coding`此时会在当前目录下生成`go.mod`文件，例如

```
module coding

go 1.15
```

之后在该目录下的包，都可以使用`coding`作为起始目录进行import。

包还可以让我们通过控制哪些名字是外部可见的来隐藏内部实现信息。在Go语言中，一个简单的规则是：如果一个名字是大写字母开头的，那么该名字是导出的。

在每个源文件的包声明前紧跟着的注释是包注释。通常，包注释的第一句应该先是包的功能概要说明。一个包通常只有一个源文件有包注释（译注：如果有多个包注释，目前的文档工具会根据源文件名的先后顺序将它们链接为一个包注释）。如果包注释很大，通常会放到一个独立的doc.go文件中。

如果导入了一个包，但是又没有使用该包将被当作一个编译错误处理。这种强制规则可以有效减少不必要的依赖。

例如如下代码结构

```
coding
	go.mad
	main.go
	tempcov
		tempcov.go
```

其中`go.mad`,内容为

```
module coding

go 1.15
```

`main.go`内容为

```
package main

import (
	"coding/tempcov"
	"fmt"
	"os"
	"strconv"
)

func main() {
	var f tempcov.Fahrenheit
	if len(os.Args) > 1 {
		v, err := strconv.ParseFloat(os.Args[1], 64)
		if err == nil {
			f = tempcov.Fahrenheit(v)
		} else {
			fmt.Scan(&f)
		}
	} else {
		fmt.Scan(&f)
	}
	fmt.Printf("f is %g, c is %g\n", f, tempcov.FToC(f))
	fmt.Printf("f is %g, c is %g\n", f, f.FToc())
}
```

`tempcov.go`内容为

```
package tempcov

import "fmt"

// Celsius is 摄氏温度
type Celsius float64

// Fahrenheit is 华氏温度
type Fahrenheit float64

func test25() {
	var c Celsius = 36.5
	// var f Fahrenheit
	fmt.Printf("c is %g, f is %g\n", c, c.CTof())
}

// CToF is 摄氏温度转华氏温度
func CToF(c Celsius) Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}

// FToC is 华氏温度转摄氏温度
func FToC(f Fahrenheit) Celsius {
	return Celsius((f - 32) * 5 / 9)
}

// CToF is Celsius类的摄氏温度转华氏温度方法
func (c Celsius) CTof() Fahrenheit {
	return Fahrenheit(c*9/5 + 32)
}

// CToF is Celsius类的摄氏温度转华氏温度方法
func (f Fahrenheit) FToc() Celsius {
	return Celsius((f - 32) * 5 / 9)
}

```

这里，在main中导入tempcov是以go mod中定义的模块coding为起始目录，使用tempcov.go中定义的包。



### 包的初始化

包的初始化首先是解决包级变量的依赖顺序，然后按照包级变量声明出现的顺序依次初始化：

```
var a = b + c // a 第三个初始化, 为 3
var b = f()   // b 第二个初始化, 为 2, 通过调用 f (依赖c)
var c = 1     // c 第一个初始化, 为 1
```

如果一个包含有多个`.go`源文件，会按照发给编译器的顺序进行初始化。

对于包级别声明的变量，如果有初始化表达式则用表达式初始化，还有一些没有初始化表达式的，例如某些表格数据初始化并不是一个简单的赋值过程。在这种情况下，我们可以用一个特殊的init初始化函数来简化初始化工作。每个文件都可以包含多个init初始化函数。

```
func init() { /* ... */ }
```

init初始化函数除了不能被调用或引用外，其他行为和普通函数类似。在每个文件中的init初始化函数，在程序开始执行时按照它们声明的顺序被自动调用。

每个包在解决依赖的前提下，以导入声明的顺序初始化，每个包只会被初始化一次。初始化工作是自下而上进行的，main包最后被初始化。以这种方式，可以确保在main函数执行之前，所有依赖的包都已经完成初始化工作了。

## 作用域

go语言作用域与C++的类似，使用`{}`分割变量的作用域。还有一种分割方式为隐式分割，例如使用for，if等，其中的赋值语句和其对应的`{}`是一个作用域。

### 与C++区别

与c++一样，同一个`{}`中，一个变量不能被重复声明，在一个函数中个，子`{}`中可以重新声明变量，此时，子`{}`中的变量会覆盖其父`{}`中的变量，但有一定的区别是，对于for和if来说，C++似乎是认为其里面的表达式和`{}`是同一个`{}`，并且都是函数的一个子`{}`，因此，函数中其他地方是不能访问for中的变量的。即如下信息是不被允许的：

```
#include<iostream>
using namespace std;
int main() {
    for(int i = 0; i< 10; i++) {
        int i = i + 1;// error
    }
    cout<<i<<endl; //error
    return 0;
}
```

这时，C++认为，其`{}`范围实际为：

```
{for{   int i = 0; i< 10; i++ 
        int i = i + 1;
}}
```

此时，在for循环中重复定义了i，因此会出编译错误。

但对于go来说，其认为for后面的语句和随后的`{}`不是一个级别的`{}`,但也都是在函数下面的一个子`{}`,因此实际存在三个作用域。例如

```
for i := 0; i < 3; i++ {
		i := i + 1
		fmt.Printf("%d \n", i)
	}
	fmt.Println(i) //error
```

其是可以正常运行的，并未重复定义i。

### for循环

for循环只有一种形式：

```
for initialization; condition; post {
    // zero or more statements
}
```

for循环的这三个部分每个都可以省略，如果省略`initialization`和`post`，分号也可以省略：

```
for condition {
    // ...
}
```

如果连condition也省略，则为无限循环。

### if

对于if来说，其有两种形式：

```
if  条件表达式 {
	   		语句1
} else ...
```

还有

```go
if 初始化表达式; 条件表达式 {
	   		语句1
}
```

### 作用域举例

对于作用域来说，for或if等其作用域均是与下方`{}`一致的，例如

```
if f, err := os.Open(fname); err != nil { // compile error: unused: f
    return err
}
f.ReadByte() // compile error: undefined f
f.Close()    // compile error: undefined f
```

这会导致编译错误，因为f的作用域不支持if结束后进行访问。

还有对于简化赋值来说，其不会对函数外边变量赋值。例如

```
var cwd string

func getCWD() {
	cwd ,err: = os.Getwd()
	if err == nil {
		fmt.Print(cwd)
	}
}
```

这里会产生错误，因为简短赋值不会对包级别的变量赋值。

改成如下方式即可

```
var cwd string

func getCWD() {
	var err error
	cwd, err = os.Getwd()
	if err == nil {
		fmt.Print(cwd)
	}
}
```

还有一种情况

```
var cwd string

func getCWD() {
	var err error
	if cwd, err = os.Getwd(); err != nil {
		fmt.Print("get cwd error \n")
	}
	fmt.Printf(cwd)
}
```

此时，有用cwd是包级别作用域，因此，cwd也可以继续使用。

# 基础数据类型

## 整型

Unicode字符rune类型是和int32等价的类型，通常用于表示一个Unicode码点。这两个名称可以互换使用。同样byte也是uint8类型的等价类型，byte类型一般用于强调数值是一个原始的数据而不是一个小的整数。

一种无符号的整数类型uintptr，没有指定具体的bit大小但是足以容纳指针。uintptr类型只有在底层编程时才需要，特别是Go语言和C语言函数库或操作系统接口相交互的地方。

go提供了如下位运算符。

```
&      位运算 AND
|      位运算 OR
^      位运算 XOR
&^     位清空 (AND NOT)
<<     左移
>>     右移
```

## 浮点数

基本与C++一致。一个float32类型的浮点数可以提供大约6个十进制数的精度，而float64则可以提供约15个十进制数的精度；通常应该优先使用float64类型，因为float32类型的累计计算误差很容易扩散，并且float32能精确表示的正整数并不是很大（译注：因为float32的有效bit位只有23个，其它的bit位用于指数和符号；当整数大于23bit能表达的范围时，float32的表示将出现误差）。

## 负数

Go语言提供了两种精度的复数类型：complex64和complex128，分别对应float32和float64两种浮点数精度。内置的complex函数用于构建复数，内建的real和imag函数分别返回复数的实部和虚部：

```
var x complex128 = complex(1, 2) // 1+2i
var y complex128 = complex(3, 4) // 3+4i
fmt.Println(x*y)                 // "(-5+10i)"
fmt.Println(real(x*y))           // "-5"
fmt.Println(imag(x*y))           // "10"
```

## 布尔型

布尔值并不会隐式转换为数字值0或1，反之亦然。布尔值可以和&&（AND）和||（OR）操作符结合，并且有短路行为。

## 字符串

一个字符串是一个不可改变的字节序列。内置的len函数可以返回一个字符串中的字节数目。索引操作s[i]返回第i个字节的字节值，i必须满足0 ≤ i< len(s)条件约束。如果试图访问超出字符串索引范围的字节将会导致panic异常。

第i个字节并不一定是字符串的第i个字符，因为对于非ASCII字符的UTF8编码会要两个或多个字节。

字符串可以使用类似于python的切片：

```
fmt.Println(s[:5]) // "hello"
fmt.Println(s[7:]) // "world"
fmt.Println(s[:])  // "hello, world"
```

使用+操作可以进行字符串拼装。

字符串可以用==和<进行比较。

因为字符串是不可修改的，因此尝试修改字符串内部数据的操作也是被禁止的：

```
s[0] = 'L' // compile error: cannot assign to s[0]
```

不变性意味着如果两个字符串共享相同的底层数据的话也是安全的，这使得复制任何长度的字符串代价是低廉的。

与C++一样，`\`表示转义

```
\a      响铃
\b      退格
\f      换页
\n      换行
\r      回车
\t      制表符
\v      垂直制表符
\'      单引号 (只用在 '\'' 形式的rune符号面值中)
\"      双引号 (只用在 "..." 形式的字符串面值中)
\\      反斜杠
```

一个原生的字符串面值形式是反引号，其内可以写入任何数据，也不会被转义，可以跨行。原生字符串面值用于编写正则表达式会很方便，因为正则表达式往往会包含很多反斜杠。原生字符串面值同时被广泛应用于HTML模板、JSON面值、命令行提示信息以及那些需要扩展到多行的场景。

```
var str string = `hello
word
\n
`
```

### UTF-8

UTF8是一个将Unicode码点编码为字节序列的变长编码。UTF8编码使用1到4个字节来表示每个Unicode码点，ASCII部分字符只使用1个字节，常用字符部分使用2或3个字节表示。每个符号编码后第一个字节的高端bit位用于表示编码总共有多少个字节。如果第一个字节的高端bit为0，则表示对应7bit的ASCII字符，ASCII字符每个字符依然是一个字节，和传统的ASCII编码兼容。

```
0xxxxxxx                             runes 0-127    (ASCII)
110xxxxx 10xxxxxx                    128-2047       (values <128 unused)
1110xxxx 10xxxxxx 10xxxxxx           2048-65535     (values <2048 unused)
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx  65536-0x10ffff (other values unused)
```

unicode/utf8包则提供了用于rune字符序列的UTF8编码和解码的功能。

对于包含中文的string求长度时，使用`unicode/utf8`中的`utf8.RuneCountInString(s)`函数即可

```
var str string = "hello,世界"
fmt.Println(utf8.RuneCountInString(str)) // 8
```

为了处理这些真实的字符，我们需要一个UTF8解码器。unicode/utf8包提供了该功能，我们可以这样使用：

```
for i := 0; i < len(s); {
    r, size := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%d\t%c\n", i, r)
    i += size
}
```

每一次调用DecodeRuneInString函数都返回一个r和长度，r对应字符本身，长度对应r采用UTF8编码后的编码字节数目。上述程序输出为

```
0       h
1       e
2       l
3       l
4       o
5       ,
6       世
9       界
```

Go语言的range循环在处理字符串的时候，会自动隐式解码UTF8字符串。

```
for i, r := range str {
		fmt.Printf("%d\t%q\t%d\n", i, r, r)
	}
	
// output
0       'h'     104
1       'e'     101
2       'l'     108
3       'l'     108
4       'o'     111
5       ','     44
6       '世'    19990
9       '界'    30028
```

UTF8字符串作为交换格式是非常方便的，但是在程序内部采用rune序列可能更方便，因为rune大小一致，支持数组索引和方便切割。

二者之间转换为

```
s := "プログラム"
fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
r := []rune(s)
fmt.Printf("%x\n", r)  // "[30d7 30ed 30b0 30e9 30e0]"

fmt.Println(string(r))
```

在第一个Printf中的`% x`参数用于在每个十六进制数字前插入一个空格。

### 字符串和Byte切片

标准库中有四个包对字符串处理尤为重要：bytes、strings、strconv和unicode包。

strings包提供了许多如字符串的查询、替换、比较、截断、拆分和合并等功能。

bytes包也提供了很多类似功能的函数，但是针对和字符串有着相同结构的[]byte类型。`byte`表示的是一个byte（字节），即大小为`0-127`。而rune（字符）是表示的一个UTF8字符，和C++中的char类似。由于string是只读类型，不方便处理，因此使用[]byte和[]rune来处理需要更改的字符串数据。使用bytes.Buffer类型是对字符串数据的一个封装。Buffer类型用于字节slice的缓存。一个Buffer开始是空的，但是随着string、byte或[]byte等类型数据的写入可以动态增长，一个bytes.Buffer变量并不需要初始化，因为零值也是有效的。

string、[]byte、[]rune之间转换为

```
var str string = "hello,世界"
var bstr []rune = []rune(str)
var cstr []byte = []byte(str)
var dstr string = string(bstr)
var estr string = string(cstr)
```

`[]byte`不能直接和`[]rune`直接进行转换。转换后，都是重新分配了空间，因此更改一个不会对另一个有影响。（编译器进行了写时复制的优化）。

strconv包提供了布尔型、整型数、浮点数和对应字符串的相互转换，还提供了双引号转义相关的转换。

unicode包提供了IsDigit、IsLetter、IsUpper和IsLower等类似功能，它们用于给字符分类。每个函数有一个单一的rune类型的参数，然后返回一个布尔值。

### 字符串和数字之间转换

由strconv包提供字符串和数值之间的转换。

数组转到字符串方式

```
x := 123
y := fmt.Sprintf("%d", x)
fmt.Println(y, strconv.Itoa(x)) // "123 123"
```

字符串转数字

```
x, err := strconv.Atoi("123")             // x is an int
y, err := strconv.ParseInt("123", 10, 64) // base 10, up to 64 bits
```

具体参考strconv包。



## 常量

常量表达式的值在编译期计算，而不是在运行期。可以批声明一组常量：

```
const (
    e  = 2.71828182845904523536028747135266249775724709369995957496696763
    pi = 3.14159265358979323846264338327950288419716939937510582097494459
)
```

一个常量的声明也可以包含一个类型和一个值，但是如果没有显式指明类型，那么将从右边的表达式推断类型。

### iota常量生成器

常量声明可以使用iota常量生成器初始化，它用于生成一组以相似规则初始化的常量，但是不用每行都写一遍初始化表达式。在一个const声明语句中，在第一个声明的常量所在的行，iota将会被置为0，然后在每一个有常量声明的行加一。

例如

```
type Flags uint

const (
    FlagUp Flags = 1 << iota // is up
    FlagBroadcast            // supports broadcast access capability
    FlagLoopback             // is a loopback interface
    FlagPointToPoint         // belongs to a point-to-point link
    FlagMulticast            // supports multicast access capability
)
```

### 无类型常量

虽然一个常量可以有任意一个确定的基础类型，但是许多常量并没有一个明确的基础类型。编译器为这些没有明确基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度。

通过延迟明确常量的具体类型，无类型的常量不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换。

math.Pi无类型的浮点数常量，可以直接用于任意需要浮点数或复数的地方：

```Go
var x float32 = math.Pi
var y float64 = math.Pi
var z complex128 = math.Pi
```

对于常量面值，不同的写法可能会对应不同的类型。例如0、0.0、0i和`\u0000`虽然有着相同的常量值，但是它们分别对应无类型的整数、无类型的浮点数、无类型的复数和无类型的字符等不同的常量类型。

除法运算符/会根据操作数的类型生成对应类型的结果。因此，不同写法的常量除法表达式可能对应不同的结果：

```
var f float64 = 212
fmt.Println((f - 32) * 5 / 9)     // "100"; (f - 32) * 5 is a float64
fmt.Println(5 / 9 * (f - 32))     // "0";   5/9 is an untyped integer, 0
fmt.Println(5.0 / 9.0 * (f - 32)) // "100"; 5.0/9.0 is an untyped float
```

只有常量可以是无类型的。当一个无类型的常量被赋值给一个变量的时候，就像下面的第一行语句，或者出现在有明确类型的变量声明的右边，如下面的其余三行语句，无类型的常量将会被隐式转换为对应的类型，如果转换合法的话。



# 复合数据类型

## 数组

数组是一个由固定长度的特定类型元素组成的序列，一个数组可以由零个或多个元素组成。因为数组的长度是固定的。len确定数组长度。

初始化方式为

```
var q [3]int = [3]int{1, 2, 3}

q := [...]int{1, 2, 3}
```

在数组字面值中，如果在数组的长度位置出现的是“...”省略号，则表示数组的长度是根据初始化值的个数来计算。

数组的长度是数组类型的一个组成部分，因此[3]int和[4]int是两种不同的数组类型。数组的长度必须是常量表达式，因为数组的长度需要在编译阶段确定。

初始化数组是，除了直接提供顺序初始化值序列，可以指定一个索引和对应值列表的方式初始化。例如

```
r := [...]int{99: -1}
```

定义了一个含有100个元素的数组r，最后一个元素被初始化为-1，其它元素都是用0初始化。

如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，这时候我们可以直接通过==比较运算符来比较两个数组。

在**函数中传递数组，是数组的拷贝而不是指针**，因此传递较大的数组是十分费时的。但我们也可以显示的传递数组的指针，例如

```
func testsz(sz *[3]int) {
	for i := 0; i < 3; i++ {
		fmt.Println(sz[i])
		sz[i]++ // = (*sz)[i]++
	}
}

func test41() {
	var sz [3]int = [3]int{1, 2, 3}
	testsz(&sz)
	for i := 0; i < 3; i++ {
		fmt.Println(sz[i])
	}
}

//output
1
2
3
2
3
4
```



## Slice(切片)

Slice（切片）代表变长的序列，序列中每个元素都有相同的类型，一个slice类型一般写作[]T。就其存储来说，Slice类似于vector。一个slice由三个部分构成：指针、长度和容量。指针指向第一个slice元素对应的底层数组元素的地址。长度对应slice中元素的数目；长度不能超过容量，容量一般是从slice的开始位置到底层数据的结尾位置。内置的len和cap函数分别返回slice的长度和容量。

多个slice之间可以共享底层的数据，并且引用的数组部分区间可能重叠。

即切片的赋值是浅拷贝，要进行深拷贝，使用copy函数。

和数组不同的是，slice之间不能比较，因此我们不能使用==操作符来判断两个slice是否含有全部相等元素。不过标准库提供了高度优化的bytes.Equal函数来判断两个字节型slice是否相等（[]byte），但是对于其他类型的slice，我们必须自己展开每个元素进行比较。

slice唯一合法的比较操作是和nil比较，例如：

```Go
if summer == nil { /* ... */ }
```

如果你需要测试一个slice是否是空的，使用len(s) == 0来判断，而不应该用s == nil来判断。

内置的make函数创建一个指定元素类型、长度和容量的slice。容量部分可以省略，在这种情况下，容量将等于长度。

```Go
make([]T, len)
make([]T, len, cap) // same as make([]T, cap)[:len]
```

slice容量的增长也和vector类似，首先创建一个空间，当slice实际空间不足是（append函数增加元素），再次分配空间，来扩大原来的空间。由于重新分配了空间，因此append函数的返回值，应该被重新赋值给其本身。

```
func test42() {
	T := make([]int, 4, 4)
	for i := 0; i < 4; i++ {
		T[i] = i
	}
	var V []int
	V = T[1:3]
	T[1] = 10
	for i := 0; i < len(V); i++ {

		fmt.Println(V[i])
	}
	for i := 4; i < 100; i++ {
		T = append(T, i)
	}

	for i := 0; i < len(V); i++ {
		fmt.Println(V[i])
	}
}

// output
10
2
10
2
```

上例中，虽然T对应的底层空间发生了变更，但V依然有效，这是因为在给T分配空间时，会先进行一次深拷贝，将原本T对应的底层数据拷贝到新的空间中，但由于V依然指向原本的底层空间，因此完成深拷贝后，原来的底层空间并不会被释放。可以看着实际进行了一次深拷贝。这时候如果再更新T中数据，例如

```
T[1] = 100
for i := 0; i < len(V); i++ {
	fmt.Println(V[i])
}

// output
10
2
```

V中数据依然不会变。但应该尽量避免这种操作，对于两者分离时，应该主动进行深拷贝（copy）操作。

copy函数的第一个参数是要复制的目标slice，第二个参数是源slice，目标和源的位置顺序和`dst = src`赋值语句是一致的。两个slice可以共享同一个底层数组，甚至有重叠也没有问题。copy函数将返回成功复制的元素的个数（我们这里没有用到），等于两个slice中较小的长度，所以我们不用担心覆盖会超出目标slice的范围。

append一次可以添加多个元素。

为了避免在函数内对传递的分片进行更改，可以使用copy来进行深拷贝，例如

```
func testCopy(f []int) {
	var value []int
	value = make([]int, len(f))
	copy(value, f)
	for i := 0; i < len(value); i++ {
		value[i] = value[i] * 10
		fmt.Print(value[i])
		fmt.Print(" ")
	}
	fmt.Println()
}

func test42() {
	T := make([]int, 4, 4)
	for i := 0; i < 4; i++ {
		T[i] = i
	}
	testCopy(T[1:3])
	for i := 0; i < len(T); i++ {
		fmt.Print(T[i])
		fmt.Print(" ")
	}
	fmt.Println()
}

// output
10 20
0 1 2 3
```

注意，这里的value一定要先使用make分配空间，否则拷贝时只会拷贝value和f中长度较小的，如果不使用make进行空间分配，则长度为0。还有应该尽量在函数内copy，这样结束后，创建的空间就会释放。

不使用深拷贝有时会发生意想不到的结果

```
func main() {
	var list []int = make([]int, 5)
	for i := 0; i < 5; i++ {
		list[i] = i
	}
	list2 := list[1:3]
	
	list2 = append(list2, 10)
	for i := 0; i < len(list2); i++ {
		fmt.Printf("%d ", list2[i])
	}

	fmt.Println()
	for i := 0; i < len(list); i++ {
		fmt.Printf("%d ", list[i])
	}
	fmt.Println()
}

//output
1 2 10 
0 1 2 10 4 
```

由于list和list2共享同一个底层数据，因此在list2后增加数据，其实是对list中的一个数据的更改。

## map

一个map就是一个哈希表的引用，map类型可以写为map[K]V。

map中所有的key都有相同的类型，所有的value也有着相同的类型，但是key和value之间可以是不同的数据类型。其中K对应的key必须是支持==比较运算符的数据类型，所以map可以通过测试key是否相等来判断是否已经存在。

可以使用`make`来创建一个空map。

创建空map：

```
value := make(map[string]int)
=
var value map[string]int = map[string]int{}
```

对于

```
var value map[string]int
```

此时value为nil。向一个nil值的map存入元素将导致一个panic异常。

创建一个简单的map：

```
ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}
```

delete删除元素

```
delete(ages, "alice") // remove element ages["alice"]
```

所有这些操作是安全的，即使这些元素不在map中也没有关系；如果一个查找失败将返回value类型对应的零值，例如，即使map中不存在“bob”下面的代码也可以正常工作，因为ages["bob"]失败时将返回0。

```Go
ages["bob"] = ages["bob"] + 1 // happy birthday!
```

map中的元素并不是一个变量，因此我们不能对map的元素进行取址操作：

```Go
_ = &ages["bob"] // compile error: cannot take address of map element
```

使用range遍历map：

```
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```

其中key顺序是不定的，每一次遍历的顺序都不相同。

对于不存在的key，访问时，返回0值，但有事我们也需要知道是否不存在key，此时使用

```
age, ok := ages["bob"]
if !ok { /* "bob" is not a key in this map; age == 0. */ }
```

有时候我们需要一个map或set的key是slice类型，但是map的key必须是可比较的类型，但是slice并不满足这个条件。不过，我们可以通过两个步骤绕过这个限制。第一步，定义一个辅助函数k，将slice转为map对应的string类型的key，确保只有x和y相等时k(x) == k(y)才成立。然后创建一个key为string类型的map，在每次对map操作时先用k辅助函数将slice转化为string类型。

```
var m = make(map[string]int)

func k(list []string) string { return fmt.Sprintf("%q", list) }

func Add(list []string)       { m[k(list)]++ }
func Count(list []string) int { return m[k(list)] }
```

Map的value类型也可以是一个聚合类型，比如是一个map或slice。

```
var graph = make(map[string]map[string]bool)
```

go未提供map的深拷贝，相应在函数中不对传递的map进行变更，可以先将map序列号为string，然后在反序列化。



## 结构体

结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。

结构体中，相同的类型可以写在同一行，结构体中，声明时如果元素顺序不同，那么就不是相同的结构体。结构体可以通过作为参数和返回值，也可以通过指数传参，如果想通过函数来更改结果体中的值，与C++类似，可以选择返回结构体的指针。go与通过结构体指针访问其中成员，与C++有一些不同，其可以直接进行点操作，而不用加*。

例如

```
type Employee struct {
    ID            int
    Name, Address string
    DoB           time.Time
    Position      string
    Salary        int
    ManagerID     int
}

var employeeOfTheMonth *Employee = &dilbert
employeeOfTheMonth.Position += " (proactive team player)"

func EmployeeByID(id int) *Employee { /* ... */ }

fmt.Println(EmployeeByID(dilbert.ManagerID).Position) // "Pointy-haired boss"

id := dilbert.ID
EmployeeByID(id).Salary = 0 // fired for... no real reason 
```

结构体类型的零值是每个成员都是零值。

### 结构体字面值

实例化结果体时对其中参数赋值有两种方式，一个是按照结构体声明顺序依次赋值，另一个是指定元素进行赋值，例如

```
type Point struct{ X, Y int }

p := Point{1, 2}

P := Point{Y:2, X:1}
```

对于未指定的值，则是对应元素的0值。不能企图在外部包中用第一种顺序赋值的技巧来偷偷地初始化结构体中未导出的成员。

默认情况下我们不能在别的文件中使用未导出的结构体数据，但似乎可以巧妙的避开限制，例如

testclass.go文件

```
package testclass

// Point is use to test
type Point struct {
	X, Y, x, y int
}

// TestSet ....
func (c *Point) TestSet(i int, j int) {
	c.X = i
	c.Y = j
}

// TestGet ...
func (c *Point) TestGet() int {
	return c.X
}
```

main.go文件

```
package main

import (
	"coding/testclass"
	"fmt"
)

func main() {
	var p testclass.Point = testclass.Point{}
	p.TestSet(2, 3)
	fmt.Println(p.TestGet())
}
```

执行输出为2.

### 结构体的比较

如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，那样的话两个结构体将可以使用==或!=运算符进行比较。

### 结构体嵌入匿名成员

Go语言有一个特性让我们只声明一个成员对应的数据类型而不指名成员的名字；这类成员就叫匿名成员。匿名成员的数据类型必须是命名的类型或指向一个命名的类型的指针。

得益于匿名嵌入的特性，我们可以直接访问叶子属性而不需要给出完整的路径。例如

```
package main

import (
	"fmt"
)

func main() {
	var f fatherclass = fatherclass{}
	f.name = "hello" // f.baseclass.name = "hello"
	f.len = 20 // f.baseclass.len = 20
	f.otherinfo = "other"
	fmt.Println(f)
}

type baseclass struct {
	len  int
	name string
}

type fatherclass struct {
	baseclass
	otherinfo string
}
```

输出为

```
{{20 hello} other}
```

在右边的注释中给出的显式形式访问这些叶子成员的语法依然有效，因此匿名成员并不是真的无法访问了。

但是对含有匿名成员赋值时，不能把其看作是子节点直接赋值，例如

```
t := fatherclass{20, "hello", "other"}
```

正常应该采用如下方式

```
t := fatherclass{baseclass{20, "hello"}, "other"}
```

因为匿名成员也有一个隐式的名字，因此不能同时包含两个类型相同的匿名成员，这会导致名字冲突。同时，因为成员的名字是由其类型隐式地决定的，所以匿名成员也有可见性的规则约束（是否是导出的）。

匿名成员的主要作用是其方法集，外层的结构体不仅仅是获得了匿名成员类型的所有成员，而且也获得了该类型导出的全部的方法。这个机制可以用于将一些有简单行为的对象组合成有复杂行为的对象。组合是Go语言中面向对象编程的核心。

## JSON

由标准库中的encoding/json、encoding/xml、encoding/asn1、github.com/golang/protobuf 包提提供对应的协议。

JSON类型有数字（十进制或科学记数法）、布尔值（true或false）、字符串，其中字符串是以双引号包含的Unicode字符序列，支持和Go语言类似的反斜杠转义特性。

一个JSON数组可以用于编码Go语言的数组和slice。

将结构体转换为json的过程叫编组（marshaling），编组通过json.Marshal完成。例如

```
package main

import (
	"encoding/json"
	"fmt"
	"log"
)

func main() {
	var movies = []Movie{
		{Title: "Casablanca", Year: 1942, Color: false,
			Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
		{Title: "Cool Hand Luke", Year: 1967, Color: true,
			Actors: []string{"Paul Newman"}},
		{Title: "Bullitt", Year: 1968, Color: true,
			Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
	}

	data, err := json.Marshal(movies)
	if err != nil {
		log.Fatal("Json marshaling failed: %s", err)
	}
	fmt.Printf("%s\n", data)
}

// Movie ...
type Movie struct {
	Title  string
	Year   int  `json:"released"`
	Color  bool `json:"color,omitempty`
	Actors []string
}
```

只有导出的结构体成员才会被编码，这也就是我们为什么选择用大写字母开头的成员名称。对于上述Movie类中，变量后为结构体成员Tag。一个结构体成员Tag是和在编译阶段关联到该成员的元信息字符串：

```
Year  int  `json:"released"`
Color bool `json:"color,omitempty"`
```

结构体的成员Tag可以是任意的字符串面值，但是通常是一系列用空格分隔的key:"value"键值对序列；因为值中含有双引号字符，因此成员Tag一般用原生字符串面值的形式书写。json开头键名对应的值用于控制encoding/json包的编码和解码的行为，并且encoding/...下面其它的包也遵循这个约定。成员Tag中json对应值的第一部分用于指定JSON对象的名字，比如将Go语言中的TotalCount成员对应到JSON中的total_count对象。Color成员的Tag还带了一个额外的omitempty选项，表示当Go语言结构体成员为空或零值时不生成该JSON对象（这里false为零值）。

这里直接输出的结果是一个字符串，为了更加美观的展示，可以选择json.MarshalIndent(movies, "", "    ")编码。该函数有两个额外的字符串参数用于表示每一行输出的前缀和每一个层级的缩进。

编码的逆序是解码，一般叫unmarshaling，通过json.Unmarshal。json.Unmarshal(data, &titles)其中data是字符串，titles是结构体实例。

下面的例子为解码http返回的json信息：

gclass.go文件

```
package github

import "time"

const IssuesURL = "https://api.github.com/search/issues"

type IssueSearchResult struct {
	TotalCount int `json:"total_count"`
	Items      []*Issue
}

type Issue struct {
	Number   int
	HTMLURL  string `json:"html_url"`
	Title    string
	State    string
	User     *User
	CreateAt time.Time
	Body     string
}

type User struct {
	Login   string
	HTMLURL string `json:"html_url"`
}
```

github.go

```
package github

import (
	"encoding/json"
	"fmt"
	"net/http"
	"net/url"
	"strings"
)

// SearchIssues ...
func SearchIssues(terms []string) (*IssueSearchResult, error) {
	q := url.QueryEscape(strings.Join(terms, " "))
	resp, err := http.Get(IssuesURL + "?q=" + q)
	if err != nil {
		return nil, err
	}

	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("search query failed: %s", resp.Status)
	}

	var result IssueSearchResult
	if err != json.NewDecoder(resp.Body).Decode(&result) {
		resp.Body.Close()
		return nil, err
	}

	resp.Body.Close()
	return &result, nil
}

```

main.go文件

```
package main

import (
	"coding/github"
	"fmt"
	"log"
	"os"
)

func main() {
	result, err := github.SearchIssues(os.Args[1:])
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("total count: %d\n", result.TotalCount)
	for _, item := range result.Items {
		fmt.Printf("#%-5d %9.9s %.55s\n",
			item.Number, item.User.Login, item.Title)
	}
}

```

基于流式的解码器json.Decoder，它可以从一个输入流解码JSON数据。



## 文本和HTML模板

text/template和html/template等模板包将格式化代码分离出来以便更安全地修改，它们提供了一个将变量值填充到一个文本或HTML格式的模板的机制。

一个模板是一个字符串或一个文件，里面包含了一个或多个由双花括号包含的`{{action}}`对象。大部分的字符串只是按字面值打印，但是对于actions部分将触发其它的行为。

每个actions都包含了一个用模板语言书写的表达式，一个action虽然简短但是可以输出复杂的打印值，模板语言包含通过选择结构体的成员、调用函数或方法、表达式控制流if-else语句和range循环语句，还有其它实例化模板等诸多特性。例如：

```go
const templ = `{{.TotalCount}} issues:
{{range .Items}}----------------------------------------
Number: {{.Number}}
User:   {{.User.Login}}
Title:  {{.Title | printf "%.64s"}}
Age:    {{.CreatedAt | daysAgo}} days
{{end}}`
```

该模板，先打印匹配到的issue总数，然后打印每个issue的编号、创建用户、标题还有存在的时间。对于每一个action，都有一个当前值的概念，对应点操作符，写作“.”。当前值“.”最初被初始化为调用模板时的参数，在当前例子中对应github.IssuesSearchResult类型的变量。模板中{ {.TotalCount}}对应action将展开为结构体中TotalCount成员以默认的方式打印的值。模板中{{range .Items}}和{{end}}对应一个循环action，因此它们直接的内容可能会被展开多次，循环每次迭代的当前值对应当前的Items元素的值。

在一个action中，`|`操作符表示将前一个表达式的结果作为后一个函数的输入，类似于UNIX中管道的概念。在Title这一行的action中，第二个操作是一个printf函数，是一个基于fmt.Sprintf实现的内置函数，所有模板都可以直接使用。对于Age部分，第二个动作是一个叫daysAgo的函数，通过time.Since函数将CreatedAt成员转换为过去的时间长度。

```
func daysAgo(t time.Time) int {
	return int(time.Since(t).Hours() / 24)
}
```

生成模板的输出需要两个处理步骤。第一步是要分析模板并转为内部表示，然后基于指定的输入执行模板。分析模板部分一般只需要执行一次。

```
report, err := template.New("report").Funcs(template.
		FuncMap{"daysAgo": github.DaysAgo}).Parse(github.Templ)
	if err != nil {
		log.Fatal(err)
	}
```

这里执行顺序为：template.New先创建并返回一个模板；Funcs方法将daysAgo等自定义函数注册到模板中，并返回模板；最后调用Parse函数分析模板。

因为模板通常在编译时就测试好了，如果模板解析失败将是一个致命的错误。template.Must辅助函数可以简化这个致命错误的处理：它接受一个模板和一个error类型的参数，检测error是否为nil（如果不是nil则发出panic异常），然后返回传入的模板。

模板完成创建，就可以将字符串作为输入源，os.StdOut作为输出源来执行模板：

```
func main() {
	result, err := github.SearchIssues(os.Args[1:])
	if err != nil {
		log.Fatal(err)
	}

	report := template.Must(template.New("report").Funcs(template.
		FuncMap{"DaysAgo": github.DaysAgo}).Parse(github.Templ))
	if err != nil {
		log.Fatal(err)
	}
	if err := report.Execute(os.Stdout, result); err != nil {
		log.Fatal(err)
	}
}
```

上述使用的是`text/template`。下面介绍`html/template`。它使用和text/template包相同的API和模板语言，但是增加了一个将字符串自动转义特性，这可以避免输入字符串和HTML、JavaScript、CSS或URL语法产生冲突的问题。具体看：https://docs.hacknode.org/gopl-zh/ch4/ch4-06.html。

# 函数

函数的类型被称为函数的标识符。如果两个函数形式参数列表和返回值列表中的变量类型一一对应，那么这两个函数被认为有相同的类型或标识符。形参和返回值的变量名不影响函数标识符，也不影响它们是否可以以省略参数类型的形式表示。

```
func add(x int, y int) int   {return x + y}
func sub(x, y int) (z int)   { z = x - y; return}
func first(x int, _ int) int { return x }
func zero(int, int) int      { return 0 }

fmt.Printf("%T\n", add)   // "func(int, int) int"
fmt.Printf("%T\n", sub)   // "func(int, int) int"
fmt.Printf("%T\n", first) // "func(int, int) int"
fmt.Printf("%T\n", zero)  // "func(int, int) int"
```

返回值列表描述了函数返回值的变量名以及类型。返回值也可以像形式参数一样被命名。在这种情况下，每个返回值被声明成一个局部变量，并根据该返回值的类型，将其初始化为0。

在函数体中，函数的形参作为局部变量，被初始化为调用者提供的值。函数的形参和有名返回值作为函数最外层的局部变量，被存储在相同的词法块中。偶尔遇到没有函数体的函数声明，这表示该函数不是以Go实现的。这样的声明定义了函数标识符。

```Go
package math

func Sin(x float64) float //implemented in assembly language
```

不像c++的一样，可以定义多个同名函数，C++不允许在同一个包中定义同名函数，即使参数不同。

## 多返回值

一个函数可以返回多个值。准确的变量名可以传达函数返回值的含义。尤其在返回值的类型都相同时，就像下面这样：

```Go
func Size(rect image.Rectangle) (width, height int)
func Split(path string) (dir, file string)
func HourMinSec(t time.Time) (hour, minute, second int)
```

如果一个函数所有的返回值都有显式的变量名，那么该函数的return语句可以省略操作数。这称之为bare return。即直接return即可。例如

```
func CountWordsAndImages(url string) (words, images int, err error) {
    resp, err := http.Get(url)
    if err != nil {
        return
    }
    doc, err := html.Parse(resp.Body)
    resp.Body.Close()
    if err != nil {
        err = fmt.Errorf("parsing HTML: %s", err)
        return
    }
    words, images = countWordsAndImages(doc)
    return
}
```

## 错误

panic是来自被调用函数的信号，表示发生了某个已知的bug。一个良好的程序永远不应该发生panic异常。内置的error是接口类型。error类型可能是nil或者non-nil。nil意味着函数运行成功，non-nil表示失败。对于non-nil的error类型,我们可以通过调用error的Error函数或者输出函数获得字符串类型的错误信息。

```
fmt.Println(err)
fmt.Printf("%v", err)
```

在Go中，函数运行失败时会返回错误信息，这些错误信息被认为是一种预期的值而非异常（exception）。

### 处理错误策略

1 传播错误，这意味着函数中某个子程序的失败，会变成该函数的失败，即发生错误是，将错误返回给调用者。

2 重试。

3 输出错误信息并结束程序。需要注意的是，这种策略只应在main中执行。对库函数而言，应仅向上传播错误，除非该错误意味着程序内部包含不一致性，即遇到了bug，才能在库函数中结束程序。

4 输出错误信息就足够了，不需要中断程序的运行。

5 忽略错误。

### 文件结尾错误

io包保证任何由文件结束引起的读取失败都返回同一个错误——io.EOF，该错误在io包中定义：

```
package io

import "errors"

// EOF is the error returned by Read when no more input is available.
var EOF = errors.New("EOF")
```

调用者只需通过简单的比较，就可以检测出这个错误。

```
in := bufio.NewReader(os.Stdin)
for {
    r, _, err := in.ReadRune()
    if err == io.EOF {
        break // finished reading
    }
    if err != nil {
        return fmt.Errorf("read failed:%v", err)
    }
    // ...use r…
}
```

## 函数值

在Go中，函数被看作第一类值（first-class values）：函数像其他值一样，拥有类型，可以被赋值给其他变量，传递给函数，从函数返回。对函数值（function value）的调用类似函数调用。例如

```
func Test5(n int) int { return n * n }

func main() {
	f := Test5
	fmt.Println(f(5))
}
```

函数类型的零值是nil。调用值为nil的函数值会引起panic错误：

```Go
    var f func(int) int
    f(3) // 此处f的值为nil, 会引起panic错误
```

函数值可以与nil比较：

```Go
    var f func(int) int
    if f != nil {
        f(3)
    }
```

但是函数值之间是不可比较的，也不能用函数值作为map的key。

函数值使得我们不仅仅可以通过数据来参数化函数，亦可通过行为。标准库中包含许多这样的例子。下面的代码展示了如何使用这个技巧。strings.Map对字符串中的每个字符调用add1函数，并将每个add1函数的返回值组成一个新的字符串返回给调用者。

```
func add1(r rune) rune { return r + 1 }

    fmt.Println(strings.Map(add1, "HAL-9000")) // "IBM.:111"
    fmt.Println(strings.Map(add1, "VMS"))      // "WNT"
    fmt.Println(strings.Map(add1, "Admix"))    // "Benjy"
```



## 匿名函数

拥有函数名的函数只能在包级语法块中被声明，通过函数字面量（function literal），我们可绕过这一限制，在任何表达式中表示一个函数值。函数字面量的语法和函数声明相似，区别在于func关键字后没有函数名。函数值字面量是一种表达式，它的值被称为匿名函数（anonymous function）。

函数字面量允许我们在使用函数时，再定义它。通过这种技巧，我们可以改写之前对strings.Map的调用：

```Go
strings.Map(func(r rune) rune { return r + 1 }, "HAL-9000")
```

通过这种方式定义的函数可以访问完整的词法环境（lexical environment），这意味着在函数中定义的内部函数可以引用该函数的变量。通过该方式，可以实现类似C++里函数中静态变量的效果。例如

```
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}
func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```

匿名内部函数可以访问和更新其所在函数的局部变量，这意味着匿名函数中，存在变量引用。Go使用闭包（closures）技术实现函数值，Go程序员也把函数值叫做闭包。

如下例子：计算机课程的相关依赖关系，即学习某个课程之前，先要学习另外的课程。这构成一个拓扑排序，前置条件可以构成有向图。图中的顶点表示课程，边表示课程间的依赖关系。显然，图中应该无环，这也就是说从某点出发的边，最终不会回到该点。下面使用深度优先搜索进行排序：

```
package funcation

// prereqs记录了每个课程的前置课程
var Prereqs = map[string][]string{
	"algorithms": {"data structures"},
	"calculus":   {"linear algebra"},
	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},
	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}

// TopoSort ....
func TopoSort(m map[string][]string) []string {
	var visited map[string]bool
	var order []string
	visited = make(map[string]bool)
	var visitAll func(classList []string)
	visitAll = func(classList []string) {
		for _, className := range classList {
			if !visited[className] {
				visited[className] = true
				if _, err := m[className]; err {
					visitAll(m[className])
				}
				order = append(order, className)
			}
		}
	}
	var keys []string
	for key := range m {
		keys = append(keys, key)
	}
	visitAll(keys)
	return order
}
```

```
package main

import (
	"coding/funcation"
	"fmt"
)

func main() {
	for i, className := range funcation.TopoSort(funcation.Prereqs) {
		fmt.Printf("%d : %s\n", i, className)
	}
}
```

当匿名函数需要被递归调用时，我们必须首先声明一个变量（在上面的例子中，我们首先声明了 visitAll），再将匿名函数赋值给这个变量。如果不分成两部，函数字面量无法与visitAll绑定，我们也无法递归调用该匿名函数。

### 捕获迭代变量

函数作为值进行传递时，其记录的是循环变量的内存地址，而不是循环变量某一时刻的值。例如：

```
package main

import (
	"os"
)

var tempDirs = []string{"temp1", "temp2", "temp3"}

func main() {
	var rmdir []func()
	for _, dir := range tempDirs {
		os.Mkdir(dir, 0755)
		rmdir = append(rmdir, func() {
			os.RemoveAll(dir)
		})
	}
	for _, f := range rmdir {
		f()
	}
}
```

这时，改代码执行完成后，依然存在前两个目录。这是因为，循环变量的作用域，在上面的程序中，for循环语句引入了新的词法块，循环变量dir在这个词法块中被声明。<font color=blue/>在该循环中生成的所有函数值都共享相同的循环变量函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值</font>。这时，随着循环的发生，dir对应的值也在变化，相应的，执行函数时，每次都是执行的对最后一个目录的删除操作。

不光上述是这样的，即使如下程序，同样是错误的：

```
package main

import (
	"os"
)

var tempDirs = []string{"temp1", "temp2", "temp3"}

func main() {
	var rmdir []func()
	for i := 0; i < len(tempDirs); i++ {
		os.Mkdir(tempDirs[i], 0755)
		rmdir = append(rmdir, func() {
			os.RemoveAll(tempDirs[i])
		})
	}
	for _, f := range rmdir {
		f()
	}
}
```

同样的原因，这里函数中保持的是局部变量i的地址，在执行完第一个for循环后，i即变成了4，此时在运行删除函数，会直接报错。

解决办法是为每一个函数中使用的值声明一个变量，这时，就不会再报错了，例如：

```
package main

import (
	"os"
)

var tempDirs = []string{"temp1", "temp2", "temp3"}

func main() {
	var rmdir []func()
	for i := 0; i < len(tempDirs); i++ {
		i := i
		os.Mkdir(tempDirs[i], 0755)
		rmdir = append(rmdir, func() {
			os.RemoveAll(tempDirs[i])
		})
	}
	for _, f := range rmdir {
		f()
	}
}
```



## 可变参数

参数数量可变的函数称为可变参数函数。在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号“...”，这表示该函数会接收任意数量的该类型参数。

```
func sum(vals...int) int {
    total := 0
    for _, val := range vals {
        total += val
    }
    return total
}

fmt.Println(sum())           // "0"
fmt.Println(sum(3))          // "3"
fmt.Println(sum(1, 2, 3, 4)) // "10"
```

在上面的代码中，调用者隐式的创建一个数组，并将原始参数复制到数组中，再把数组的一个切片作为参数传给被调用函数。如果原始参数已经是切片类型，我们该如何传递给sum？只需在最后一个参数后加上省略符。

```
values := []int{1, 2, 3, 4}
fmt.Println(sum(values...)) // "10"
```

```
func errorf(linenum int, format string, args ...interface{}) {
    fmt.Fprintf(os.Stderr, "Line %d: ", linenum)
    fmt.Fprintf(os.Stderr, format, args...)
    fmt.Fprintln(os.Stderr)
}
linenum, name := 12, "count"
errorf(linenum, "undefined: %s", name) // "Line 12: undefined: count"
```



## deferred函数

Go语言提供了独有的defer机制。在调用普通函数或方法前加上关键字defer，就完成了defer所需要的语法。当执行到该条语句时，函数和参数表达式得到计算，但直到包含该defer语句的函数（而不是defer包含的函数）执行完毕时，defer后的函数（defer包含的函数）才会被执行，不论包含defer语句的函数是通过return正常结束，还是由于panic导致的异常结束。你可以在一个函数中执行多条defer语句，它们的执行顺序与声明顺序相反。

defer语句经常被用于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。通过defer机制，不论函数逻辑多复杂，都能保证在任何执行路径下，资源被释放。例如：

```
func ReadFile(fileName string) ([]byte, error) {
	f, err := os.Open(fileName)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	return ioutil.ReadAll(f)
}
```

defer机制也常被用于记录何时进入和退出函数。例如

```
package main

import (
	"log"
	"time"
)

func main() {
	slowOperation()
}

func trace(msg string) func() {
	start := time.Now()
	log.Printf("enter %s\n", msg)
	return func() {
		log.Printf("exist %s (%s)\n", msg, time.Since((start)))
	}
}

func slowOperation() {
	defer trace("slowOperation")()
	time.Sleep(10 * time.Second)
}
```

这里，在执行slowOperation时，先defer了trace("slowOperation")()，首先会计算trace("slowOperation")，此时会返回一个匿名函数，该匿名函数会在函数返回时被执行。也就是前文所述：当执行到该条语句时，函数和参数表达式得到计算，但直到包含该defer语句的函数执行完毕时，defer后的函数才会被执行。

如果这里的defer trace("slowOperation")()少了后面的括号，则只会在函数退出时执行一次trace，返回的匿名函数并未使用。

被延迟执行的匿名函数甚至可以修改函数返回给调用者的返回值：

```
func triple(x int) (result int) {
    defer func() { result += x }()
    return double(x)
}
fmt.Println(triple(4)) // "12"
```

这里double是乘以2，不是C++的double。

在循环体中的defer语句需要特别注意，因为只有在函数执行完毕后，这些被延迟的函数才会执行。下面的代码会导致系统的文件描述符耗尽，因为在所有文件都被处理之前，没有文件会被关闭。

```
for _, filename := range filenames {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // NOTE: risky; could run out of file descriptors
    // ...process f…
}
```



## Panic异常

当panic异常发生时，程序会中断运行，并立即执行在该goroutine（可以先理解成线程，在第8章会详细介绍）中被延迟的函数（defer 机制）。随后，程序崩溃并输出日志信息。日志信息包括panic value和函数调用的堆栈跟踪信息。panic value通常是某种错误信息。对于每个goroutine，日志信息中都会有与之相对的，发生panic时的函数调用堆栈跟踪信息。

不是所有的panic异常都来自运行时，直接调用内置的panic函数也会引发panic异常；panic函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用panic。比如，当程序到达了某条逻辑上不可能到达的路径：

```Go
switch s := suit(drawCard()); s {
    case "Spades":                                // ...
    case "Hearts":                                // ...
    case "Diamonds":                              // ...
    case "Clubs":                                 // ...
    default:
        panic(fmt.Sprintf("invalid suit %q", s)) // Joker?
}
```

为了方便诊断问题，runtime包允许程序员输出堆栈信息。在下面的例子中，我们通过在main函数中延迟调用printStack输出堆栈信息。

```
package main

import (
	"fmt"
	"os"
	"runtime"
)

func main() {
	defer printStack()
	f1(3)
}
func f1(x int) {
	fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
	defer fmt.Printf("defer %d\n", x)
	f1(x - 1)
}

func printStack() {
	var buf [4096]byte
	n := runtime.Stack(buf[:], false)
	os.Stdout.Write(buf[:n])
}
```

程序输出为

```
f(3)
f(2)
f(1)
defer 1
defer 2
defer 3
goroutine 1 [running]:
main.printStack()
        /Users/wangyinkui/go/coding/main.go:21 +0x5b
panic(0x11a1800, 0x130a4f0)
        /usr/local/Cellar/go/1.15.6/libexec/src/runtime/panic.go:969 +0x1b9
main.f1(0x0)
        /Users/wangyinkui/go/coding/main.go:14 +0x1e5
main.f1(0x1)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.f1(0x2)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.f1(0x3)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.main()
        /Users/wangyinkui/go/coding/main.go:11 +0x4c
panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.f1(0x0)
        /Users/wangyinkui/go/coding/main.go:14 +0x1e5
main.f1(0x1)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.f1(0x2)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.f1(0x3)
        /Users/wangyinkui/go/coding/main.go:16 +0x185
main.main()
        /Users/wangyinkui/go/coding/main.go:11 +0x4c
```

在Go的panic机制中，延迟函数的调用在释放堆栈信息之前，因此runtime.Stack能输出已经被释放函数的信息。

## recover捕获异常

如果在deferred函数中调用了内置函数recover，并且定义该defer语句的函数发生了panic异常，recover会使程序从panic中恢复，并返回panic value。导致panic异常的函数不会继续运行，但能正常返回。在未发生panic时调用recover，recover会返回nil。

例如

```
func Parse(input string) (s *Syntax, err error) {
    defer func() {
        if p := recover(); p != nil {
            err = fmt.Errorf("internal error: %v", p)
        }
    }()
    // ...parser...
}
```

不加区分的恢复所有的panic异常，不是可取的做法，不应该试图去恢复其他包引起的panic。公有的API应该将函数的运行失败作为error返回，而不是panic。同样的，你也不应该恢复一个由他人开发的函数引起的panic，比如说调用者传入的回调函数，因为你无法确保这样做是安全的。



# 方法

在函数声明时，在其名字之前放上一个变量，即是一个方法。这个附加的参数会将该函数附加到这种类型上，即相当于为这种类型定义了一个独占的方法。例如：

```
type Point struct {
	X, Y float64
}

func (p Point) Distance(q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}

p := Point{1, 2}
q := Point{4, 6}
fmt.Println(p.Distance(q))
```

函数代码中附加的参数p，叫做方法的接收器。p.Distance的表达式叫做选择器，因为他会选择合适的对应p这个对象的Distance方法来执行。由于方法和字段都是在同一命名空间，所以如果我们在这里声明一个X方法的话，编译器会报错，因为在调用p.X时会有歧义。



## 基于指针的方法

当调用一个函数时，会对其每一个参数值进行拷贝，如果一个函数需要更新一个变量，或者函数的其中一个参数实在太大我们希望能够避免进行这种默认的拷贝，这种情况下我们就需要用到指针了。对应到我们这里用来更新接收器的对象的方法，当这个接受者变量本身比较大时，我们就可以用其指针而不是对象来声明方法，如下：

```
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

只有类型(Point)和指向他们的指针`(*Point)`，才可能是出现在接收器声明里的两种接收器。

在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个例子：

```go
type P *int
func (P) f() { /* ... */ } // compile error: invalid receiver type
```

想要调用指针的方法，有两种方式，第一：直接使用普通类型（非指针），编译器会自动转换成指针；第二：什么类型的指针，调用函数。例如

```
// 1
r := Point{1, 2}
r.ScaleBy(2)
fmt.Println(*r) // "{2, 4}"

// 2
r := &Point{1, 2}
r.ScaleBy(2)
fmt.Println(*r) // "{2, 4}"
```

### nil也是一个合法的接收器

就像一些函数允许nil指针作为参数一样，方法理论上也可以用nil指针作为其接收器，尤其当nil对于对象来说是合法的零值时，比如map或者slice。例如

```
// An IntList is a linked list of integers.
// A nil *IntList represents the empty list.
type IntList struct {
    Value int
    Tail  *IntList
}
// Sum returns the sum of the list elements.
func (list *IntList) Sum() int {
    if list == nil {
        return 0
    }
    return list.Value + list.Tail.Sum()
}
```

将方法看作普通函数，不过其中一个参数传参方式不一样更好理解。



## 通过嵌入结构体扩展方法

通过内嵌结构体，新的结构体可以直接调用内嵌的结构体的方法。例如：

```go
// Point is ....
type Point struct {
	X, Y float64
}

// Distance is ...
func Distance(p, q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}

// ScaleBy ...
func (p *Point) ScaleBy(factor float64) {
	p.X *= factor
	p.Y *= factor
}

// ColoredPoint ...
type ColoredPoint struct {
	Point
	Color color.RGBA
}


func main() {
	p := geometry.ColoredPoint{Point: geometry.Point{4, 6}}
	q := geometry.ColoredPoint{Point: geometry.Point{3, 7}}
	dis := p.Distance(q.Point)
	fmt.Println(dis)
  p.ScaleBy(2)
  fmt.Println(p.Point) // {8, 12}
}
```

内嵌字段会指导编译器去生成额外的包装方法来委托已经声明好的方法，和下面的形式是等价的：

```go
func (p ColoredPoint) Distance(q Point) float64 {
    return p.Point.Distance(q)
}

func (p *ColoredPoint) ScaleBy(factor float64) {
    p.Point.ScaleBy(factor)
}
```

在类型中内嵌的匿名字段也可能是一个命名类型的指针。

```
type ColoredPoint struct {
    *Point
    Color color.RGBA
}

p := ColoredPoint{&Point{1, 1}, red}
q := ColoredPoint{&Point{5, 4}, blue}
fmt.Println(p.Distance(*q.Point)) // "5"
q.Point = p.Point                 // p and q now share the same Point
p.ScaleBy(2)
fmt.Println(*p.Point, *q.Point) // "{2 2} {2 2}"
```

对于链表类数据比较有用。

一个struct可以有多个匿名字段。例如

```
type ColoredPoint struct {
    Point
    color.RGBA
}
```

当编译器解析一个选择器到方法时，比如p.ScaleBy，它会首先去找直接定义在这个类型里的ScaleBy方法，然后找被ColoredPoint的内嵌字段们引入的方法，然后去找Point和RGBA的内嵌字段引入的方法，然后一直递归向下找。如果选择器有二义性的话编译器会报错，比如你在同一级里有两个同名的方法。



## 方法值和方法表达式

选择一个方法，并且在同一个表达式里执行，比如常见的p.Distance()形式，实际上将其分成两步来执行也是可能的。p.Distance叫作“选择器”，选择器会返回一个方法"值"->一个将方法(Point.Distance)绑定到特定接收器变量的函数。这个函数可以不通过指定其接收器即可被调用；即调用时不需要指定接收器(译注：因为已经在前文中指定过了)，只要传入函数的参数即可：

```
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance        // method value
fmt.Println(distanceFromP(q))      // "5"
var origin Point                   // {0, 0}
fmt.Println(distanceFromP(origin)) // "2.23606797749979", sqrt(5)

scaleP := p.ScaleBy // method value
scaleP(2)           // p becomes (2, 4)
scaleP(3)           //      then (6, 12)
scaleP(10)          //      then (60, 120)
```



## 封装

一个对象的变量或者方法如果对调用方是不可见的话，一般就被定义为“封装”。

Go语言只有一种控制可见性的手段：大写首字母的标识符会从定义它们的包中被导出，小写字母的则不会。这种限制包内成员的方式同样适用于struct或者一个类型的方法。因而如果我们想要封装一个对象，我们必须将其定义为一个struct。

封装如前文的例子，并不是完全无法获取到，可以使用导出的函数对封装值进行操作。



# 接口

## 接口约定

接口类型是一种抽象的类型。它不会暴露出它所代表的对象的内部值的结构和这个对象支持的基础操作的集合；它们只会表现出它们自己的方法。也就是说当你有看到一个接口类型的值时，你不知道它是什么，唯一知道的就是可以通过它的方法来做什么。

之前我们使用fmt.Printf，它会把结果写到标准输出，和fmt.Sprintf，它会把结果以字符串的形式返回。这两个函数都使用了另一个函数fmt.Fprintf来进行封装。fmt.Fprintf这个函数对它的计算结果会被怎么使用是完全不知道的。

```
package fmt

func Fprintf(w io.Writer, format string, args ...interface{}) (int, error)
func Printf(format string, args ...interface{}) (int, error) {
    return Fprintf(os.Stdout, format, args...)
}
func Sprintf(format string, args ...interface{}) string {
    var buf bytes.Buffer
    Fprintf(&buf, format, args...)
    return buf.String()
}
```

其中Fprintf的前缀F表示文件(File)也表明格式化输出结果应该被写入第一个参数提供的文件中。在Printf函数中的第一个参数os.Stdout是`*os.File`类型；在Sprintf函数中的第一个参数&buf是一个指向可以写入字节的内存缓冲区，然而它 并不是一个文件类型尽管它在某种意义上和文件类型相似。

Fprintf函数中的第一个参数也不是一个文件类型。它是io.Writer类型，这是一个接口类型定义如下：

```go
package io

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    // Write writes len(p) bytes from p to the underlying data stream.
    // It returns the number of bytes written from p (0 <= n <= len(p))
    // and any error encountered that caused the write to stop early.
    // Write must return a non-nil error if it returns n < len(p).
    // Write must not modify the slice data, even temporarily.
    //
    // Implementations must not retain p.
    Write(p []byte) (n int, err error)
}
```

io.Writer类型定义了函数Fprintf和这个函数调用者之间的约定。一方面这个约定需要调用者提供具体类型的值就像`*os.File`和`*bytes.Buffer`，这些类型都有一个特定签名和行为的Write的函数。另一方面这个约定保证了Fprintf接受任何满足io.Writer接口的值都可以工作。Fprintf函数可能没有假定写入的是一个文件或是一段内存，而是写入一个可以调用Write函数的值。

因为fmt.Fprintf函数没有对具体操作的值做任何假设，而是仅仅通过io.Writer接口的约定来保证行为，所以第一个参数可以安全地传入一个只需要满足io.Writer接口的任意具体类型的值。一个类型可以自由地被另一个满足相同接口的类型替换，被称作可替换性(LSP里氏替换)。

例如

```go
package interfaceInfo

type ByteCounter int

func (c *ByteCounter) Write(p []byte) (int, error) {
	*c += ByteCounter(len(p))
	return len(p), nil
}


package main

import (
	"coding/interfaceInfo"
	"fmt"
)

func main() {
	var v interfaceInfo.ByteCounter
	v.Write([]byte("hello"))
	fmt.Println(v) // 5
}
```



## 接口类型

接口类型具体描述了一系列方法的集合，一个实现了这些方法的具体类型是这个接口类型的实例。（类似C++中的纯虚函数，不过可以进行实例化）。

io中定义了多个接口，例如

```
package io
type Reader interface {
    Read(p []byte) (n int, err error)
}
type Closer interface {
    Close() error
}
```

新的接口类型可以通过组合已有接口来定义。

```
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Writer
}
```



## 接口实现的条件

一个类型如果拥有一个接口需要的所有方法，那么这个类型就实现了这个接口。一个实现了接口的类型可以给接口类型赋值（反之不行），接口直接可以进行赋值，不过只能是从实现接口多的向实现接口少的赋值。例如

```
package interfaceInfo

import "fmt"

type BaseInter interface {
	One()
	Second()
}

type ParInter interface {
	One()
	Second()
	Thrid()
}

type BaseClass struct{}

func (b BaseClass) One() {
	fmt.Println("base one")
}

func (b BaseClass) Second() {
	fmt.Println("base second")
}

type ParClass struct{}

func (p ParClass) One() {
	fmt.Println("par one")
}

func (p ParClass) Second() {
	fmt.Println("par second")
}

func (p ParClass) Thrid() {
	fmt.Println("par thrid")
}

```

```go
package main

import (
	"coding/interfaceInfo"
)

func main() {
	var base interfaceInfo.BaseInter
	base = new(interfaceInfo.BaseClass)
	base.One() // base one
	base = new(interfaceInfo.ParClass)
	base.One() //par one

	var par interfaceInfo.ParInter
	par = new(interfaceInfo.BaseClass) // compile error: interfaceInfo.BaseClass lacks Third method
	par = new(interfaceInfo.ParClass)
	par.Thrid() //par thrid

	base = par
	partmp := par
	base.One() // par one
	partmp = base // missing method Thrid
	
	var baseclass interfaceInfo.BaseClass
	baseclass.One()
	baseclass = base // compiler error
}

```

在T类型的参数上调用一个`*T`的方法是合法的，只要这个参数是一个变量；编译器隐式的获取了它的地址。但这仅仅是一个语法糖：T类型的值不拥有所有`*T`指针的方法，这样它就可能只实现了更少的接口。（个人理解，从普通的类调用指针方法是OK的，但指针的方法不属于普通类，只属于指针）。

例如

```
type T struct{}

func (t T) One() {
	fmt.Println("T one")
}

func (t *T) Second() {
	fmt.Println("*T second")
}
```

```
func main() {
	var T interfaceInfo.T
	var _ interfaceInfo.BaseInter = T // compile error: interfaceInfo.BaseClass lacks Second method

	var v interfaceInfo.BaseInter
	v = &interfaceInfo.T{} // true
	v.One() //T one
	v.Second() // *T second
}
```

interface{}被称为空接口，因为空接口类型对实现它的类型没有要求，所以我们可以将任意一个值赋给空接口类型。

```go
var any interface{}
any = true
any = 12.34
any = "hello"
any = map[string]int{"one": 1}
any = new(bytes.Buffer)
```

我们当然不能直接对它持有的值做操作，因为interface{}没有任何方法。我们会在7.10章中学到一种用类型断言来获取interface{}中值的方法。



## flag.Value接口

flag.Value帮助命令行标记定义新的符号。其中定义为

```
type Value interface {
	String() string
	Set(string) error
}
```

其存在两个函数。String方法格式化标记的值用在命令行帮组消息中；这样每一个flag.Value也是一个fmt.Stringer。Set方法解析它的字符串参数并且更新标记变量的值。

在调用flag.Parser时，会读取命令行参数，如果存在设置的参数值，则会调用Set函数进行赋值。

flag的整体的处理逻辑为：首先注册需要关注的命令行参数（注册一个flag.Value类），之后解析命令行，对关注的参数进行赋值。以之前的温度转换举例：

```
type CelsiusFlag struct{ Celsius }

func (f *CelsiusFlag) Set(s string) error {
	var unit string
	var value float64
	fmt.Sscanf(s, "%f%s", &value, &unit)
	switch unit {
	case "C", "°C":
		f.Celsius = Celsius(value)
		return nil
	case "F", "°F":
		f.Celsius = FToC(Fahrenheit(value))
		return nil
	}
	return fmt.Errorf("invalid temperature %q", s)
}
```

由于CelsiusFlag内嵌了Celsius，之前已经实现过String，因此为了实现flag.Value只需要实现Set方法即可。如上。

之后调用flag.value的注册方法。

```
func CelsiusFlags(name string, value Celsius, usage string) *Celsius {
	f := CelsiusFlag{value}
	flag.CommandLine.Var(&f, name, usage)
	return &f.Celsius
}
```

其中flag.CommandLine.Var是注册flag.Value方法，其参数为Var(value Value, name string, usage string)。具体可以看源码。

之后我们就可以在主函数中使用CelsiusFlags进行注册，在命令行中更改值。如下：

```
package main

import (
	"coding/tempcov"
	"flag"
	"fmt"
)

var temp = tempcov.CelsiusFlags("temp", 20.0, "the temperature")

func main() {
	flag.Parse()
	fmt.Println(*temp)
}

```

```
$ go build gopl.io/ch7/tempflag
$ ./tempflag
20°C
$ ./tempflag -temp -18C
-18°C
$ ./tempflag -temp 212°F
100°C
```



## 接口值

概念上讲一个接口的值，接口值，由两个部分组成，一个具体的类型和那个类型的值。它们被称为接口的动态类型和动态值。类型是编译期的概念；因此一个类型不是一个值。在我们的概念模型中，一些提供每个类型信息的值被称为类型描述符，比如类型的名称和方法。在一个接口值中，类型部分代表与之相关类型的描述符。

如下w得到了3个值（开始的值和最后值是一样的）

```
var w io.Writer
w = os.Stdout
w = new(bytes.Buffer)
w = nil
```

在Go语言中，变量总是被一个定义明确的值初始化，即使接口类型也不例外。对于一个接口的零值就是它的类型和值的部分都是nil。

```
var w io.Writer
```

一个接口值基于它的动态类型被描述为空或非空，所以这是一个空的接口值。你可以通过使用w==nil或者w!=nil来判断接口值是否为空。调用一个空接口值上的任意方法都会产生panic:

```go
w.Write([]byte("hello")) // panic: nil pointer dereference
```

第二个语句将一个`*os.File`类型的值赋给变量w:

```go
w = os.Stdout
```

这个赋值过程调用了一个具体类型到接口类型的隐式转换，这和显式的使用io.Writer(os.Stdout)是等价的。这类转换不管是显式的还是隐式的，都会刻画出操作到的类型和值。

通常在编译期，我们不知道接口值的动态类型是什么，所以一个接口上的调用必须使用动态分配。因为不是直接进行调用，所以编译器必须把代码生成在类型描述符的方法Write上，然后间接调用那个地址。这个调用的接收者是一个接口动态值的拷贝。例如

```
w.Write([]byte("hello")) // writes "hello" to the bytes.Buffers
```

这次类型描述符是*bytes.Buffer，所以调用了(*bytes.Buffer).Write方法，并且接收者是该缓冲区的地址。这个调用把字符串“hello”添加到缓冲区中。

一个接口值可以持有任意大的动态值。例如，表示时间实例的time.Time类型，这个类型有几个对外不公开的字段。我们从它上面创建一个接口值,

```go
var x interface{} = time.Now()
```

接口值可以使用==和!＝来进行比较。两个接口值相等仅当它们都是nil值，或者它们的动态类型相同并且动态值也根据这个动态类型的==操作相等。因为接口值是可比较的，所以它们可以用在map的键或者作为switch语句的操作数。然而，如果两个接口值的动态类型相同，但是这个动态类型是不可比较的（比如切片），将它们进行比较就会失败并且panic。

当我们处理错误或者调试的过程中，得知接口值的动态类型是非常有帮助的。所以我们使用fmt包的%T动作:

```go
var w io.Writer
fmt.Printf("%T\n", w) // "<nil>"
w = os.Stdout
fmt.Printf("%T\n", w) // "*os.File"
w = new(bytes.Buffer)
fmt.Printf("%T\n", w) // "*bytes.Buffer"
```

在fmt包内部，使用反射来获取接口动态类型的名称。

### 包含nil指针的接口不是nil接口

一个不包含任何值的nil接口值和一个刚好包含nil指针的接口值是不同的。例如

```
const debug = true

func main() {
    var buf *bytes.Buffer
    if debug {
        buf = new(bytes.Buffer) // enable collection of output
    }
    f(buf) // NOTE: subtly incorrect!
    if debug {
        // ...use buf...
    }
}

// If out is non-nil, output will be written to it.
func f(out io.Writer) {
    // ...do something...
    if out != nil {
        out.Write([]byte("done!\n"))
    }
}
```

当main函数调用函数f时，它给f函数的out参数赋了一个*bytes.Buffer的空指针，所以out的动态值是nil。然而，它的动态类型是*bytes.Buffer，意思就是out变量是一个包含空指针值的非空接口，所以防御性检查out!=nil的结果依然是true。

动态分配机制依然决定(*bytes.Buffer).Write的方法会被调用，但是这次的接收者的值是nil。对于一些如*os.File的类型，nil是一个有效的接收者，但是*bytes.Buffer类型不在这些种类中。这个方法会被调用，但是当它尝试去获取缓冲区时会发生panic。

解决方案就是将main函数中的变量buf的类型改为io.Writer，因此可以避免一开始就将一个不完整的值赋值给这个接口：

```go
var buf io.Writer
if debug {
    buf = new(bytes.Buffer) // enable collection of output
}
f(buf) // OK
```



## sort.Interface接口

sort.Interface接口是用来进行排序的，实现改接口只需要实现如下三个函数

```
package sort

type Interface interface {
    Len() int
    Less(i, j int) bool // i, j are indices of sequence elements
    Swap(i, j int)
}
```

其中Len是长度，Less是小于判断，Swap是交换。例如

```
type StringSlice []string
func (p StringSlice) Len() int           { return len(p) }
func (p StringSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p StringSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
```

要想实现倒序，可以使用sort.Reverse。例如

```
sort.Sort(sort.Reverse(byArtist(tracks)))
```

其中sort.Reverse函数将排序顺序转换成逆序，其实现是，新生成一个sort.Interface。如下

```
package sort

type reverse struct{ Interface } // that is, sort.Interface

func (r reverse) Less(i, j int) bool { return r.Interface.Less(j, i) }

func Reverse(data Interface) Interface { return reverse{data} }
```

由于使用的是内嵌方式，只需要对Less函数进行重写即可。



## http.Handler接口

http.Handler接口定义如下

```
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```

ListenAndServe函数需要一个例如“localhost:8000”的服务器地址，和一个所有请求都可以分派的Handler接口实例。它会一直运行，直到这个服务因为一个错误而失败（或者启动失败），它的返回值一定是一个非空的错误。例如：

```
package httpServer

import (
	"fmt"
	"net/http"
)

type dollars float32

func (d dollars) String() string { return fmt.Sprintf("$%.2f", d) }

type Database map[string]dollars

func (db Database) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	for item, price := range db {
		fmt.Fprintf(w, "%s: %s\n", item, price)
	}
}
```

```
package main

import (
	"coding/httpServer"
	"io"
	"net/http"
)

const debug = false

func main() {
	db := httpServer.Database{"shoes": 50, "socks": 5}
	http.ListenAndServe("localhost:8888", db)
}
```

启动后，执行：

```
$ curl http://localhost:8888
shoes: $50.00
socks: $5.00
```

之后可以指定url。例如

```
func (db Database) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/list":
		for item, price := range db {
			fmt.Fprintf(w, "%s: %s\n", item, price)
		}
	case "/price":
		item := req.URL.Query().Get("item")
		price, ok := db[item]
		if !ok {
			w.WriteHeader(http.StatusNotFound) // 404
			fmt.Fprintf(w, "no such item: %q\n", item)
			return
		}
		fmt.Fprintf(w, "%s\n", price)
	default:
		w.WriteHeader(http.StatusNotFound) // 404
		fmt.Fprintf(w, "no such page: %s\n", req.URL)
	}
}
```

请求

```
$ curl http://localhost:8888/price?item=socks
$5.00
```

net/http包提供了一个请求多路器ServeMux来简化URL和handlers的联系。一个ServeMux将一批http.Handler聚集到一个单一的http.Handler中。

下面创建一个ServeMux并且使用它将URL和相应处理/list和/price操作的handler联系起来，这些操作逻辑都已经被分到不同的方法中。然后我门在调用ListenAndServe函数中使用ServeMux为主要的handler。

```
func main() {
    db := database{"shoes": 50, "socks": 5}
    mux := http.NewServeMux()
    mux.Handle("/list", http.HandlerFunc(db.list))
    mux.Handle("/price", http.HandlerFunc(db.price))
    log.Fatal(http.ListenAndServe("localhost:8000", mux))
}

type database map[string]dollars

func (db database) list(w http.ResponseWriter, req *http.Request) {
    for item, price := range db {
        fmt.Fprintf(w, "%s: %s\n", item, price)
    }
}

func (db database) price(w http.ResponseWriter, req *http.Request) {
    item := req.URL.Query().Get("item")
    price, ok := db[item]
    if !ok {
        w.WriteHeader(http.StatusNotFound) // 404
        fmt.Fprintf(w, "no such item: %q\n", item)
        return
    }
    fmt.Fprintf(w, "%s\n", price)
}
```

这里，由于mux.Handle的第二个参数也要求是handler，因此不能直接传递db.list函数，而需要进行一个转换，这里http.HandlerFunc并不是一个函数，而是一个转换类，其实现如下

```
package http

type HandlerFunc func(w ResponseWriter, r *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

HandlerFunc本身是一个方法（函数）,但其依然有自己的ServeHTTP方法（尽管调用的是自身）。通过这个方式，其符合handler接口。

因为handler通过这种方式注册非常普遍，ServeMux有一个方便的HandleFunc方法，它帮我们简化handler注册代码成这样：

```
mux.HandleFunc("/list", db.list)
mux.HandleFunc("/price", db.price)
```

注意：web服务器在一个新的协程中调用每一个handler，所以当handler获取其它协程或者这个handler本身的其它请求也可以访问到变量时，一定要使用预防措施，比如锁机制。

## error接口

error类型是一个接口，其定义如下

```
type error interface {
    Error() string
}
```

创建一个error最简单的方法就是调用errors.New函数，它会根据传入的错误信息返回一个新的error。整个errors包仅只有4行：

```go
package errors

func New(text string) error { return &errorString{text} }

type errorString struct { text string }

func (e *errorString) Error() string { return e.text }
```

有一个方便的封装函数fmt.Errorf，它还会处理字符串格式化。我们曾多次在第5章中用到它。

```go
package fmt

import "errors"

func Errorf(format string, args ...interface{}) error {
    return errors.New(Sprintf(format, args...))
}
```



## 类型断言

类型断言是一个使用在接口值上的操作。语法上它看起来像x.(T)被称为断言类型，这里x表示一个接口的类型和T表示一个类型。一个类型断言检查它操作对象的动态类型是否和断言的类型匹配。

有两种情况：第一种，如果断言的类型T是一个具体类型，然后类型断言检查x的动态类型是否和T相同。如果这个检查成功了，类型断言的结果是x的动态值，当然它的类型是T。换句话说，具体类型的类型断言从它的操作对象中获得具体的值。如果检查失败，接下来这个操作会抛出panic。

例如：

```
var w io.Writer
w = os.Stdout
f := w.(*os.File)      // success: f == os.Stdout
c := w.(*bytes.Buffer) // panic: interface holds *os.File, not *bytes.Buffer
```

第二种，如果断言的类型T是一个接口类型，然后类型断言检查是否x的动态类型满足T。如果这个检查成功了，动态值没有获取到；这个结果仍然是一个有相同动态类型和值部分的接口值，但是结果为类型T。换句话说，对一个接口类型的类型断言改变了类型的表述方式，改变了可以获取的方法集合（通常更大），但是它保留了接口值内部的动态类型和值的部分。例如

```
package main

import (
	"fmt"
	"io"
	"os"
)

const debug = false

func main() {
	var w io.Writer
	w = os.Stdin
	w.Write([]byte("hello\n")) // hello
	os.Stdin.Write([]byte("hello\n")) //hello
	rw := w.(io.ReadWriter)
	fmt.Printf("%T\n", w) // *os.File
	fmt.Printf("%T\n", rw) // *os.File
}
```

w和rw都持有os.Stdin，因此它们都有一个动态类型`*os.File`，但是变量w是一个io.Writer类型，只对外公开了文件的Write方法，而rw变量还公开了它的Read方法。但实际，w也可以正常执行写操作。

如果断言操作的对象是一个nil接口值，那么不论被断言的类型是什么这个类型断言都会失败。

如果类型断言出现在一个预期有两个结果的赋值操作中，例如如下的定义，这个操作不会在失败的时候发生panic，但是替代地返回一个额外的第二个结果，这个结果是一个标识成功与否的布尔值：

```go
var w io.Writer = os.Stdout
f, ok := w.(*os.File)      // success:  ok, f == os.Stdout
b, ok := w.(*bytes.Buffer) // failure: !ok, b == nil
```

当类型断言的操作对象是一个变量，你有时会看见原来的变量名重用而不是声明一个新的本地变量名，这个重用的变量原来的值会被覆盖（理解：其实是声明了一个同名的新的本地变量，外层原来的w不会被改变），如下面这样：

```go
if w, ok := w.(*os.File); ok {
    // ...use w...
}
```



## 通过类型断言查询接口

下面这段逻辑和net/http包中web服务器负责写入HTTP头字段：

```
func writeHeader(w io.Writer, contentType string) error {
    if _, err := w.Write([]byte("Content-Type: ")); err != nil {
        return err
    }
    if _, err := w.Write([]byte(contentType)); err != nil {
        return err
    }
    // ...
}
```

由于使用Write中写入字符串，需要使用切片，这时需要一个[]byte(...)进行转换。这个转换分配内存并且做一个拷贝，但是这个拷贝在转换后几乎立马就被丢弃掉。让我们假装这是一个web服务器的核心部分并且我们的性能分析表示这个内存分配使服务器的速度变慢。这里我们可以避免掉内存分配么？

net/http包中的内幕，我们知道在这个程序中的w变量持有的动态类型也有一个允许字符串高效写入的WriteString方法；这个方法会避免去分配一个临时的拷贝。（许多满足io.Writer接口的重要类型同时也有WriteString方法，包括`*bytes.Buffer`，`*os.File`和`*bufio.Writer`。）。

不能对任意io.Writer类型的变量w，假设它也拥有WriteString方法。但是我们可以定义一个只有这个方法的新接口并且使用类型断言来检测是否w的动态类型满足这个新接口。

```
// writeString writes s to w.
// If w has a WriteString method, it is invoked instead of w.Write.
func writeString(w io.Writer, s string) (n int, err error) {
    type stringWriter interface {
        WriteString(string) (n int, err error)
    }
    if sw, ok := w.(stringWriter); ok {
        return sw.WriteString(s) // avoid a copy
    }
    return w.Write([]byte(s)) // allocate temporary copy
}

func writeHeader(w io.Writer, contentType string) error {
    if _, err := writeString(w, "Content-Type: "); err != nil {
        return err
    }
    if _, err := writeString(w, contentType); err != nil {
        return err
    }
    // ...
}
```

fmt.Fprintf函数怎么从其它所有值中区分满足error或者fmt.Stringer接口的值。在fmt.Fprintf内部，有一个将单个操作对象转换成一个字符串的步骤，像下面这样：

```go
package fmt

func formatOneValue(x interface{}) string {
    if err, ok := x.(error); ok {
        return err.Error()
    }
    if str, ok := x.(Stringer); ok {
        return str.String()
    }
    // ...all other types...
}
```

我们可以使用类型断言来决定执行代码，例如sql请求中，对于参数处理

```
func sqlQuote(x interface{}) string {
    switch x := x.(type) {
    case nil:
        return "NULL"
    case int, uint:
        return fmt.Sprintf("%d", x) // x has type interface{} here.
    case bool:
        if x {
            return "TRUE"
        }
        return "FALSE"
    case string:
        return sqlQuoteString(x) // (not shown)
    default:
        panic(fmt.Sprintf("unexpected type %T: %v", x, x))
    }
}
```



# Goroutines和Channels

## Goroutines

在go语言中，每一个并发的执行单元叫作一个goroutine。如果程序中包含多个goroutine，对两个函数的调用则可能发生在同一时刻。以简单地把goroutine类比作一个线程。

当一个程序启动时，其主函数即在一个单独的goroutine中运行，我们叫它main goroutine。新的goroutine会用go语句来创建。在语法上，go语句是一个普通的函数或方法调用前加上关键字go。go语句会使其语句中的函数在一个新创建的goroutine中运行。而go语句本身会迅速地完成。

例如，main goroutine将计算菲波那契数列的第45个元素值。由于计算函数使用低效的递归，所以会运行相当长时间，在此期间我们想让用户看到一个可见的标识来表明程序依然在正常运行，所以来做一个动画的小图标：

```
package main

import (
	"fmt"
	"time"
)

func main() {
	go spinner(100 * time.Microsecond)
	const n = 45
	fibn := fib(n)
	fmt.Printf("\rFibonacci(%d) = %d\n", n, fibn)
}

func fib(x int) int {
	if x < 2 {
		return x
	}
	return fib(x-1) + fib(x-2)
}

func spinner(delay time.Duration) {
	for {
		for _, r := range `-\|/` {
			fmt.Printf("\r%c", r)
			time.Sleep(delay)
		}
	}
}

```

程序会有一个动画，之后会打印结果。

然后主函数返回。主函数返回时，所有的goroutine都会被直接打断，程序退出。除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个goroutine来打断另一个的执行，但是之后可以看到一种方式来实现这个目的，通过goroutine之间的通信来让一个goroutine请求其它的goroutine，并让被请求的goroutine自行结束执行。

由于没有执行锁的机制，因此程序有可能出现如下输出：

```
$ go run ./main.go 
Fibonacci(45) = 1134903170
-$
```



## 示例：并发的Clock服务

一个顺序执行的时钟服务器，它会每隔一秒钟将当前时间写到客户端：

```
package main

import (
	"io"
	"log"
	"net"
	"time"
)

func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err)
			continue
		}
		handleConn(conn)
	}
}

func handleConn(c net.Conn) {
	defer c.Close()
	for {
		_, err := io.WriteString(c, time.Now().Format("15:04:49\n"))
		if err != nil {
			return // 断开连接
		}
		time.Sleep(1 * time.Second)
	}
}

```

这里net.Listen创建监听端口，listener.Accept创建连接。handleConn(conn)处理连接，函数中定义defer c.Close()用于出现错误或者断开连接时关闭连接。

程序在两个窗口建立两条连接时：

```
// 窗口1                   //窗口2
nc localhost 8000         nc localhost 8000
13:32:329
13:32:329
13:32:329
^C                       
                         13:32:329
                         13:32:329
                         13:32:329
```

其无法处理并发连接。只需在请求handleConn(conn)前加上go即可处理并发请求。



## Channels

如果说goroutine是Go语言程序的并发体的话，那么channels则是它们之间的通信机制。一个channel是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。每个channel都有一个特殊的类型，也就是channels可发送数据的类型。一个可以发送int类型数据的channel一般写为chan int。

使用内置的make函数，我们可以创建一个channel：

```Go
ch := make(chan int) // ch has type 'chan int'
```

和map类似，channel也对应一个make创建的底层数据结构的引用。当我们复制一个channel或用于函数参数传递时，我们只是拷贝了一个channel引用，因此调用者和被调用者将引用同一个channel对象。和其它的引用类型一样，channel的零值也是nil。

一个channel有发送和接受两个主要操作，都是通信行为。一个发送语句将一个值从一个goroutine通过channel发送到另一个执行接收操作的goroutine。发送和接收两个操作都使用`<-`运算符。在发送语句中，`<-`运算符分割channel和要发送的值。在接收语句中，`<-`运算符写在channel对象之前。一个不使用接收结果的接收操作也是合法的。

```Go
ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```

Channel还支持close操作，用于关闭channel，随后对基于该channel的任何发送操作都将导致panic异常。对一个已经被close过的channel进行接收操作依然可以接受到之前已经成功发送的数据；如果channel中已经没有数据的话将产生一个零值的数据。没有办法直接测试一个channel是否被关闭，但是接收操作有一个变体形式：它多接收一个结果，多接收的第二个结果是一个布尔值ok，ture表示成功从channels接收到值，false表示channels已经被关闭并且里面没有值可接收。

使用内置的close函数就可以关闭一个channel：

```Go
close(ch)
```

以最简单方式调用make函数创建的是一个无缓存的channel，但是我们也可以指定第二个整型参数，对应channel的容量。如果channel的容量大于零，那么该channel就是带缓存的channel。

```Go
ch = make(chan int)    // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

### 无缓存的Channels

一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞，直到另一个goroutine在相同的Channels上执行接收操作，当发送的值通过Channels成功传输之后，两个goroutine可以继续执行后面的语句。反之，如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。

Channels也可以用于将多个goroutine连接在一起，一个Channel的输出作为下一个Channel的输入。这种串联的Channels就是所谓的管道（pipeline）。下面的程序用两个channels将三个goroutine串联起来，如图8.1所示。

![管道](https://docs.hacknode.org/gopl-zh/images/ch8-01.png)

其实现为

```
package main

import (
	"fmt"
)

func main() {
	input := make(chan int)
	output := make(chan int)

	go squares(input, output)
	go printValue(output)
	for i := 0; i < 100; i++ {
		input <- i
	}
	close(input)
}

func squares(input, output chan int) {
	for {
		value, ok := <-input
		if !ok {
			break
		}
		output <- value * value
	}
	close(output)
}

func printValue(value chan int) {
	for {
		value, ok := <-value
		if !ok {
			break
		}
		fmt.Println(value)
	}
	close(value)
}
```

使用range循环是上面处理模式的简洁语法，它依次从channel接收数据，当channel被关闭并且没有值可接收时跳出循环。上述代码改为：

```
package main

import (
	"fmt"
)

func main() {
	input := make(chan int)
	output := make(chan int)

	go squares(input, output)
	go printValue(output)
	for i := 0; i < 100; i++ {
		input <- i
	}
	close(input)
}

func squares(input, output chan int) {
	for i := range input {
		output <- i * i
	}
	close(output)
}

func printValue(value chan int) {
	for i := range value {
		fmt.Println(i)
	}
	close(value)
}

```

为了表明这种意图并防止被滥用，Go语言的类型系统提供了单方向的channel类型，分别用于只发送或只接收的channel。类型`chan<- int`表示一个只发送int的channel，只能发送不能接收。相反，类型`<-chan int`表示一个只接收int的channel，只能接收不能发送。（箭头`<-`和关键字chan的相对位置表明了channel的方向。）这种限制将在编译期检测。

chan int可以隐式转换成chan<- int或chan<- int。因为关闭操作只用于断言不再向channel发送新的数据，所以只有在发送者所在的goroutine才会调用close函数，因此对一个只接收的channel调用close将是一个编译错误。程序改为：

```
package main

import (
	"fmt"
)

func main() {
	input := make(chan int)
	output := make(chan int)

	go squares(input, output)
	go printValue(output)
	for i := 0; i < 100; i++ {
		input <- i
	}
	close(input)
}

func squares(input <-chan int, output chan<- int) {
	for i := range input {
		output <- i * i
	}
	close(output)
}

func printValue(value <-chan int) {
	for i := range value {
		fmt.Println(i)
	}
}

```

任何双向channel向单向channel变量的赋值操作都将导致该隐式转换。这里并没有反向转换的语法：也就是不能将一个类似`chan<- int`类型的单向型的channel转换为`chan int`类型的双向型的channel。

### sync.WaitGroup

有时，我们需要从goroutine中接收返回值，但是，我们可能并不知道是否所以的goroutine是否都已经执行完成了，这时就需要一个计数器，在增加一个goroutine是改值加1，在一个goroutine运行完成返回时，减一，等到最终值为0时，表示所以goroutine都运行完成了。（类似于c语言多线程中的屏障）。改类型为sync.WaitGroup。其使用示例如下，即对上述打印数据求和：

```
package main

import (
	"fmt"
	"sync"
)

func main() {
	result := make(chan int)
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go squares(i, result, &wg)
	}

	go func() {
		wg.Wait()
		close(result)
	}()

	var all int
	for i := range result {
		all += i
	}
	fmt.Printf("all is %d\n", all)
}

func squares(input int, result chan<- int, wg *sync.WaitGroup) {
	defer wg.Done()
	result <- input * input
}

```

这里wg.Add(1)是对改值进行加1操作，注意Add和Done方法的不对称。Add是为计数器加一，必须在worker goroutine开始之前调用，而不是在goroutine中；否则的话我们没办法确定Add是在"closer" goroutine调用Wait之前被调用（即，在调研wait时，无法保证每个子goroutine已经被执行了，如果add在goroutine中，则可能goroutine还没执行wait就返回了）。并且Add还有一个参数，但Done却没有任何参数；其实它和Add(-1)是等价的。wg.Wait()是判断是该值是否为0了，如果没有则阻塞直到为0，由于我们还要计算所有返回值，因此改方法应该在一个gorotune中执行，确认为0后时，reslut的计算也就完成了，此时关闭result，则main goroptinue中的reslut循环也就结束了。

### 带缓存的Channels

带缓存的Channel内部持有一个元素队列。队列的最大容量是在调用make函数创建channel时通过第二个参数指定的。下面的语句创建了一个可以持有三个字符串元素的带缓存Channel。下图ch变量对应的channel的图形表示形式。

```Go
ch = make(chan string, 3)
```

![img](https://docs.hacknode.org/gopl-zh/images/ch8-02.png)

向缓存Channel的发送操作就是向内部缓存队列的尾部插入元素，接收操作则是从队列的头部删除元素。如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个goroutine执行接收操作而释放了新的队列空间。相反，如果channel是空的，接收操作将阻塞直到有另一个goroutine执行发送操作而向队列插入元素。

在某些特殊情况下，程序可能需要知道channel内部缓存的容量，可以用内置的cap函数获取：

```Go
fmt.Println(cap(ch)) // "3"
```

同样，对于内置的len函数，如果传入的是channel，那么将返回channel内部缓存队列中有效元素的个数。因为在并发程序中该信息会随着接收操作而失效，但是它对某些故障诊断和性能优化会有帮助。

```Go
fmt.Println(len(ch)) // "2"
```



## 基于select的多路复用

考虑如下场景，我们存在两个并发的goroutine，在main goroutine要同时监听分别从这两个goroutine发送的chan。这时，如果采用之前的方式，顺序监听，例如

```go
for {
  x,ok := <-chan1
  y,ok := <-chan2
}
```

此时会出现问题是，当chan1未返回信息时，就会阻塞，chan2返回的数据我们也接收不到了。这时就需要使用select多路复用了。其语法如下：

```
select {
case <-ch1:
    // ...
case x := <-ch2:
    // ...use x...
case ch3 <- y:
    // ...
default:
    // ...
}

```

和switch语句稍微有点相似，也会有几个case和最后的default选择分支。每一个case代表一个通信操作（在某个channel上进行发送或者接收），并且会包含一些语句组成的一个语句块。一个接收表达式可能只包含接收表达式自身（译注：不把接收到的值赋值给变量什么的），就像上面的第一个case，或者包含在一个简短的变量声明中，像第二个case里一样；第二种形式让你能够引用接收到的值。

select会等待case中有能够执行的case时去执行。当条件满足时，select才会去通信并执行case之后的语句；这时候其它通信是不会执行的。一个没有任何case的select语句写作select{}，会永远地等待下去。

典型的运用例子如下，一个倒计时程序，time.Tick函数返回一个channel，程序会周期性地像一个节拍器一样向这个channel发送事件。每一个事件的值是一个时间戳，在用户输入回车时，倒计时会立即停止，并结束程序：

```
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	abort := make(chan struct{})
	go func() {
		os.Stdin.Read(make([]byte, 1))
		abort <- struct{}{}
	}()
	tick := time.Tick(1 * time.Second)

	for i := 10; i > 0; i-- {
		select {
		case <-tick:
			fmt.Printf("\r%d ", i)
		case <-abort:
		  fmt.Println("STOP!!!")
			return
		}
	}
}
```

time.Tick函数表现得好像它创建了一个在循环中调用time.Sleep的goroutine，每次被唤醒时发送一个事件。当触发了停止监听（即跳出for循环时），它会停止从tick中接收事件，但是ticker这个goroutine还依然存活，继续徒劳地尝试向channel中发送值，然而这时候已经没有其它的goroutine会从该channel中接收值了--这被称为goroutine泄露。

Tick函数挺方便，但是只有当程序整个生命周期都需要这个时间时我们使用它才比较合适。否则的话，我们应该使用下面的这种模式：

```go
ticker := time.NewTicker(1 * time.Second)
<-ticker.C    // receive from the ticker's channel
ticker.Stop() // cause the ticker's goroutine to terminate
```

这样，我们就可以在触发停止监听时，关闭定时的chan。

如果多个case同时就绪时，select会随机地选择一个执行，这样来保证每一个channel都有平等的被select的机会。例如如下例子：

```
package main

import (
	"fmt"
)

func main() {
	ch := make(chan int, 1)

	for i := 1; i < 10; i++ {
		select {
		case x := <-ch:
			fmt.Printf("%d ", x) // 1 3 5 7
		case ch <- i:
		}
	}
	fmt.Println()
}
```

ch缓存大小为1，所以会交替的为空或为满，所以只有一个case可以进行下去。第一次执行时，由于ch是空的，因此x := <-ch未就绪，ch <- i就绪，第二次执行，因为ch已经满了，因此x := <-ch就绪，ch <- i未就绪，以此循环。如果ch的buffer不是1，返回结果就是不一定的了。

有时候我们希望能够从channel中发送或者接收值，并避免因为发送或者接收导致的阻塞，尤其是当channel没有准备好写或者读时。select语句就可以实现这样的功能。select会有一个default来设置当其它的操作都不能够马上被处理时程序需要执行哪些逻辑。

下面的select语句会在abort channel中有值时，从其中接收值；无值时什么都不做。这是一个非阻塞的接收操作；反复地做这样的操作叫做“轮询channel”。

```go
select {
case <-abort:
    fmt.Printf("Launch aborted!\n")
    return
default:
    // do nothing
}
```

channel的零值是nil。也许会让你觉得比较奇怪，nil的channel有时候也是有一些用处的。因为对一个nil的channel发送和接收操作会永远阻塞，在select语句中操作nil的channel永远都不会被select到。

这使得我们可以用nil来激活或者禁用case，来达成处理其它输入或输出事件时超时和取消的逻辑。我们会在下一节中看到一个例子。



## 示例：并发遍历目录

创建一个程序来生成指定目录的硬盘使用情况报告。存在两个goroutine，一个用于深度优先搜索，遍历目录的所有文件，主goroutine进行统计。由于运行时间会过长，因此在遍历的同时，增加一个参数，来控制是否展示当前进度。

```
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"time"
)

var verbose = flag.Bool("v", false, "show verbose progress messages")

func main() {
	flag.Parse()
	roots := flag.Args()
	if len(roots) == 0 {
		roots = []string{"."}
	}
	fileSizes := make(chan int64)
	go func() {
		for _, root := range roots {
			walkDir(root, fileSizes)
		}
		close(fileSizes)
	}()

	var tick <-chan time.Time
	if *verbose {
		tick = time.Tick(500 * time.Millisecond)
	}
	var nfiles, nbytes int64
loop:
	for {
		select {
		case size, ok := <-fileSizes:
			if !ok {
				break loop
			}
			nfiles++
			nbytes += size
		case <-tick:
			printDiskUsage(nfiles, nbytes)
		}
	}
	printDiskUsage(nfiles, nbytes)
}

func walkDir(dir string, fileSizes chan<- int64) {
	for _, entry := range dirents(dir) {
		if entry.IsDir() {
			subdir := filepath.Join(dir, entry.Name())
			walkDir(subdir, fileSizes)
		} else {
			fileSizes <- entry.Size()
		}
	}
}

func dirents(dir string) []os.FileInfo {
	entries, err := ioutil.ReadDir(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "du1: %v\n", err)
	}
	return entries
}

func printDiskUsage(nfiles, nbytes int64) {
	fmt.Printf("%d files  %.1f GB\n", nfiles, float64(nbytes)/1e9)
}
```

只有在调用时提供了-v的flag才会显示程序进度信息，如果-v的flag在运行时没有传入的话，tick这个channel会保持为nil，这样在select里的case也就相当于被禁用了。

从上式可以看出，对于一个已经关闭的chan（fileSizes），对其读取操作也是就绪的。

上式依然执行比较慢，对每一个walkDir的调用创建一个新的goroutine来加速。使用sync.WaitGroup来计数，以决定何时关闭fileSizes。但如果每个递归调用walkDir都创建goroutine，则会导致创建成百上千的goroutine，我们需要修改dirents函数，用计数信号量来阻止他同时打开太多的文件。更改后，代码如下：

```
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"sync"
	"time"
)

var sema = make(chan struct{}, 20)

var verbose = flag.Bool("v", false, "show verbose progress messages")

func main() {
	flag.Parse()
	roots := flag.Args()
	if len(roots) == 0 {
		roots = []string{"."}
	}
	var n sync.WaitGroup
	fileSizes := make(chan int64)
	go func() {
		for _, root := range roots {
			n.Add(1)
			go walkDir(root, fileSizes, &n)
		}
	}()

	go func() {
		n.Wait()
		close(fileSizes)
	}()

	var tick <-chan time.Time
	if *verbose {
		tick = time.Tick(500 * time.Millisecond)
	}
	var nfiles, nbytes int64
loop:
	for {
		select {
		case size, ok := <-fileSizes:
			if !ok {
				break loop
			}
			nfiles++
			nbytes += size
		case <-tick:
			printDiskUsage(nfiles, nbytes)
		}
	}
	printDiskUsage(nfiles, nbytes)
}

func walkDir(dir string, fileSizes chan<- int64, n *sync.WaitGroup) {
	defer n.Done()
	for _, entry := range dirents(dir) {
		if entry.IsDir() {
			subdir := filepath.Join(dir, entry.Name())
			n.Add(1)
			go walkDir(subdir, fileSizes, n)
		} else {
			fileSizes <- entry.Size()
		}
	}
}

func dirents(dir string) []os.FileInfo {
	sema <- struct{}{}
	defer func() { <-sema }()
	entries, err := ioutil.ReadDir(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "du1: %v\n", err)
	}
	return entries
}

func printDiskUsage(nfiles, nbytes int64) {
	fmt.Printf("%d files  %.1f GB\n", nfiles, float64(nbytes)/1e9)
}
```

加速明显。

