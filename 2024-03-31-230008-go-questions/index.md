# go相关问题


## 概述

> 以下内容为ChatGPT生成



### 关于golang 的 gmp 相关问题，怎样回答比较简洁



在面试中，如果面试官问到关于Go语言的GMP模型相关的问题，你可以从以下几个方面简洁回答：

1. **GMP是什么**：GMP是Go语言运行时(runtime)调度器的核心模型，代表Goroutine、M（machine或者说是工作线程）、P（processor或者说是逻辑处理器）。它是Go语言并发性的基础，确保了高效的goroutine调度。

2. **Goroutine（G）**：Goroutine是Go语言中的轻量级线程，由Go运行时管理。Goroutines比系统线程更轻量，启动更快，且在运行时由Go的调度器进行调度，不是由操作系统直接调度。

3. **M（Machine）**：M代表系统的物理线程，每个M都绑定一个操作系统级别的线程。M负责执行G中的代码，每个M在执行过程中可能会与不同的Goroutines关联。

4. **P（Processor）**：P代表逻辑处理器，它的数量决定了同时有多少Goroutines可以并发执行。每个P都有自己的本地运行队列，P的数量通常由GOMAXPROCS环境变量控制，默认值为机器的CPU核心数。P是连接M和G的桥梁，每个M在执行Goroutine之前，需要先获取一个P。

5. **调度过程简述**：当一个Goroutine需要运行时，它会被放入到一个P的本地运行队列中。如果该P已经绑定了一个M，则该M会从队列中取出Goroutine来执行。如果所有的P都在忙，额外的Goroutine会被放入全局队列中，等待空闲的P。当M因为Goroutine阻塞（如系统调用）而空闲时，它会尝试从其它P的本地队列或全局队列中偷取Goroutine来执行，以保持高效的利用CPU资源。

通过上述几点，可以简洁且全面地回答面试官关于Go语言GMP模型的问题。



### 关于golang 的 内存模型 相关问题，怎样回答比较简洁

在面试中，当被问及Go语言的内存模型时，可以简洁地从以下几个核心点来回答：

1. **并发模型**：Go语言采用CSP（Communicating Sequential Processes）并发模型，通过Goroutines（轻量级线程）和Channels（通道）来实现并发编程。Goroutines在运行时被多个线程调度执行，而Channels则用于Goroutines之间的安全通信。

2. **内存分配**：Go语言使用一个高效的垃圾回收机制来管理内存。新对象通常通过`new`函数或复合字面量分配，而内存分配则由Go的运行时进行管理。Go运行时包括一个垃圾收集器，它可以自动释放不再使用的内存。

3. **栈与堆**：Go语言运行时会为每个Goroutine分配一个独立的栈，这个栈可以根据需要动态地伸缩。栈用于存储局部变量，如果编译器无法确定变量的生命周期，或者变量太大，它会被分配在堆上。堆上的对象生命周期由垃圾收集器管理。

4. **内存模型**：Go的内存模型定义了在多线程程序中如何同步访问共享数据。Go内存模型强调，不同的Goroutines之间通过通道（Channels）、锁（如sync.Mutex）或原子操作（如sync/atomic包中的函数）来进行数据同步，以保证内存访问的一致性和可见性。

5. **逃逸分析**：Go编译器执行逃逸分析来决定变量是分配在栈上还是堆上。如果一个函数内部创建的对象在函数外部仍然可达，则认为此对象“逃逸”到堆上，因此需要通过垃圾收集器来管理。

简而言之，Go语言的内存模型旨在简化并发编程，通过Goroutines和Channels提供了一种高效的方式来处理并发，同时通过垃圾回收、逃逸分析等机制来优化内存管理。



### 详解内存分配

Go语言的内存分配机制是高效而独特的，它旨在提供快速的分配速度和低延迟的垃圾回收。Go的内存分配可以从以下几个方面来详细解释：

1. **堆和栈**

首先，理解Go中的堆（heap）和栈（stack）是重要的。栈用于存储函数的局部变量，每个Goroutine都有自己的栈，这个栈的大小可以动态调整。相比之下，堆用于存放那些可能在函数调用结束后还需要继续存在的对象，或者是那些太大而不能放在栈上的对象。

2. **内存分配过程**

- **小对象分配**：Go使用固定大小的内存块，称为“页”（pages），来管理堆内存。~~对于小对象的分配，Go使用了一种称为`Tcache`（线程缓存）的机制，每个P（代表处理器，是Go运行时的调度单位）都有自己的一组`Tcache`，用于存储小对象。这减少了对全局堆的锁竞争，从而加速了内存分配。~~

- **大对象分配**：对于大对象，Go直接从堆上分配内存。大对象的分配会触发更多的堆管理操作，包括可能的内存页映射（从操作系统请求更多的内存）。

3. **逃逸分析**

Go编译器会进行逃逸分析来决定变量应该分配在栈还是堆上。如果一个变量在函数返回后还能被访问，那么这个变量就会“逃逸”到堆上。逃逸分析有助于减少垃圾回收的压力，因为栈上的内存可以在函数返回时立即回收。

4. **垃圾回收**

Go的垃圾回收器是并发的，采用标记清除（Mark-Sweep）算法。它在运行时并行于用户的Goroutines，以减少停顿时间。垃圾回收过程主要分为两个阶段：标记阶段，它识别出所有活动对象；清除阶段，它回收所有未被标记的对象内存。

- **三色标记算法**：Go的垃圾回收器使用三色标记算法，通过黑色、灰色、白色三种颜色来标记对象的状态，以有效地进行垃圾回收，同时减少对程序执行的干扰。

- **写屏障**：为了减少标记过程中的停顿时间，Go使用了写屏障（write barrier），这是一种同步机制，用于在垃圾回收期间处理堆上对象的写操作。

总结

Go语言的内存分配旨在提供高效的并发执行和低延迟的垃圾回收。通过逃逸分析优化内存使用，以及采用并发垃圾回收算法，Go尝试平衡性能和内存安全之间的关系，使得开发者能够更加专注于业务逻辑的实现。





### 关于golang 的 make 和 new 

在面试中，当面试官问到关于Go语言中的`make`和`new`的区别时，可以简洁地回答如下：

1. **作用对象不同**：
   - `new(T)`用于为T类型的新项分配零值内存空间，并返回其地址，即一个*T类型的值。适用于所有类型，包括基本类型、数组、结构体等。
   - `make(T, args)`仅用于初始化内置的引用类型，即切片(slice)、映射(map)和通道(chan)，并返回类型T的初始化值（非零值）。它不仅分配内存，还初始化内部的数据结构，比如切片的长度和容量。

2. **返回值不同**：
   - `new`返回的是指针，指向新分配的类型T的零值。
   - `make`返回的是初始化后的T类型的实例，这个实例已经准备好使用，不是指针。

3. **用途不同**：
   - 使用`new`时，通常是想分配一个值的内存，该值将被初始化为零值。
   - 使用`make`时，意味着除了分配内存，还需要进行一些初始化操作，比如指定切片的长度和容量，这样这个切片就可以直接使用了。

简单示例：

- 使用`new`分配内存给一个整数，并返回指向这个零值的指针：
  ```go
  ptr := new(int) // ptr的类型是*int，指向的值是0
  ```

- 使用`make`创建一个长度为0，容量为10的切片：
  ```go
  s := make([]int, 0, 10) // s的类型是[]int，是一个长度为0，容量为10的切片
  ```

这样的回答能够覆盖`make`和`new`的基本区别和用法，对于面试是一个简洁且全面的回答。



### 关于golang的零值

在Go语言中，零值（Zero Value）是变量在声明后自动赋予的初始值，即在没有明确初始化的情况下的默认值。每种类型的零值都是确定的，这意味着不同类型的变量会有不同的零值。Go语言的这一设计可以减少未初始化变量导致的错误，因为每个变量总是有一个初始的、确定的值。

以下是Go语言中几种主要类型的零值：

1. **布尔型（`bool`）**：零值为`false`。
2. **整型（`int`、`int8`、`int16`、`int32`、`int64`等）**：零值为`0`。
3. **浮点型（`float32`、`float64`）**：零值为`0.0`。
4. **复数型（`complex64`、`complex128`）**：零值为`0 + 0i`。
5. **字符串（`string`）**：零值为空字符串`""`。
6. **指针（`*T`）**：零值为`nil`，表示没有指向任何对象。
7. **切片（`slice`）**：零值为`nil`，表示没有分配内存。
8. **映射（`map`）**：零值为`nil`，表示没有分配内存，需要通过`make`函数初始化。
9. **通道（`chan`）**：零值为`nil`，表示没有分配内存，需要通过`make`函数初始化。
10. **接口（`interface`）**：零值为`nil`，表示没有绑定具体的类型和值。
11. **结构体（`struct`）**：每个成员都被初始化为其类型的零值。

下面是一个Go语言的示例，展示了不同类型变量的零值：

```go
package main

import "fmt"

func main() {
	//var a int
	a := new(int)
	//var b float64
	b := new(float64)
	//var c bool
	c := new(bool)
	//var d string
	d := new(string)
	//var e *int
	e := new(*int)
	//var f []int
	f := new([]int)
	//var g map[string]int
	g := new(map[string]int)
	//var h chan int
	h := new(chan int)
	//var i interface{}
	i := new(interface{})
	//var j struct {
	//	x int
	//	y float64
	//}
	j := new(struct {
		x int
		y float64
	})

	fmt.Println("int zero value:", *a)
	fmt.Println("float64 zero value:", *b)
	fmt.Println("bool zero value:", *c)
	fmt.Println("string zero value:", *d)
	fmt.Println("*int zero value:", *e)
	fmt.Println("slice zero value:", *f)
	fmt.Println("map zero value:", *g)
	fmt.Println("chan zero value:", *h)
	fmt.Println("interface zero value:", *i)
	fmt.Println("struct zero value:", *j)
}
int zero value: 0
float64 zero value: 0
bool zero value: false
string zero value: 
*int zero value: <nil>
slice zero value: []
map zero value: map[]
chan zero value: <nil>
interface zero value: <nil>
struct zero value: {0 0}

```



```go
package main

import "fmt"

func main() {
    var a int
    var b float64
    var c bool
    var d string
    var e *int
    var f []int
    var g map[string]int
    var h chan int
    var i interface{}
    var j struct {
        x int
        y float64
    }

    fmt.Println("int zero value:", a)
    fmt.Println("float64 zero value:", b)
    fmt.Println("bool zero value:", c)
    fmt.Println("string zero value:", d)
    fmt.Println("*int zero value:", e)
    fmt.Println("slice zero value:", f)
    fmt.Println("map zero value:", g)
    fmt.Println("chan zero value:", h)
    fmt.Println("interface zero value:", i)
    fmt.Println("struct zero value:", j)
}

int zero value: 0
float64 zero value: 0
bool zero value: false
string zero value:
*int zero value: <nil>
slice zero value: []
map zero value: map[]
chan zero value: <nil>
interface zero value: <nil>
struct zero value: {0 0}
```

注意，对于切片、映射和通道，虽然它们的零值是`nil`，但它们在使用前通常需要通过`make`函数进行初始化，因为`nil`切片、映射和通道在使用时会有限制，例如不能向`nil`映射添加元素。

https://goplay.tools/snippet/TB1Ib-8sHZI

https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/

`make` 关键字的作用是创建切片、哈希表和 Channel 等内置的数据结构，而 `new` 的作用是为类型申请一片内存空间，并返回指向这片内存的指针。

## 小结


