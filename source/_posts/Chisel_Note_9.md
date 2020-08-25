---
title: Chisel学习笔记（九）：容器
date: 2020/08/25 15:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

在这个模块中，我们将使用Chisel容器作为硬件的生成器。

## 背景：FIR滤波器

我们首先来了解一下FIR滤波器：

![image-20200825135235362](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200825135235362.png)

对于FIR滤波器的输出，我们这样定义：

$$ y[n] = b_0x[n]+b_1x[n-1]+b_2x[n-2]+\cdots$$

其中，

- $y[n]$是在第$n$时刻的输出信号
- $x[n]$是输入信号
- $b_i$是滤波器的参数，或脉冲反馈
- $x[n-1]$是上个周期的$x[n]$，以此类推。

### 简单实现

对于上图出现的滤波器，我们可以简单的实现：

```scala
class FIR_4(b0: Int, b1: Int, b2: Int, b3: Int) extends Module {
	val io = IO(new Bundle{
		val in = Input(UInt(8.W))
		val out = Output(UInt(8.W))
	})
	val x_n1 = RegNext(io.in, 0.U)
	val x_n2 = RegNext(x_n1, 0.U)
	val x_n3 = RegNext(x_n2, 0.U)
	io.out := io.in * b0.U(8.W) + 
    	x_n1 * b1.U(8.W) +
	    x_n2 * b2.U(8.W) + 
	    x_n3 * b3.U(8.W)
}
```

## 参数化实现与验证

如果我们想要引入更多的状态，例如，在上面的例子中，我们如果需要做到让FIR的Taps数量可以参数化，应该怎么实现呢？

- 首先，使用软件建立一个可以用于仿真的模型
- 其次，重新设计这个硬件，通过刚刚的仿真确定是否正常工作

### 软件实现的Golden Model

```scala
/**
  * A naive implementation of an FIR filter with an arbitrary number of taps.
  */
class ScalaFirFilter(taps: Seq[Int]) {
  var pseudoRegisters = List.fill(taps.length)(0)

  def poke(value: Int): Int = {
    pseudoRegisters = value :: pseudoRegisters.take(taps.length - 1)
    var accumulator = 0
    for(i <- taps.indices) {
      accumulator += taps(i) * pseudoRegisters(i)
    }
    accumulator
  }
}
```

我们使用了一个Var来记录上图中的X寄存器阵列。

对于Poke方法，我们也对其进行了相应的重写，以模拟硬件的行为。

此处需要注意的是，`value :: pseudoRegisters.take(taps.length - 1)`表示将某个值追加到列表的头部。

### 使用容器实现

此处发生了一系列的改动：

- 输入的常量从`b0,b1,b2`变成了一个整数序列
- 宽度也可以定制了：`bitWidth`

```scala
class MyManyElementFir(consts: Seq[Int], bitWidth: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(bitWidth.W))
    val out = Output(UInt(bitWidth.W))
  })
  // 注意到这是Mutable的，也就是可变的
  val regs = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      // 寄存器阵列，除了第一个是直接采样输入，其他的都是前几个时刻的输入
      if(i == 0) regs += io.in
      else       regs += RegNext(regs(i - 1), 0.U)
  }
  
  // 做完乘法的结果
  val muls = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      muls += regs(i) * consts(i).U
  }

  // 
  val scan = mutable.ArrayBuffer[UInt]()
  for(i <- 0 until consts.length) {
      if(i == 0) scan += muls(i)
      else scan += muls(i) + scan(i - 1)
  }

  io.out := scan.last
}
```

#### 代码注解

代码中存在三个并行块，分别使用了Scala的容器 `ArrayBuffer`。 `ArrayBuffer` 可以使用 `+=` 往后追加元素。

在第一个块中，我们创建了一个`regs` 的`ArrayBuffer`，其中的元素是`UInt`，注意，这个集合仅仅包含硬件的输出（寄存器、组合逻辑的输出），而不包含实际的寄存器。创建的寄存器`RegNext`，是匿名的硬件节点。

>  这里跟`Vec`是不一样的，我们是否可以通过其访问某个硬件呢？

## 实例：RISC寄存器堆

不多说，再熟悉不过了吧。

```scala
class RegisterFile(readPorts: Int) extends Module {
    require(readPorts >= 0)
    val io = IO(new Bundle {
        val wen   = Input(Bool())
        val waddr = Input(UInt(5.W))
        val wdata = Input(UInt(32.W))
        val raddr = Input(Vec(readPorts, UInt(5.W)))
        val rdata = Output(Vec(readPorts, UInt(32.W)))
    })
    
    // A Register of a vector of UInts
    val reg = RegInit(VecInit(Seq.fill(32)(0.U(32.W))))
    when (io.wen) {
        reg(io.waddr) := io.wdata
    }
    for (i <- 0 until readPorts) {
        when (io.raddr(i) === 0.U) {
            io.rdata(i) := 0.U
        } .otherwise {
            io.rdata(i) := reg(io.raddr(i))
        }
    }
}
```



## 使用生成器的方法实现

另一种实现方法，我们将在后面讨论。

```scala
class MyManyDynamicElementVecFir(length: Int) extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(8.W))
    val valid = Input(Bool())
    val out = Output(UInt(8.W))
    val consts = Input(Vec(length, UInt(8.W)))
  })
  
  // Such concision! You'll learn what all this means later.
  val taps = Seq(io.in) ++ Seq.fill(io.consts.length - 1)(RegInit(0.U(8.W)))
  taps.zip(taps.tail).foreach { case (a, b) => when (io.valid) { b := a } }

  io.out := taps.zip(io.consts).map { case (a, b) => a * b }.reduce(_ + _)
}
```

