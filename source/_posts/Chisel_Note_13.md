---
title: Chisel学习笔记（十三）：类型系统
date: 2020/08/26 02:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

Scala是强类型语言。类似于Python这种动态类型语言，在解释执行的时候出现的错误，可能会在Scala的编译阶段出现，因此，在编译阶段报错，能够尽可能地减少运行时错误。

在这一节中，我们将讨论“类型是Scala中的一等公民”这个话题。

## 静态类型

Scala中的所有对象都有类型，常常是该对象所属的class，可以通过`getClass`方法取得。

```scala
println(10.getClass)
println(10.0.getClass)
println("ten".getClass)
```

强烈推荐在声明函数的时候定义输入和输出的类型，以避免奇怪的错误。特别是，对于重载过的运算符而言。

```scala
def double(s: String): String = s + s
// double("hi")      // 正确用法
// double(10)        // 编译报错
// double("hi") / 10 // 编译报错
```

返回值类型为`Unit`的函数不返回任何东西。

## Scala类型 VS. Chisel类型

以下的代码是合法的：

```scala
val a = Wire(UInt(4.W))
a := 0.U
```
因为`0.U` 是`UInt` ，是Chisel类型。

而以下代码：

```scala
val a = Wire(UInt(4.W))
a := 0
```
是非法的，因为0是 `Int` ，是Scala类型。

对于 `Bool`而言也是如此，在Chisel中是`Boolean`类型。
```scala
val bool = Wire(Bool())
val boolean: Boolean = false
// legal
when (bool) { ... }
if (boolean) { ... }
// illegal
if (bool) { ... }
when (boolean) { ... }
```

在编译的时候，Scala会做静态类型检查，所以能够找到类型不匹配的问题。

## Scala类型转换

### asInstanceOf

`x.asInstanceOf[T]` 将对象 `x` 转换成`T`类型。当无法转换时，将抛出异常。

```scala
val x: UInt = 3.U
try {
  println(x.asInstanceOf[Int])
} catch {
  case e: java.lang.ClassCastException => println("As expected, we can't cast UInt to Int")
}

// But we can cast UInt to Data since UInt inherits from Data.
println(x.asInstanceOf[Data])
```

在上面的运行结果中，UInt不能转换成Int。

### Chisel中的类型转换

### Type Casting in Chisel

下面代码的问题在于，强行地将一个`UInt` 赋值给了`SInt`，这在Chisel中是不允许的。

Chisel拥有一系列的类型转换函数，最常用的就是 `asTypeOf()`，其参数应该是某个Chisel信号。

```scala
class TypeConvertDemo extends Module {
    val io = IO(new Bundle {
        val in  = Input(UInt(4.W))
        val out = Output(SInt(4.W))
    })
    io.out := io.in//应当加上.asTypeOf(io.out) 
}
```

某些时候，我们还需要使用`asUInt()`和`asSInt()`。

## 类型匹配

### 匹配操作

在之前，我们讨论过类型匹配的问题。类型匹配包含于Scala的模式匹配机制之内，对于编写那些“通用类型”的代码，非常有用。

下面的例子中介绍了一种能够进行`UInt`和`SInt`的类型匹配机制。

```scala
class ConstantSum(in1: Data, in2: Data) extends Module {
    val io = IO(new Bundle {
        val out = Output(in1.cloneType)
    })
    (in1, in2) match {
        case (x: UInt, y: UInt) => io.out := x + y
        case (x: SInt, y: SInt) => io.out := x + y
        case _ => throw new Exception("I give up!")
    }
}
```

`Data`是Chisel中所有类型的超类，能够接收所有Chisel类型的参数。

注意：模式匹配仅仅能够使用在电路生成阶段，这是Scala的语法，而并不能对应到实际的电路中。

例如，下面的代码就有问题：

```scala
class InputIsZero extends Module {
    val io = IO(new Bundle {
        val in  = Input(UInt(16.W))
        val out = Output(Bool())
    })
    io.out := (io.in match {
        // note that case 0.U is an error
        case (0.U) => true.B
        case _   => false.B
    })
}
```

### Unapply方法

为什么我们可以直接做这样的类型匹配？

```scala
case class Something(a: String, b: Int)
val a = Something("A", 3)
a match {
    case Something("A", value) => value
    case Something(str, 3)     => 0
}
```

对于每个Case Class，Scala自动为其生成了`unapply`方法。这是一种语法糖，**能够让`match`语句在匹配类型的同时，将对象中的值提取出来进行匹配**。

让我们来看下面一个例子，例如，我们确定一个参数，如果`pipelineMe`为真，则取`delay`为 `3*totalWidth`；如果为假，则取为`2*someOtherWidth`。

```scala
case class SomeGeneratorParameters(
    someWidth: Int,
    someOtherWidth: Int = 10,
    pipelineMe: Boolean = false
) {
    require(someWidth >= 0)
    require(someOtherWidth >= 0)
    val totalWidth = someWidth + someOtherWidth
}

def delay(p: SomeGeneratorParameters): Int = p match {
    case sg @ SomeGeneratorParameters(_, _, true) => sg.totalWidth * 3
    case SomeGeneratorParameters(_, sw, false) => sw * 2
}
```

我们看到，这个`case`引用了实例中的字段。

下面的这种语法，能够让我们同时引用对象用于匹配的内部的值，同时引用其父对象`sg`。

```scala
case sg@SomeGeneratorParameters(_, sw, true) => sw
```

如果我们想对某个非case class采用`unapply`方法，可以手动在其伴生对象中写入：

```scala
class Boat(val name: String, val length: Int)
object Boat {
    def unapply(b: Boat): Option[(String, Int)] = Some((b.name, b.length))
    def apply(name: String, length: Int): Boat = new Boat(name, length)
}

def getSmallBoats(seq: Seq[Boat]): Seq[Boat] = seq.filter { b =>
    b match {
        case Boat(_, length) if length < 60 => true
        case Boat(_, _) => false
    }
}

val boats = Seq(Boat("Santa Maria", 62), Boat("Pinta", 56), Boat("Nina", 50))
println(getSmallBoats(boats).map(_.name).mkString(" and ") + " are small boats!")
```

## Chisel的类型层级

`chisel3.Data` 是所有Chisel硬件类型的基类。 `UInt`, `SInt`, `Vec`, `Bundle`等都是`Data`的子类。 `Data` 可以被用于IO，并且支持被`:=`赋值，例如wires, regs等。

在Chisel中，寄存器是多态类型的代表。 对于`RegEnable`寄存器的实现，如果想要其支持通用类型，最关键的就是那句`[T <: Data]`，使得`RegEnable`对于Chisel所有的硬件类型都能够工作。

```scala
def apply[T <: Data](next: T, enable: Bool): T = {
	val r = Reg(chiselTypeOf(next))
	when (enable) { r := next }
	r
}
```

某些操作符仅仅对于 `Bits`有定义，例如`+`。这就是你可以累加`UInt` 或`SInt`，但不能累加`Bundle`或`Vec`的原因。

### 例子：通用类型的移位寄存器

在Scala中，不仅仅是对象和函数可以当作参数，类型也可以当作参数。

我们通常需要提供类型相关的限制，在这个情况下，我们需要将信号放在一个`Bundle`里面，`:=`连接他们，并以其创建一个寄存器。

赋值语句不能随便进行，例如，`wire := 3`是不合法的，因为3是Scala的整数类型，并不是Chisel的`UInt`类型。如果我们创建了一个类型约束，并且声明`T`是Data的子类，那么我们就可以自由地使用`:=`了。

```scala
class ShiftRegisterIO[T <: Data](gen: T, n: Int) extends Bundle {
    require (n >= 0, "Shift register must have non-negative shift")
    
    val in = Input(gen.cloneType)
    val out = Output(Vec(n + 1, gen.cloneType)) // + 1 because in is included in out
    override def cloneType: this.type = (new ShiftRegisterIO(gen, n)).asInstanceOf[this.type]
}

class ShiftRegister[T <: Data](gen: T, n: Int) extends Module {
    val io = IO(new ShiftRegisterIO(gen, n))
    
    io.out.foldLeft(io.in) { case (in, out) =>
        out := in
        RegNext(in)
    }
}
```

注解：上面的`foldLeft`方法有点绕，需要一定时间理解。

（TBC）