---
title: Chisel学习笔记（五）：组合逻辑与时序逻辑描述
date: 2020/08/23 17:00:00
tag:
	- Chisel
catagory:
	- Chisel
	- 笔记
---

# Part A: 组合逻辑

## 组合逻辑描述

在Chisel中，组合逻辑电路最简单的表示方法是将组合逻辑赋值给一个`val`，并可以通过这个`val`引用。

```scala
val e = ( a & b ) | c
```

以上表达式右侧的逻辑被**命名**为`e`，对于`val`的重新赋值是不允许的。

Chisel当然也支持使用条件语句生成的组合逻辑。这样的电路以`Wire`定义（在Verilog中，我们以Reg定义，在`always@(*)`块中对其赋值）。此后，我们使用条件操作，例如`when`，来描述这个电路的行为。

```scala
val w = Wire(UInt())
w := 0.U
when(cond) {
	w := 3.U
}
```

`when`语句还可以后接`otherwise`语句。

```
when (cond) {
	w := 1.U
} otherwise {
	w := 2.U
}
```

还可以插入`elsewhen`，注意代码中的"."是必要的。

```
when (cond) {
	w := 1.U
} .elsewhen(cond2) {
	w := 2.U
} otherwise {
	w := 3.U
}
```

**如果`cond`仅仅依赖于一个信号，建议还是使用`switch`语句。**

---

对于复杂的逻辑，我们应该对Wire赋予初值，以避免锁存器。

```scala
val w = WireDefault(0.U)
when (cond) {
    w := 3.U
}
```

需要注意的是，Scala中的`if`,`else`是控制流代码，是用于控制生成硬件的条件的，并不能用来描述组合逻辑，也并不会生成硬件。

## Example: Decoder

Decoder一般可以使用`switch`语句来进行描述。

```scala
import chisel3.util._

result := 0.U

switch(sel)	{
	is(0.U) { result := 1.U }
	is(1.U) { result := 2.U }
	is(2.U) { result := 4.U }
	is(3.U) { result := 8.U }
}
```

**注意：就算在`switch`里面列举了所有的可能，我们仍然需要给某个信号赋予初值，为了防止生成锁存器的可能，Chisel严禁使用不完备的赋值。**

# Part B： 时序逻辑

## 时序逻辑描述：寄存器

![image-20200823174343426](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823174343426.png)

在Chisel中，上图寄存器可以被这样描述：

```scala
val q = RegNext(d)
```

我们可以注意到，这个寄存器并没有显式地声明时钟、声明复位，端口被默认的连接到了全局时钟上，注意：Chisel不支持异步复位。

RegNext会自动根据输入信号推断寄存器的宽度。

还可以使用以下的形式，声明一个寄存器，并赋值：

```scala
val q = Reg(UInt(4.W))
q := data_in
```

- 由于在Chisel中，并不存在阻塞赋值和非阻塞赋值，也不存在`always`块，所以难以某个赋值语句是在对寄存器赋值还是组合逻辑赋值。
  - **推荐的命名规范是，寄存器信号以`Reg`结尾！**
  - 并且，在Scala推荐的命名规范中，建议采用**驼峰命名法**。

- 在声明寄存器时，还可以对寄存器指定初始值。

```scala
val valReg = RegInit(0.U(4.W))
valReg := inVal
```

- 声明一个带使能的寄存器：

```scala
val enableReg = RegInit(0.U(4.W))
when(enable) {
	enableReg := inVal
}
```

- 寄存器甚至可以在表达式中声明，甚至作为表达式的一个部分进行使用，但此时，这是一个匿名的寄存器：

```scala
val risingEdge = din & !RegNext(din)
```

>  此处有一个思考，之前使用`val`来定义某个逻辑，不管是寄存器也好，还是组合逻辑也好，本质上是对某个硬件节点进行命名操作。此处在表达式中使用RegNext，相当于创建了一个新的寄存器，并匿名地引用它的值，其用法和组合逻辑有点类似。

## Example: Counter

一个最简单的计数器：

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := cntReg + 1.U
when(cntReg === N) {
	cntReg := 0.U
}
```

或者，我们可以使用一个多选器来决定计数器的输入：

```scala
val cntReg = RegInit(0.U(8.W))
cntReg := Mux(cntReg === N, 0.U, cntReg + 1.U)
```

再高层次地，我们使用一个计数器生成器：

```scala
def genCounter(n: Int) = {
	val cntReg = RegInit(0.U(8.W))
	cntReg := Mux(cntReg === n.U, 0.U, cntReg + 1.U)
	cntReg
}
val cnt99 = genCounter(99)
```

注意函数的最后一行，代表返回一个创建好的硬件节点。

### Counter使用实例

![image-20200823183445450](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823183445450.png)

```scala
// tick是一个三分频时钟，其占空比不为0.5
val tickCounterReg = RegInit(0.U(4.W))
val tick = tickCounterReg === (N-1).U

tickCounterReg := Mux(tick, 0, tickCounterReg + 1.U)
val lowFreqCntReg = RegInit(0.U(4.W))
when(tick) {
    lowFreqCntReg := lowFreqCntReg + 1.U
}
```

其中，`tick`是一个占空比不为50%的时钟，控制一个更慢的计数器进行更新操作。

### 更优化的计数器

我们转变思路，将计数器转变为递减计数器。从N-1数到0，转变为N-2数到-1，计满的情况，仅需要看计数器最高位即可。

## Example: PWM

![image-20200823191133656](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823191133656.png)

PWM调制，时钟占空比随着时间的变化而变化。

以下的代码可以生成占空比为0.7的PWM波形：

```scala
def pwm(nrCycles: Int, din: UInt) = {
	val cntReg = RegInit(0.U(unsignedBitLength(nrCycles-1).W))		// 自动计算宽度
	cntReg := Mux(cntReg === (nrCycles-1).U, 0.U, cntReg + 1.U)		// 普通的循环计数器
	din > cntReg	// 当din > cntReg的时候为高，din即为输出从高到低的交界点
}

val pwm_out = (10, 3.U)
```

上述代码即为一个可复用的、轻量的代码。它接收两个参数，一个是时钟周期的数量（`nrCycles`），另一个是给出了脉冲宽度（`din`）。

**注意：函数的最后一行是返回值。**

```scala
val FREQ = 100000000 // a 100 MHz clock input
val MAX = FREQ /1000 // 1 kHz
val modulationReg = RegInit (0.U(32.W))
val upReg = RegInit(true.B)
when ( modulationReg < FREQ.U && upReg) {
	modulationReg := modulationReg + 1.U
} .elsewhen ( modulationReg === FREQ.U && upReg) {
	upReg := false.B								// 计数加模式
} .elsewhen ( modulationReg > 0.U && !upReg) {
	modulationReg := modulationReg - 1.U			// 计数减模式
} . otherwise { // 0
	upReg := true.B
}
// divide modReg by 1024 (about the 1 kHz)
val sig = pwm(MAX , modulationReg >> 10)
```

### Example: 移位寄存器

```scala
val shiftReg = Reg(UInt(4.W))
shiftReg := Cat(shiftReg(2,0), din)
val dout = shiftReg(3)
```

`Cat`用于串接输入输出，对于Reg和Wire的位选，使用括号，也就是Scala中的`Apply()`方法。

## 存储器

存储器可以用寄存器阵列进行构建，也可以使用SRAM。在ASIC设计中，使用`memory compiler`进行设计；在FPGA设计中，片上拥有block RAM资源。

Chisel提供了存储器构建器`SyncReadMem`，对于同时读写一个地址，Chisel将其定义为未定义的行为，此时，我们可以手动进行旁路。

```scala
class MemoryWithForwarding() extends Module {
	val io = IO(new Bundle{
		val rdAddr = Input(UInut(10.W))
		val rdData = Input(UInt(8.W))
		val wrEna = Input(Bool())
		val wrData = Input(UInt(8.W))
		val wrAddr = Input(UInt(10.W))
	})
	val mem = SyncReadMem(1024,UInt(8.W))
	
	val wrDataReg = RegNext(io.wrData)
	val doForwardingReg = RegNext(io.wAddr === io.rdAddr && io.wrEna)
	
	val memData = mem.read(io.rdAddr)
	
	when(io.wrEna) {
		mem.write(io.wrAddr, io.wrData)
	}
	
	io.rdData := Mux(doForwardingReg, wrDataReg, memData)
}
```

Chisel也提供了`Mem`，异步读，同步写，类似于FPGA中的`distributed RAM`。