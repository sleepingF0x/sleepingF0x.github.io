# Golang的注意点

------

<h3>目录</h3>

[TOC]

------

## 1. 可以返回局部变量的指针

作为少数包含指针的语言，它与C还是有所不同。C中函数不能够返回局部变量的指针，因为函数结束时局部变量就会从栈中释放。而golang可以做到返回局部变量的一点

```
#include <iostream>

using namespace std;

int* get_some() {

int a = 1;

return &a;

}

int main() {

cout << "a = " << *get_some() << endl;

return 0;

}
```

*这个明显在c/c++中是错误的写法，a出栈后什么都没了。 **会发生一下错误：**

```
$ g++ t.cpp

> t.cpp: In function 'int* get_some()':

> t.cpp:4:6: warning: address of local variable 'a' > returned [-Wreturn-local-addr]

>  int a = 1;

      ^
```

go语言试验代码如下：

```
package main

import "fmt"

func GetSome() *int {

a := 1;

return &a;

}

func main() {

fmt.Printf("a = %d", *GetSome())

}
```

*基本相同的代码，但是有以下运行结果*

```
> $ go run t.go 

> a = 1 
```

显然不是go的编译器识别不出这个问题，而是在这个问题上做了优化。参考go FAQ的原文：

> How do I know whether a variable is allocated on the heap or the stack?

> From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.

> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.

> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

这里的意思就是让我们无需担心返回的指针是空悬指针。我理解的意思是，普通情况下函数中局部变量会存储在堆栈中，但是如果这个局部变量过大的话编译器可能会选择将其存储在堆中，这样会更加有意义。还有一种情况，当编译器无法证明在函数结束后变量不被引用那么就会将变量分配到垃圾收集堆上。总结一句：**编译器会进行分析后决定局部变量分配在栈还是堆中**

嗯。。。利用这个特性我们可以使用以下方式来达到并发做某事的作用

```
func SomeFun() <-chan int {

    out := make(chan int)

    go func() {

        //做一些不可告人的事情。。。

    }()

    return out

}
```

## 2. Go提供的两种分配原语——内建函数new和make

Go语言提供了两种分配的原语，即内建函数new和make。它们做的事情不同。

1. new它不会**初始化**内存，而是将**内存置零**。也就是说new(T)会为类型T的新项分配一个已置零的内存空间,并返回它的地址，也就是*T。即它会返回一个指针，这个指针是指向这个类型T的零值的那份空间。
2. make的函数签名make(T, args)。它仅用于切片、map和chan类型的创建。make会直接返回一个类型为T的**值**而非**指针**,当然这个值是已初始化过的。用法已切片为例，例如：

```
make([]int, 10, 100)
```

会分配一个容量为100，长度为10的int类型的切片结构。

```
new([]int)
```

这个会返回一个指向新分配得，已置零得切片结构，即指向nil切片值的指针。

下面例子阐明了new和make之间的区别：

```
var p *[]int = new([]int)      //分配切片结构；*p = nil;基本没用

var v []int = make([]int, 100)  //切片v现在引用了一个具有100个int元素的新数组

//没必要这么麻烦

var p *[]int = new([]int)

*p = make([]int, 100, 100)

//习惯用法

v := make([]int, 100)
```

**记住**，make只适用于map、切片和chan且不返回指针。若要获得明确的指针，请使用new分配内存

## 3. 复合字面

在os标准包中有以下代码，这个函数相当于其他语言中的构造函数

```
func NewFile(fd int, name string) *File {

    if fd < 0 {

        return nil

    }

    f := new(File)

    f.fd = fd

    f.name = name

    f.dirinfo = nil

    f.nepipe = 0

    return f

}
```

这里显得代码显得过于冗长，可以使用复合字面来简化代码

```
func NewFile(fd int, name string) *File {

if fd < 0 {

return nil

}

f := File{fd, name, nil, 0}

return &f

}
```

由上面的代码可以知道复合字面<code>File{fd, name, nil, 0}</code>返回的是一个量的引用而非**指针**,所以最后返回时需要取地址符。

## 4. 基础类型中数组、切片、map、chan是值类型还是引用类型

这个是很有必要注意的一件事，用为当你让函数中传入一个数组时，能不能改变外部数值的值呢？这就要考验到数值类型是值类型还是引用类型了。如果是引用类型的话，相当于c语言中传入指针一样，可以在函数内部改变传入参数的外部的值，但是如果是值类型的话，在传入函数过程中只是将一份拷贝传入，故不可在函数内部修改外部的值。

go语言中的**数组是值类型的**，这与其他语言大不一样，拿c/c++为例：

```
#include <iostream>

using namespace std;

const int NUM = 5;

int a[NUM] = {5,4,3,2,1};

void change_a(int arr[],int n) {

        for(int i = 0; i < n; i++){

                arr[i]--;

        }

}

int main() {

        change_a(a,NUM);

        for(int i = 0; i < NUM; i++) {

                cout << "a[" << i << "] = " << a[i] << endl;

        }

}
```

**结果如下：**

```
Administrator@PC-201809211459 MINGW64 ~/Desktop

$ g++ v.cpp -o v.exe

Administrator@PC-201809211459 MINGW64 ~/Desktop

$ ./v.exe

a[0] = 4

a[1] = 3

a[2] = 2

a[3] = 1

a[4] = 0
```

很显然，函数内部改变了形参数组导致全局变量a数组发生了改变

下面是golang的代码

```
package main

import "fmt"

var a [5]int = [5]int{5,4,3,2,1}

func changeA(arr [5]int) {

        for i := 0; i < 5; i++ {

                arr[i]--

        }

}

func main() {

        changeA(a)

        for i,v := range a {

                fmt.Println("a[",i,"] = ",v)

        }

}
```

**结果如下：**

```
Administrator@PC-201809211459 MINGW64 ~/Desktop

$ go run v.go

a[ 0 ] =  5

a[ 1 ] =  4

a[ 2 ] =  3

a[ 3 ] =  2

a[ 4 ] =  1
```

从此可以看出go语言中的数组是值类型。事实上我们很少用数组去传参数，因为在 go中如果用数组传参的话需要在函数的参数形式列表中写死数组的大小，而这种情况在c/c++中是不需要的。

但是go中传参可以使用切片，因为**切片是引用类型的**。同上例子如下：

```
package main

import "fmt"

func main() {

        a := []int{5,4,3,2,1}

        func(arr []int) {

                for i := 0; i < len(arr); i++ {

                        arr[i]--

                }

        }(a)

        for i,v := range a {

                fmt.Println("a[",i,"] = ",v)

        }

}
```

**结果如下：**

```
Administrator@PC-201809211459 MINGW64 ~/Desktop

$ go run vv.go

a[ 0 ] =  4

a[ 1 ] =  3

a[ 2 ] =  2

a[ 3 ] =  1

a[ 4 ] =  0
```

由此可见**golang的数组是值类型的，但是切片是引用类型的。**

记下 []T{}、map、chan作为基础系统类型里的三个引用类型，而且这三个都是可以使用make这个内联函数的。第七点会归纳这部分内联函数

## 5.初始化函数init

这个函数比较神奇啊，我看官方文档的时候有些看不懂.官方文档（镜像网站上的中文官方文档）的一段

> 最后，每个源文件都可以通过定义自己的无参数 init 函数来设置一些必要的状态。 （其实每个文件都可以拥有多个 init 函数。）而它的结束就意味着初始化结束： 只有该包中的所有变量声明都通过它们的初始化器求值后 init 才会被调用， 而那些 init 只有在所有已导入的包都被初始化后才会被求值。

> 除了那些不能被表示成声明的初始化外，init 函数还常被用在程序真正开始执行前，检验或校正程序的状态。

我在试验了之后大概得出结论，在import某个包的会执行该包下所有文件的init函数，执行顺序与文件在文件系统的排序有关。

**vv.go**,main函数所在文件

```
package main

import(

"fmt"

"./some"

_ "./another"

)

func init() {

fmt.Println("hello")

}

func main() {

a := []int{5,4,3,2,1}

some.ChangeA(a)

for i,v := range a {

fmt.Println("a[",i,"] = ",v)

    }

var c int = 10

fmt.Println(c)

}
```

package some下有三个文件

some0.go

```
package some

import "fmt"

func init() {

fmt.Println("some0")

}
```

some1.go

```
package some

import (

"fmt"

)

var a int

func init(){

a = 10

fmt.Println("package some init done!",a)

}

func ChangeA(arr []int) {

for i := 0; i < len(arr); i++ {

arr[i]--

}

}
```

some2.go

```
package some

import "fmt"

func init() {

fmt.Println("some2")

}
```

package another下一个文件

another.go

```
package another

import "fmt"

func init() {

fmt.Println("package another init done!")

}
```

以上代码为了节省空间，某些为了美观的空行省略了。

最后运行结果如下：

```
some0

package some init done! 10

some2

package another init done!

hello

a[ 0 ] =  4

a[ 1 ] =  3

a[ 2 ] =  2

a[ 3 ] =  1

a[ 4 ] =  0

10
```

假如将another.go的文件小做修改，修改如下：

```
package another

import "fmt"

import _ "../some"

func init() {

fmt.Println("package another init done!")

}
```

得到的结果不变，可见，init函数只会执行一遍，而不是碰到import它所在的包就执行。

## 6.关于指针与值

这个我也是比较糊的，所以在这里进行了部分整理和试验。估计以后还有更多关于这点的问题

先把**重要点**记下：

1. 绑定在类型指针*T上的方法可以改变该类型的值，但是只绑定在类型T上的方法是无法改变该类型的值的。有以下代码：

```
package main

import "fmt"

type Si int

func (s *Si)Plus1(a Si) {

*s += a

}

func (s Si)Plus2(a Si) {

s += a

}

func main() {

var s Si = 10

s.Plus1(1)

fmt.Println("after Plus1: s = ",s)

s.Plus2(1)

fmt.Println("after Plus2: s = ",s)

}
```

运行结构如下：

```
after Plus1: s =  11

after Plus2: s =  11
```

可见Plus2并没有发挥其作用。

go语言是一门一眼就能看得懂的语言，其他语言中把成员函数神奇的封装在一个类里，但go不是，函数在前面的小括号里写的参数就是指定了我这个函数是归属于哪个类型的，而且显式的将该类型的值传入函数了，也就是函数名前面的括号其实就可以看作是形参列表

1. 类型向接口赋值的时候应该取地址

```
package main

import "fmt"

type Si int

type Plus interface {

Plus1(a Si)

Plus2(a Si)

}

func (s *Si)Plus1(a Si) {

*s += a

}

func (s Si)Plus2(a Si) {

s += a

}

func main() {

    var s Si = 10

var ss Plus = &s

ss.Plus1(1)

fmt.Println("after Plus1: s = ",s)

}
```

以上代码是成立并且是能正确运行的

但是如果将上面的<code>var ss Plus = &s</code>变成<code>var ss Plus = s</code>就会出现编译错误。该编译错误如下：

```
# command-line-arguments

cmd\tt.go:26:6: cannot use s (type Si) as type Plus in assignment:

Si does not implement Plus (Plus1 method has pointer receiver)
```

这个编译错误提示很有意思啊，前半段提醒我们并没有实现Plus接口，我们可能会认为Si类型明明实现了Plus接口啊。这是怎么回事呢？其实看括号里的话结合最上面未出错的程序就会明白，其实编译器的意思就是*Si实现了接口Plus但是Si并没有实现。

为什么会出现这种情况呢，其实是因为接口有一个函数绑定在指针上<code>func (s *Si)Plus1(a Si)</code>，而Si类型是没有实现这个函数的，故没有实现Plus接口。可是为什么<code>func (s Si)Plus1(a Si)</code>绑定在Si上但是*Si也实现了Plus接口。那是因为go编译器可以自动根据<code>func (s Si)Plus1(s Si)</code>这个函数生成<code>func (s *Si)Plus1(a Si)</code>，故而*Si实现了所有函数。

当然以上自动生成的过程反过来是无法实现的，因为指针的权限大的原因，<code>func (s *Si)Plus1(a Si)</code>可能会改变s的值，而<code>func (s Si)Plus1(a Si)</code>无法做到,故而编译器也不会自动生成。

通过以上分析，我们以以下例子做试验：

```
package main

import "fmt"

type Si int

type Plus interface {

Plus1(a Si)

Plus2(a Si)

}

func (s Si)Plus1(a Si) {

s += a

}

func (s Si)Plus2(a Si) {

s += a

}

func main() {

var s Si = 10

var ss Plus = s

ss.Plus1(1)

fmt.Println("after Plus1: s = ",s)

}

运行结果：

after Plus1: s =  10
```

为什么是10在第一点已有介绍了。编译通过，第二点分析合理！

1. 根据以上例子发现了一个奇怪的但又不奇怪的现象。不论是*T还是T都可以直接调用函数。而且在对接口赋值时接口声明部分无需声明为指针也不能声明为指针。接口赋值好后，不能取内容，虽然有些接口看上去是一个指针。

这里不再举例

考虑到以上三点，我觉得自己有必要养成的几个习惯：

1.**接口赋值时应该最好使用类型指针对其赋值**

2.**写成员函数时遵循最小权限原则，注意\*T和T的区别**

3.**在调用成员函数时无论是指针还是类型本身都可以直接调用**

4.**接口在调用成员函数时就直接调用**

## 7. 切片、map和chan有关的内联函数

这一部分也是比较乱的一点，关联这三个基本类型的内建函数大致可以分成三类。分别与创建、删除、操作。

1. 创建：map、slice、channel的创建一般都是用make函数来进行内存分配。

2. 删除：delete主要用于map中删除实例。嗯，channel的close也放在这项中吧。

3. 操作：len、cap可用于不同的类型，len可用于string、slice、array的长度。cap一般返回slice的分配空间的大小。copy用于复制slice。append用于追加slice.

   ps：new用于各种类型的内存分配不止以上几种。

<h6>*具体用法如下*：</h6>

1. make

   1.1. channel: 这里只拿常用类型int做例子：

```
ch1 := make(chan int)        //不带缓存的channel

ch2 := make(chan int, 1024)    //带缓存的channel
1.2. slice: 针对slice的函数签名make([]type,len)和make([]type,len,cap)
slice1 := make([]int, 10)   //slice1中有10个初始值为零值的元素

slice2 := make([]int, 10, 100)  //slice2中有10个初始值为零的元素，且初始容量为100
1.3. map：签名make(map[keyType]valueType)
mp := make(map[string]int)

mp["啊啊啊"] = 3
```

1. append

   2.1. slice: append(slice []Type, elems ...Type) []Type

   append函数主要用于向slice的末尾添加元素的，作为一个特殊的存在可以在字节切片【】byte("hello")中添加字符串string。它会返回一个被更新过的slice，如果要使用它就需要一个变量接收这个更新的值。例子如下：

```
slice1 = append(slice1,2,3,4)

slice2 = append(slice2,slice1...)

slice3 := append([]byte("hello"),"world"...)
```

1. copy

   3.1. slice: copy(dst, src []Type) int

   这个函数需要小心的一点，slice1和slice2两个长度分别为5和3.还是用代码表示吧。。。

```
//len(slice1)是5

//len(slice2)是3

//i==3,只会复制slice1的前三个元素到slice2中

i := copy(slice2,slice1)

//i==3，只会将slice2中的前三个元素复制到slice1中

i = copy(slice1,slice2)
```

1. len、cap

   4.1. len用于获取切片和map长度,channel未取元素个数，cap用于获取切片的容量和channel的缓冲容量。签名：len(v Type) int，cap(v Type) int

```
len(ch1)

len(slice1)

len(mp)

cap(slice1)

cap(ch1)
```

len不只用于这三个数据类型，还包括string、数组和指向数组的指针。源代码注释如下：

> // The len built-in function returns the length of v, according to its type:

// Array: the number of elements in v.

// Pointer to array: the number of elements in *v (even if v is nil).

// Slice, or map: the number of elements in v; if v is nil, len(v) is zero.

// String: the number of bytes in v.

// Channel: the number of elements queued (unread) in the channel buffer;

// if v is nil, len(v) is zero.

cap虽然也可以用在数组和指向数据的指针但是其返回内容与len函数相同。源代注释如下：

> // The cap built-in function returns the capacity of v, according to its type:

// Array: the number of elements in v (same as len(v)).

// Pointer to array: the number of elements in *v (same as len(v)).

// Slice: the maximum length the slice can reach when resliced;

// if v is nil, cap(v) is zero.

// Channel: the channel buffer capacity, in units of elements;

// if v is nil, cap(v) is zero.

1. delete

   5.1. delete 只用于map删除元素。签名：delete(m map[Type]Type1, key Type)

```
delete(mp,"啊啊啊")
```

1. close

   6.1. channel可以接受和发送数据，也可以被关闭。当channel关闭后向channel发送数据的操作会引起panic。但是当channel关闭后，我们还能向其中取数据，若是之前的数据还没有取完那么还可以将这些数据取出。当缓存的数据全部取完后，仍然可以对channel取数据，此时的数据为零值数据。签名：close(c chan<- Type)

```
close(ch)
```

