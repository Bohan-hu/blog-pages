---
title: Chisel学习笔记（八）：Chisel标准库实例
date: 2020/08/25 13:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

Chisel提供了大量的标准接口，并且可以为我们提供可复用的硬件模块。

## Decoupled: 标准Ready-Valid接口

`DecoupledIO`是Chisel提供的一个标准接口，它提供了一个用于数据传输的Ready-Valid界面。

- 发送方（数据源）：控制`bits`和`valid`
- 接收方：控制`ready`，当其准备好接收数据的时候，拉高`ready`

这个接口提供了一个双向流控机制，也可以做到接收方反压发送方的效果。

注意：`ready`和`valid`信号不应该组合地依赖于对方，否则会导致组合逻辑环路。`ready`仅仅应当依赖于接收方是否准备好接收数据，`valid`应当仅仅依赖于发送方是否已经准备好数据。当传输完成后，`ready`和`valid`信号才允许变化。

Chisel提供了一个`DecoupledIO`接口。

```scala
val myChiselData = UInt(8.W)
val myDecoupled = Decoupled(myChiselData)
```

### 实例：队列

Chisel提供了队列，一个标准的Ready-Valid接口模型。

```scala
Driver(() => new Module {
    // Example circuit using a Queue
    val io = IO(new Bundle {
      val in = Flipped(Decoupled(UInt(8.W)))
      val out = Decoupled(UInt(8.W))
    })
    val queue = Queue(io.in, 2)  // 2-element queue
    io.out <> queue
  }) { c => new PeekPokeTester(c) {
    // Example testsequence showing the use and behavior of Queue
    poke(c.io.out.ready, 0)
    poke(c.io.in.valid, 1)  // Enqueue an element
    poke(c.io.in.bits, 42)
    println(s"Starting:")
    println(s"\tio.in: ready=${peek(c.io.in.ready)}")
    println(s"\tio.out: valid=${peek(c.io.out.valid)}, bits=${peek(c.io.out.bits)}")
    // in.Ready:1, out.Valid: 0
    step(1)
  	
    poke(c.io.in.valid, 1)  // Enqueue another element
    poke(c.io.in.bits, 43)
    // What do you think io.out.valid and io.out.bits will be?
    println(s"After first enqueue:")
    println(s"\tio.in: ready=${peek(c.io.in.ready)}")
    println(s"\tio.out: valid=${peek(c.io.out.valid)}, bits=${peek(c.io.out.bits)}")
    step(1)
    // in.Ready:1, out.Valid:1
    
    poke(c.io.in.valid, 1)  // Read a element, attempt to enqueue
    poke(c.io.in.bits, 44)
    poke(c.io.out.ready, 1)
    // What do you think io.in.ready will be, and will this enqueue succeed, and what will be read?
    println(s"On first read:")
    println(s"\tio.in: ready=${peek(c.io.in.ready)}")
    println(s"\tio.out: valid=${peek(c.io.out.valid)}, bits=${peek(c.io.out.bits)}")
    step(1)
    // in.Ready:0, out.Valid:1
  
    poke(c.io.in.valid, 0)  // Read elements out
    poke(c.io.out.ready, 1)
    // What do you think will be read here?
    println(s"On second read:")
    println(s"\tio.in: ready=${peek(c.io.in.ready)}")
    println(s"\tio.out: valid=${peek(c.io.out.valid)}, bits=${peek(c.io.out.bits)}")
    step(1)
    // in.Ready:1, out.Valid:1
  
    // Will a third read produce anything?
    println(s"On third read:")
    println(s"\tio.in: ready=${peek(c.io.in.ready)}")
    println(s"\tio.out: valid=${peek(c.io.out.valid)}, bits=${peek(c.io.out.bits)}")
    // in.Ready:1, out.Valid:0
    step(1)
} }
```

### 实例：仲裁器

根据预先设定好的优先级，仲裁器将数据从`DecoupledIO`源路由到`DecoupledIO`目的。

仲裁器分为

- 普通仲裁器：优先允许Index较低的请求
- 轮询仲裁器：轮询各个请求，优先级相等

**发起请求的时候，是拉高`Valid`信号，而请求被满足的时候，接收方会拉高`Ready`信号。**

```scala
Driver(() => new Module {
    // Example circuit using a priority arbiter
    val io = IO(new Bundle {
      val in = Flipped(Vec(2, Decoupled(UInt(8.W))))
      val out = Decoupled(UInt(8.W))
    })
    // Arbiter doesn't have a convenience constructor, so it's built like any Module
    val arbiter = Module(new Arbiter(UInt(8.W), 2))  // 2 to 1 Priority Arbiter
    arbiter.io.in <> io.in
    io.out <> arbiter.io.out
  }) { c => new PeekPokeTester(c) {
    poke(c.io.in(0).valid, 0)
    poke(c.io.in(1).valid, 0)
    println(s"Start:")
    println(s"\tin(0).ready=${peek(c.io.in(0).ready)}, in(1).ready=${peek(c.io.in(1).ready)}")
    println(s"\tout.valid=${peek(c.io.out.valid)}, out.bits=${peek(c.io.out.bits)}")
    poke(c.io.in(1).valid, 1)  // Valid input 1
    poke(c.io.in(1).bits, 42)
    // What do you think the output will be?
    println(s"valid input 1:")
    println(s"\tin(0).ready=${peek(c.io.in(0).ready)}, in(1).ready=${peek(c.io.in(1).ready)}")
    println(s"\tout.valid=${peek(c.io.out.valid)}, out.bits=${peek(c.io.out.bits)}")
    poke(c.io.in(0).valid, 1)  // Valid inputs 0 and 1
    poke(c.io.in(0).bits, 43)
    // What do you think the output will be? Which inputs will be ready?
    println(s"valid inputs 0 and 1:")
    println(s"\tin(0).ready=${peek(c.io.in(0).ready)}, in(1).ready=${peek(c.io.in(1).ready)}")
    println(s"\tout.valid=${peek(c.io.out.valid)}, out.bits=${peek(c.io.out.bits)}")
    poke(c.io.in(1).valid, 0)  // Valid input 0
    // What do you think the output will be?
    println(s"valid input 0:")
    println(s"\tin(0).ready=${peek(c.io.in(0).ready)}, in(1).ready=${peek(c.io.in(1).ready)}")
    println(s"\tout.valid=${peek(c.io.out.valid)}, out.bits=${peek(c.io.out.bits)}")
} }
```

## 其他常用函数

### 位操作

#### PopCount

PopCount对某个向量中的1的个数进行计数。

#### Reverse

反转输入的向量。

### 计数器

计数器每个周期加1.

```scala
val counter = Counter(3)  // 3-count Counter (outputs range [0...2])
when(io.count) {
	counter.inc()
}
```

其余的函数可以参见CheetSheet。