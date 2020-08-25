---
title: Chisel学习笔记（十一）：函数式编程
date: 2020/08/25 19:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

## Scala中的函数式编程

函数，以多个值为输入，以单个值为输出。输入值通常叫做参数。如果函数没有输出，则返回`Unit`类型。

以下是函数定义的实例：

```scala
// 没有参数的函数，可以省略去括号和返回值类型（返回值类型可以是Unit）
def hello1(): Unit = print("Hello!")
def hello2 = print("Hello again!")

// Math operation: one input and one output.
def times2(x: Int): Int = 2 * x

// 参数可以有默认值，并且建议写明返回值类型
def timesN(x: Int, n: Int = 2) = n * x

// Call the functions listed above.
hello1()	// 没有参数，可以使用括号，也可以省略
hello2		// 调用的时候，如果原来定义时没有括号，则也不可以使用括号
times2(4)
timesN(4)         // no need to specify n to use the default value
timesN(4, 3)      // argument order is the same as the order where the function was defined
timesN(n=7, x=2)  // arguments may be reordered and assigned to explicitly
```

### 函数是一等公民

**函数**可以被赋值给某个`val`，并且传递给类、对象，或者作为某个参数传递给其他函数。

```scala
// 常规的函数定义
def plus1funct(x: Int): Int = x + 1
def times2funct(x: Int): Int = x * 2

// 函数字面量赋值给某个Val
val plus1val: Int => Int = x => x + 1
val times2val = (x: Int) => x * 2

// Calling both looks the same.
plus1funct(4)
plus1val(4)
plus1funct(x=4)
//plus1val(x=4) // 不可以这样调用
```

为什么需要有`val`定义的函数这种操作呢？使用`val`定义函数，那么函数就和往常的变量一样，可以在各个函数之间作为参数传递。我们甚至可以自己定义高阶函数。

### 定义高阶函数

```scala
// 首先定义自己的函数
val plus1 = (x: Int) => x + 1
val times2 = (x: Int) => x * 2

// 传递给Map，在List上面调用
val myList = List(1, 2, 5, 9)
val myListPlus = myList.map(plus1)
val myListTimes = myList.map(times2)

// 定义一个可以进行递归计算的函数
def opN(x: Int, n: Int, op: Int => Int): Int = {
  if (n <= 0) { x }
  else { opN(op(x), n-1, op) }
}

opN(7, 3, plus1)
opN(7, 3, times2)
```

我们注意到，定义的`opN`函数，接收一个函数参数`op`。

### 函数VS对象

有时候，我们会看到，使用一些不带参数的函数，会造成一定的误会。

**`val`在定义的时候就已经求值，而`def`需要在调用的时候才会被求值。**

```scala
import scala.util.Random

// x和y都是Random函数，但是x在定义的时候，其已经被求值了，而y是一个函数，每次对他进行引用的时候，都会重新求值
val x = Random.nextInt
def y = Random.nextInt

// x已经被求值了，所以不会再发生改变了
println(s"x = $x")
println(s"x = $x")

// y的输出会和上次的不一样，因为调用的时候被重新进行了求值
println(s"y = $y")
println(s"y = $y")
```

### 匿名函数

我们仅仅使用一次这个函数，所以这个函数可以不用赋值给`val`，当作一个字面量（如C中的常量）来进行使用。

```scala
val myList = List(5, 6, 7, 8)

// 将列表中的每一个值都加1
myList.map( (x:Int) => x + 1 )
myList.map(_ + 1)

// 匿名函数中可以使用case，可以进行模式匹配
val myAnyList = List(1, 2, "3", 4L, myList)
myAnyList.map {
  case (_:Int|_:Long) => "Number"
  case _:String => "String"
  case _ => "error"
}
```

## Chisel中的函数式编程

### 实例：FIR

我们可以看一个在Chisel中使用函数式编程的例子，还是以刚才的FIR为例。

之前的所有`b_i`全部都是以固定参数的形式传入，这次，我们传入一个能够计算参数的函数。这个计算函数以窗口长度和位宽为参数，产生一个`b_i`的参数列表。

```scala
// get some math functions
import scala.math.{abs, round, cos, Pi, pow}

// simple triangular window
// 这个语法是先声明函数的类型，然后用'=’来用一个函数初始化val
val TriangularWindow: (Int, Int) => Seq[Int] = (length, bitwidth) => {
  val raw_coeffs = (0 until length).map( (x:Int) => 1-abs((x.toDouble-(length-1)/2.0)/((length-1)/2.0)) )
  val scaled_coeffs = raw_coeffs.map( (x: Double) => round(x * pow(2, bitwidth)).toInt)
  scaled_coeffs
}

// Hamming window
val HammingWindow: (Int, Int) => Seq[Int] = (length, bitwidth) => {
  val raw_coeffs = (0 until length).map( (x: Int) => 0.54 - 0.46*cos(2*Pi*x/(length-1)))
  val scaled_coeffs = raw_coeffs.map( (x: Double) => round(x * pow(2, bitwidth)).toInt)
  scaled_coeffs
}
```

然后就可以使用它来进行生成了：

```scala
// our FIR has parameterized window length, IO bitwidth, and windowing function
class MyFir(length: Int, bitwidth: Int, window: (Int, Int) => Seq[Int]) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(bitwidth.W))
    val out = Output(UInt((bitwidth*2+length-1).W)) // expect bit growth, conservative but lazy
  })

  // 将所有的参数转换成UInt硬件节点，宽度自动推断
  val coeffs = window(length, bitwidth).map(_.U)
  
  // 注意：我们不使用Vec，因为不需要按照索引访问，我们只需要在编译阶段把这些寄存器连接到正确的位置
  val delays = Seq.fill(length)(Wire(UInt(bitwidth.W))).scan(io.in)( (prev: UInt, next: UInt) => {
    next := RegNext(prev)
    next
  })
  
  // multiply, putting result in "mults"
  val mults = delays.zip(coeffs).map{ case(delay: UInt, coeff: UInt) => delay * coeff }
  
  // add up multiplier outputs with bit growth
  val result = mults.reduce(_+&_)

  // connect output
  io.out := result
}
```

这个实现和之前的简洁实现差不多，只是将连续的`map`,`reduce`操作拆分开了。

### 实例：感知机

![image-20200825221618486](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200825221618486.png)

```scala
class Neuron(inputs: Int, act: FixedPoint => FixedPoint) extends Module {
  val io = IO(new Bundle {
    val in      = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val weights = Input(Vec(inputs, FixedPoint(16.W, 8.BP)))
    val out     = Output(FixedPoint(16.W, 8.BP))
  })
  io.out := ( io.in zip io.weights ).map( case(a,b) => a*b ).reduce(_+_)
}
```

