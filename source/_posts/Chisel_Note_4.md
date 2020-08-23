---
title: Chisel学习笔记（四）：组件
date: 2020/08/23 15:00:00
tag:
	- Chisel
catagory:
	- Chisel
	- 笔记
---

## Chisel模块

在Chisel中，每一个硬件模块继承`Module`类，并且拥有一个名为`io`的域。IO被Bundle所包裹，端口的方向通过调用Input()或者Output()来声明。

以下是声明两个组件的代码

```scala
class CompA extends Module {
	val io = IO(new Bundle {
		val a = Input(UInt(8.W))
		val b = Input(UInt(8.W))
		val x = Output(UInt(8.W))
		val y = Output(UInt(8.W))
    })
}

class CompB extends Module {
	val io = IO(new Bundle {
		val in1 = Input(UInt(8.W))
		val in2 = Input(UInt(8.W))
		val out = Output(UInt(8.W))
    })
}
```

以下是在顶层模块中实例化两个部件，并进行连线的代码。

```scala
class CompC extends Module {
	val io = IO(new Bundle {
		val in_a = Input(UInt(8.W))
		val in_b = Input(UInt(8.W))
		val in_c = Input(UInt(8.W))
		val out_x = Output(UInt(8.W))
		val out_y = Output(UInt(8.W))
	})
	val compA = Module(new CompA())
	val compB = Module(new CompB())
	
	compA.io.a := io.in_a
	compA.io.b := io.in_b
	io.out_x := compA.io.x
	
	compB.io.in1 := compA.io.y
	compB.io.in2 := io.in_c
	io.out_y := compB.io.out
}
```

组件通过`new`关键字来创建，并且需要被包裹在一个对于`Module`的调用中。

**对于组件的引用存储在一个本地变量，例如，compA中**，可以通过io域来访问组件的IO端口。

## 实例：ALU

```scala
import chisel3.util._
class Alu extends Module {
	val io = IO(new Bundle{
		val a = Input(UInt(16.W))
		val b = Input(UInt(16.W))
		val fn = Input(UInt(2.W))
		val y = Output(UInt(16.W))
	})
	io.y := 0.U // 需要赋初值
	switch(io.fn) {
		is(0.U) { io.y := io.a + io.b }
		// .................
	}
}
```

这里使用了`switch/is`表达式来表示case。

## 批量端口连接

我们可以使用`<>`运算符进行批量端口连接。Chisel会自动对那些同名的、方向相反的端口进行连接，如果没有同名端口，那么则不会被连接。

例如，流水线处理器中的几个阶段：

```scala
class Fetch extends Module {
	val io = IO(new Bundle {
		val instr = Output(UInt(32.W))
		val pc = Output(UInt(32.W))
	})
	// Fetch
}
```

```scala
class Decode extends Module {
	val io = IO(new Bundle {
		val instr = Input(UInt(32.W))
		val pc = Input(UInt(32.W))
		val aluOp = Ouput(UInt(32.W))
	})
}
```

在顶层模块对他们进行连接的时候，可以：

```scala
class cpu extends Module {
	val io = IO( ..... )
	val ifu = Module(new Fetch())
	val dec = Module(new Decode())
	ifu.io <> dec.io	// 完成同名端口的连接
}
```

在实际编写代码的时候，我们为了清晰起见，可以使用`src_dst`的后缀来编码信号。

## 使用函数生成轻量化的模块

某些硬件模块 ，除了参数变动，或是连线变动之外，不会发生其他的改变，这时候我们可以使用函数返回相应的硬件模块。

比Verilog的参数化更强大的是，函数可以以实际的硬件信号为输入，这样，就免去了声明模块-实例化模块-连线的过程。

例如，我们定义一个加法器。

```scala
def adder ( x: UInt, y: UInt ) = {
	x + y
}
```

我们可以非常轻松地使用`adder`函数创建，并连线两个加法器。

```scala
val x = adder(a,b) 
val y = adder(c,d)
```

应该可以注意到，这个函数是一个硬件生成器。在调用函数的过程中，并没有进行加法的运算，而是只是在函数里面创建了一个新的硬件节点，并对信号进行了连接。这个例子带来的收益并不明显，因为加法在Chisel中已经是一个重载过的运算符了。

函数中同样可以使用寄存器，函数被多次调用，会产生多个硬件的寄存器实例。

例如，我们声明一个`delay`函数，将某个信号延时一拍。

```scala
def delay(x: UInt) = RegNext(x)
```

我们可以嵌套调用`delay`函数，生成一个将信号延迟两拍的器件。

```scala
val delay_2cyc = delay(delay(delay_in))
```

**函数可以作为Module的一个部分被声明，然而，如果是在不同的Module中被使用的函数，最好被放在一个专门的`object`中。**