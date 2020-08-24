---
title: Chisel学习笔记（七）：硬件生成器
date: 2020/08/24 14:00:00
tag:
	- Chisel
catagory:
	- Chisel
	- 笔记
---

Chisel的一大优势，就是不仅可以编写硬件描述代码，还可以灵活地编写硬件生成器，即“生成硬件”的代码。再其他硬件描述语言，例如Verilog中 ，笔者比较经常使用Python来生成一些繁复的代码，例如端口连线、实例化等。

## 硬件模块参数化

参数可以是非常普通的Scala整形常量，也可以是Chisel的硬件类型。

### 简单的参数

- 对电路定制最简单的参数化方法是使用参数来定义位宽。

- 参数可以作为Chisel模块的构造器参数传入类的构造函数中。

下面是一个可定制宽度的加法器的例子：

```scala
class ParamAdder(n: Int) extends Module {
	val io = IO(new Bundle{
		val a = Input(UInt(n.W))
		val b = Input(UInt(n.W))
		val c = Output(UInt(n.W))
	})
	io.c := io.a + io.b
}
val add_8 = Module(new ParamAdder(8))
```

为了保证鲁棒性，防止硬件和预期不同，一般需要在类的头部加上一个`require`语句，来对参数的合法性进行断言。

```scala
class ParameterizedWidthAdder(in0Width: Int, in1Width: Int, sumWidth: Int) extends Module {
  require(in0Width >= 0)
  require(in1Width >= 0)
  require(sumWidth >= 0)
  val io = IO(new Bundle {
    val in0 = Input(UInt(in0Width.W))
    val in1 = Input(UInt(in1Width.W))
    val sum = Output(UInt(sumWidth.W))
  })
  // a +& b includes the carry, a + b does not
  io.sum := io.in0 +& io.in1
}
```

### 将硬件的行为参数化

```scala
class Sort4(ascending: Boolean) extends Module {
  val io = IO(new Bundle {
    val in0 = Input(UInt(16.W))
    val in1 = Input(UInt(16.W))
    val in2 = Input(UInt(16.W))
    val in3 = Input(UInt(16.W))
    val out0 = Output(UInt(16.W))
    val out1 = Output(UInt(16.W))
    val out2 = Output(UInt(16.W))
    val out3 = Output(UInt(16.W))
  })
    
  // this comparison funtion decides < or > based on the module's parameterization
  def comp(l: UInt, r: UInt): Bool = {
      if (ascending) {
        l < r
      } else {
        l > r
    }
  }

  val row10 = Wire(UInt(16.W))
  val row11 = Wire(UInt(16.W))
  val row12 = Wire(UInt(16.W))
  val row13 = Wire(UInt(16.W))

  when(comp(io.in0, io.in1)) {
    row10 := io.in0            // preserve first two elements
    row11 := io.in1
  }.otherwise {
    row10 := io.in1            // swap first two elements
    row11 := io.in0
  }
```

在上述例子中，我们看到，硬件的行为被参数化了，`comp(UInt, UInt) => Bool`函数接收的是两个硬件的节点，返回的也是两个硬件的节点。

### 以类型为参数的函数

类型同样可以作为参数，传递入函数中。

以下的例子可以让Chisel生成对应数据类型的多路选择器。

```scala
def myMux[ T <: Data](sel: Bool, tPath: T, fPath: T): T = {
	val ret = WireDefault(fPath) 
	when(sel) {
		ret := tPath
	}
	ret				// 返回一个“硬件”
}
```

在上述的例子中，表达式中`[T <: Data]`定义了一个类型参数`T`，其中`T`应当是`Data`类或者`Data`的子类。`Data`是Chisel类型系统的根类型。

我们可以通过以下的方式获得一个多路选择器：

```scala
val resA = myMux(selA, 5.U, 10.U)
```

如果使用不同类型的参数，将会造成runtime error.

我们甚至可以传入一个复杂的Bundle作为多路选择器的选择值。

```scala
val tVal = Wire(new ComplexIO)
val fVal = Wire(new ComplexIO)
val resB = myMux(selB, tVal, fVal)
```

我们可以使用cloneType方法来获取某个数据的类型。

```
def myMux[ T <: Data](sel: Bool, tPath: T, fPath: T): T = {
	val ret = Wire(fPath.cloneType)		// 获取数据类型
    ret := fPath
	when(sel) {
		ret := tPath
	}
	ret				// 返回一个“硬件”
}
```

### 以类型为模块的参数

设想，我们要实现一个NOC中在不同处理器核之间移动数据的模块，我们并不会在路由接口中对其进行硬编码，而是将其参数化。除此之外，我们还将路由端口的数量进行了参数化。

```scala
class NoCRouter[ T <: Data](dt: T, n: Int) extends Module {
	val io = IO(new Bundle {
		val inPort = Input(Vec(n, dt))
		val address = Input(Vec(n, UInt(8.W)))
		val outPort = Output(Vec(n, dt))
	})
	// 根据地址进行路由
}
```

对于在不同的处理器之间传递的信息，我们使用一个`Payload`来代表。

```scala
class Payload extends Bundle {
	val data = UInt(16.W)
	val flag = Bool()
}
```

现在，我们可以根据此来创建一个新的路由模块了。

```scala
val router = Module(new NocRouter(new Payload, 2))
```

### 可参数化的Bundle

在上面的例子中，我们对于Payload使用了统一的Data类型Bool，进一步思考，是否可以将Data类型也进行参数化呢？

```scala
class Port[ T <: Data ](dt: T) extends Bundle {
	val address = UInt(8.W)
	val data = dt.cloneType
}
```

注意上面的`cloneType`方法，`Bundle`的参数`T`，应当是属于Chisel的`Data`类型的子类。在Bundle中，我们定义了一个`data`域，通过在参数上面应用`cloneType`方法，来定义数据类型。

> 以下语句摘录自Chisel-Book，用于备忘：
>
> However, when we use a constructor parameter, this parameter becomes a public field of the class. When Chisel needs to clone the type of the Bundle, e.g., when it is used in a Vec, this public field is in the way.

为了避免上述情况，可以使用以下的方法定义：

```scala
class Port[ T <: Data ](dt: T) extends Bundle {
	val address = UInt(8.W)
	val data = dt.cloneType
}
```

此时，我们可以定义路由模块了：

```scala
class NocRouter2[ T <: Data ](dt: T, n: Int) extends Module {
	val io = IO(new Bundle {
		val inPort = Input(Vec(n, dt))
		val outPort = Output(Vec(n, dt))
	})
}
```

实例化代码：

```scala
val router = Module(new NocRouter2(new Port(new Payload), 2))
```

### 可选的IO端口

常用于有时可以选择去除某些调试信号。

注意到如果val是None，那么就没有这个信号了。

```scala
class HalfFullAdder(val hasCarry: Boolean) extends Module {
  val io = IO(new Bundle {
    val a = Input(UInt(1.W))
    val b = Input(UInt(1.W))
    val carryIn = if (hasCarry) Some(Input(UInt(1.W))) else None
    val s = Output(UInt(1.W))
    val carryOut = Output(UInt(1.W))
  })
  val sum = io.a +& io.b +& io.carryIn.getOrElse(0.U)
  io.s := sum(0)
  io.carryOut := sum(1)
}
```

### 可选的参数

可以使用`Option`类型，来定义某个可选的参数，在实例化的时候，如果不提供这个参数，那么这个参数的`isDefined`字段就为`False`。

```scala
class DelayBy1(resetValue: Option[UInt] = None) extends Module {
    val io = IO(new Bundle {
        val in  = Input( UInt(16.W))
        val out = Output(UInt(16.W))
    })
    val reg = if (resetValue.isDefined) { // resetValue = Some(number)
        RegInit(resetValue.get)
    } else { //resetValue = None
        Reg(UInt())
    }
    reg := io.in
    io.out := reg
}

println(getVerilog(new DelayBy1))
println(getVerilog(new DelayBy1(Some(3.U))))
```



## 用代码生成组合逻辑

在Chisel中，我们可以通过从Scala的`Array`转为Chisel的`Vec`类型，非常方便地创建组合逻辑表格。我们可以使用存储在文件中的数据，在硬件生成阶段创建一个逻辑表：

```scala
import chisel3._
import scala.io.Source

class FileReader extends Module {
	val io = IO(new Bundle {
		val address = Input(UInt(8.W))
		val data = Output(UInt(8.W))
	})
	
	val array = new Array[Int](256)
	val idx = 0
	
	// 将文件的内容读入到scala的数组中
	val source = Source.fromFile("data.txt")
	for(line <- source.getLines()) {
		array(idx) = line.toInt
		idx += 1
	}
	// 将scala整数array转换成chisel的vec类型
	val table = VecInit(array.map(_.U(8.W)))
	// 使用索引访问Table
	io.data := table(io.address)
}
```

让我们把目光聚焦到这一行代码：

```scala
val table = VecInit(array.map(_.U(8.W)))
```

一个Scala的数组（Array）可以被隐式地转换成一个序列（Sequence），序列支持`map`函数。`map`函数的意义是，将函数应用到序列中的每一个对象，并返回经过处理后的序列。在上述代码中，`_.U(8.W)`是一个匿名函数，`_`是一个通配符，代表列表中的元素，这个函数将列表中的Scala Int类型转换为Chisel中硬件的`UInt`，位宽为8. 

有了这个方法，我们可以非常便捷的编码某些查找表逻辑，例如，二进制转BCD逻辑，等等。

## 使用继承

Chisel基于Scala，而Scala是一门面向对象的语言。因此，我们可以充分利用面向对象的特性，抽象出不同的硬件模块的共同行为，构造出一个父类。

在之前的学习中，我们已经构造了不同的计数器。假设现在有了新的场景，我们需要实现不同版本的计数器，来比较其资源消耗情况。

此时，我们需要先定义一个抽象类：

```scala
abstract class Ticker(n: Int) extends Module {
	val io = IO(new Bundle{
		val tick = Output(Bool())
	})
}
```

如果我们需要具体的实现某个Ticker，则需要继承这个抽象类：

```scala
class UpTicker (n: Int) extends Ticker(n) {
	val N = (n-1).U
	val cntReg = RegInit (0.U(8.W))
    cntReg := cntReg + 1.U
    when(cntReg === N) {
        cntReg := 0.U
    }
    io.tick := cntReg === N
}
```

测试代码的参数包括：

（1）类型：仅接收Ticker类型

（2）DUT：用以测试的代码

（3）期待得到tick的周期数

```scala
import chisel3.iotesters.PeekPokeTester
import org.scalatest._
class TickerTester[ T <: Ticker ]( dut: T, n: Int) extends
	PeekPokeTester(dut: T) {
		// -1 is the notion that we have not yet seen the first tick
		var count = -1
		for (i <- 0 to n * 3) {
			if (count > 0) {
			expect(dut.io.tick , 0)
		}
		if (count == 0) {
			expect(dut.io.tick , 1)
        }
        val t = peek(dut.io.tick)
        // On a tick we reset the tester counter to N-1,
        // otherwise we decrement the tester counter
        if (t == 1) {
        	count = n-1
        } else {
        	count -= 1
        }
       	step (1)
        }
}
```

## 隐式

Scala引入了隐式的概念，允许编译器引入部分语法糖。

### 隐式参数

有时，我们的代码可能需要访问一些顶层的变量，特别是在比较深的嵌套函数调用中。相比于我们手动将这些变量在每次函数调用中传递，我们可以使用隐式参数。

在Scala中，我们可以隐式或显式地传入参数：

```scala
object CatDog {
  implicit val numberOfCats: Int = 3
  //implicit val numberOfDogs: Int = 5

  def tooManyCats(nDogs: Int)(implicit nCats: Int): Boolean = nCats > nDogs
    
  val imp = tooManyCats(2)    // 隐式地传入了参数nCats 
  val exp = tooManyCats(2)(1) // 显式地传入了参数nCats 
}
CatDog.imp
CatDog.exp
```

此处，我们首先定义了一个**隐式值**`numberOfCats`。在某个定义域内，隐式值只能有一个值。

然后，我们定义了一个函数，接收两个参数列表，第一个是显式的参数值，第二个是隐式的参数值。

当我们调用`tooManyCats`函数的时候，我们可以隐藏第二个参数列表（让编译器为我们寻找隐式值），或是显式地提供相应的参数（可以和默认的隐式值不同）。

### 隐式转换

类似于隐式参数的，隐式转换常常被用于减少模板代码的数量。更具体地，它们用来**自动将Scala对象转换为其他对象**。

在下面的例子中，我们有两个类，分别为`Animal`和`Human`类，`Animal`有`Species`字段，但是`Human`没有。

当我们在`Human`上面调用`Species`时，编译器会尝试进行隐式转换。

因此，为了完成`Animal`到`Human`之间的转换，我们需要定义一个转换函数。

```scala
class Animal(val name: String, val species: String)
class Human(val name: String)
implicit def human2animal(h: Human): Animal = new Animal(h.name, "Homo sapiens")
val me = new Human("Adam")
println(me.species)
```

## 例子：Mealy机生成器

```scala
// Mealy machine has
case class BinaryMealyParams(
  // number of states
  nStates: Int,
  // initial state
  s0: Int,
  // function describing state transition
  stateTransition: (Int, Boolean) => Int,
  // function describing output
  output: (Int, Boolean) => Int
) {
  require(nStates >= 0)
  require(s0 < nStates && s0 >= 0)
}

class BinaryMealy(val mp: BinaryMealyParams) extends Module {
  val io = IO(new Bundle {
    val in = Input(Bool())
    val out = Output(UInt())
  })

  val state = RegInit(UInt(), mp.s0.U)

  // output zero if no states
  io.out := 0.U
  for (i <- 0 until mp.nStates) {
    when (state === i.U) {
      when (io.in) {
        state  := mp.stateTransition(i, true).U
        io.out := mp.output(i, true).U
      }.otherwise {
        state  := mp.stateTransition(i, false).U
        io.out := mp.output(i, false).U
      }
    }
  }
}
```

上述的代码是一个Mealy状态机的生成逻辑。首先，声明了一个`case class`，在里面包装了所有构建Mealy机所需的参数，包括状态的数量、初态、状态转移函数和输出函数，并且使用了两个`require`语句做断言，保证状态的合法性。

`BinaryMealy`是状态机的硬件模块，其构造器接收参数`mp`，这个参数用以构建状态机。

![image-20200824210948423](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200824210948423.png)

下面构造这个状态机的参数：

```scala
val nStates = 3
val s0 = 2
// 将上述函数翻译成状态转移函数
def stateTransition(state: Int, in: Boolean): Int = {
	if(in) {
		1
	} else {
		0
	}
}
// 状态机输出函数
def output(state:Int, in: Boolean): Int = {
	if (state == 2) {
		0
	} else if ((state == 1 && !in) || (state == 0 && in)) {
		1
	} else {
		0
	}
}
val testParams = BinaryMealyParams(nStates, s0, stateTransition, output)
```

