---
title: Chisel学习笔记（十）：高阶函数
date: 2020/08/25 17:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

我们在前面的代码中，经常使用到`for`循环，显然过于冗长，并且和函数式编程的宗旨相悖。

在本节中，我们以之前实现的FIR滤波器为例，通过Scala的特性进行重构。

## 回顾：FIR滤波器

![image-20200825135235362](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200825135235362.png)

$$ y[n] = b_0x[n]+b_1x[n-1]+b_2x[n-2]+\cdots$$

### 之前的实现

```scala
class MyManyDynamicElementVecFir(length: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val out = Output(UInt(8.W))
    val consts = Input(Vec(length, UInt(8.W)))
  })

  // Reference solution
  val regs = RegInit(VecInit(Seq.fill(length - 1)(0.U(8.W))))
  for(i <- 0 until length - 1) {
      if(i == 0) regs(i) := io.in
      else       regs(i) := regs(i - 1)
  }
  
  val muls = Wire(Vec(length, UInt(8.W)))
  for(i <- 0 until length) {
      if(i == 0) muls(i) := io.in * io.consts(i)
      else       muls(i) := regs(i - 1) * io.consts(i)
  }

  val scan = Wire(Vec(length, UInt(8.W)))
  for(i <- 0 until length) {
      if(i == 0) scan(i) := muls(i)
      else scan(i) := muls(i) + scan(i - 1)
  }

  io.out := scan(length - 1)
}
```

回顾我们的实现：

- 对于每一个`regs(i)`，和其对应的`io.const`相乘，并且存储到`muls`向量中。
- 对于每一个`muls(i)`，`scan(0) = muls(0)`, `scan(1) = scan(0) + muls(1) = muls(0) + muls(1)`......
- 取出`scan`中的最后一个元素，并且赋值给`io.out`

事实上，我们可以使用一个更简单的方法实现。

### 更简单的实现

前方高能。

----

```scala
class MyManyDynamicElementVecFir(length: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val out = Output(UInt(8.W))
    val consts = Input(Vec(length, UInt(8.W)))
  })
  io.out := (taps zip io.consts).map { case (a, b) => a * b }.reduce(_ + _)
}
```

>  就这？就这？？？？？？你在逗我？？？？？？？？？？

让我们来解析一下这个代码。

- `(taps zip io.consts)` 的输入是两个List： `taps` 和 `io.consts`，这个函数最终返回的是一个列表，其中的元素是二元组，两个数组相同下标的元素被组合成一个二元组。最终，列表会长这样：`[(taps(0), io.consts(0)), (taps(1), io.consts(1)), ..., (taps(n), io.consts(n))]`。注意：在Scala中，由于对于仅有一个参数的方法，其调用可以省略`.`和`()`，所以这个等效于`(taps.zip(io.consts))`。

- `.map { case (a, b) => a * b }` 在列表中的每一个函数上面都应用了一个匿名函数，并返回如此操作之后的列表。这个匿名函数的输入是一个元组`(a,b)`，函数输出是`a*b`。最终，返回的列表是 `[taps(0) * io.consts(0), taps(1) * io.consts(1), ..., taps(n) * io.consts(n)]`。

- 最终, `.reduce(_ + _)` 也对列表进行了操作。这个函数拥有两个参数：

  -  当前的累加和

  - 当前处理到的元素

    最终返回的结果应当是 `(((muls(0) + muls(1)) + muls(2)) + ...) + muls(n)`，最深层的括号是最先被计算的。这就是`reduce`模型。

## 函数作为参数

在我们上面所见的`map`和`reduce`被称为**高阶函数**。为什么称之为高阶函数？因为他们输入的参数是**函数**。

函数式编程的一个好处是，我们不必聚焦于控制相关的代码，而是可以专注于编写逻辑。

### 声明匿名函数的不同方法

- 对于那些**每个参数都仅仅被使用过一次**的函数，可以使用下划线(`_`)来引用每一个参数。在上面的例子中，`reduce`的就是拥有两个参数，可以被写作是`_+_`，代表第一个参数加第二个参数
- 也可以显式地声明那些输入参数，例如，上面的函数可以写成是`(a,b) => a + b`。将参数列表放在括号中，后接一个`=>`，然后是函数体。
- 当需要对元组进行解包的时候，需要使用`case`语句。`case (a,b) => a * b`

### 在Scala中的实践

#### Map函数

`List[A].map`的定义是`map[B](f: (A)=>B)):List[B]`。定义看起来略微有点复杂，我们先将`A`认为是`Int`（软件类型），`B`认为是`UInt`（硬件类型）。

上面的声明可以看作是：`map`函数接收一个输入类型为`A`，返回类型为`B`的函数，并且返回一个元素类型为`B`的列表。

#### zipWithIndex函数

`List.zipWithIndex`的定义是`zipWithIndex: List[(A, Int)]`。

`List("a", "b", "c", "d").zipWithIndex`将返回`List(("a", 0), ("b", 1), ("c", 2), ("d", 3))`。

#### Reduce函数

`List[A].reduce`的定义是`reduce(op: (A, A) ⇒ A): A`。

事实上，`A`只需要是子类就可以了。

```scala
println(List(1, 2, 3, 4).reduce((a, b) => a + b))  // returns the sum of all the elements
println(List(1, 2, 3, 4).reduce(_ * _))  // returns the product of all the elements
println(List(1, 2, 3, 4).map(_ + 1).reduce(_ + _))  // you can chain reduce onto the result of a map
```

#### Fold函数

`fold`函数和`reduce`函数非常类似，有一点不同的是，`fold`函数对于迭代具有初始值，可以从`fold`函数的定义中看出： `fold(z: A)(op: (A, A) ⇒ A): A`

```scala
println(List(1, 2, 3, 4).fold(0)(_ + _))  // equivalent to the sum using reduce
println(List(1, 2, 3, 4).fold(1)(_ + _))  // like above, but accumulation starts at 1
println(List().fold(1)(_ + _))  // unlike reduce, does not fail on an empty input
```

## 高阶函数应用实例：仲裁器

我们将构建一个仲裁器，拥有`n`个输入和1个输出，选择编号最小的有效输出。

这个例子需要一定的时间消化。

```scala
class MyRoutingArbiter(numChannels: Int) extends Module {
  val io = IO(new Bundle {
    val in = Vec(Flipped(Decoupled(UInt(8.W))), numChannels)
    val out = Decoupled(UInt(8.W))
  } )

  io.out.valid := io.in.map(_.valid).reduce(_ || _)		// 取出Valid位，并将所有的Valid或起来
  val channel = PriorityMux(
    io.in.map(_.valid).zipWithIndex.map { case (valid, index) => (valid, index.U) }
  )
  io.out.bits := io.in(channel).bits					// 将编码好的进行选择
  io.in.map(_.ready).zipWithIndex.foreach { case (ready, index) =>
    ready := io.out.ready && channel === index.U		// 被选中的就Ready
  }
}
```

`PriorityMux(List[Bool, Bits])`，按照Index从低到高，选中第一个有效的，其实是一个优先编码+多路选择。

---

```scala
io.out.valid := io.in.map(_.valid).reduce(_ || _)
```

`io.in.map(_.valid)`将输入中所有的Valid取出，组成一个新的向量。

`.reduce(_ || _)`将向量中所有的Bit都或在一起。

---

```scala
io.in.map(_.valid).zipWithIndex.map { case (valid, index) => (valid, index.U) }
```

`io.in.map(_.valid).zipWithIndex`将每一项都和他的Index串联在一起。

`.map { case (valid, index) => (valid, index.U) }`将`index`转换为硬件信号，因为Mux的输出是硬件信号，同时`Vec`也需要硬件信号做索引。

---

```scala
io.in.map(_.ready).zipWithIndex.foreach { case (ready, index) =>
	ready := io.out.ready && channel === index.U		// 被选中的就Ready
}
```

>  注意：此处虽然定义了新的函数，`case(ready, index) => `，但是传入的仍然是原来的硬件节点，也就是说，传入函数的硬件节点不会被重复的创建，相当于传递的是引用。

