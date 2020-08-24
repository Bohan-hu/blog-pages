---
title: Chisel学习笔记（六）：有限状态机
date: 2020/08/23 19:00:00
tag:
	- Chisel
catagory:
	- Chisel
	- 笔记
---

## Moore机编码风格

![image-20200823220700155](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823220700155.png)

考虑以上状态机。

我们使用Chisel中的`Enum`来表示状态，使用`switch`来描述状态转移。

```scala
import chisel3._
import chisel3.util._

class SimpleFsm extends Module {
	val io = IO(new Bundle {
		val badEvent = Input(Bool())
		val clear = Input(Bool())
		val ringBell = Output(Bool())
	})
	
	val green :: orange :: red :: Nil = Enum(3)		// Enum的参数是状态的数量
	val stateReg = RegInit(green)					// 使用状态去初始化Reg
	
	// 状态转移逻辑
	switch(stateReg) {
		is(green) {
			when(io.badEvent) {
				stateReg := orange
			}
		}
		is(orange) {
			when(io.badEvent) {
				stateReg := red
			}
		}
		is(red) {
			when(io.clear) {
				stateReg := green
			}
		}
	}
	
	// 输出逻辑
	io.ringBell := stateReg === red
}
```

> 需要注意到，和Verilog不同的是，在实现状态机时，我们并没有在代码中引入`next_state`这个信号，也就是说，不同于我们在Verilog中描述的三段式/二段式状态机。
>
> 其原因大抵是因为，Verilog不允许在同一个过程块中，对组合逻辑和时序逻辑进行同时的赋值更新，而Chisel是允许的。
>
> 在后面的Mealy机编码中，我们可以看到这样实现的好处。

## Mealy机编码风格

一个最简单的上升沿检测机制，可以使用一行Chisel代码实现：

```scala
val risingEdge = din & !RegNext(din)
```

其综合出的电路如图所示：

![image-20200823223351935](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823223351935.png)

对于一个上升沿检测，如果我们使用状态机进行表示，其状态转移图如图所示：

![image-20200823223610841](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823223610841.png)

由于是Mealy状态机，所以状态机的输出也和输入有关。

```scala
import chisel3._
import chisel3.util._

class RisingFsm extends Module {
	val io = IO(new Bundle{
		val din = Input(Bool())
		val risingEdge = Output(Bool())
	})
	
	val zero :: one :: Nil = Enum(2)
	val stateReg = RegInit(zero)
	
	io.risingEdge := false.B	// 组合逻辑输出必须有默认值
	
	switch (stateReg) {
		is(zero) {
			when(io.din) {	
				stateReg := one
				io.risingEdge := true.B
			}
		}
		is(one) {
			when(!io.din) {
				stateReg := zero
				io.risingEdge := false.B
			}
		}
	}
}
```

> 在上述代码中，我们可以看到，和Verilog编码风格完全不同，时序逻辑和组合逻辑在同一个Block（块）中进行了赋值，组合逻辑将立即发生跳变，而时序逻辑将在下一个时钟上升沿更新，刚刚从Verilog转过来的设计者可能不太适应这种风格。
>
> 或许我们应该转变思路，应该将 `stateReg := one`这样的代码看作是**次态转移逻辑**，而不是**对现态寄存器的更新**，此时：
>
> 猜测这种写法的设计哲学是，将**次态转移和输出逻辑**写在了同一个块中，方便维护。

