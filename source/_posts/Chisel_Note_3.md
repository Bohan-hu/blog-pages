---
title: Chisel学习笔记（三）：Chisel的构建与测试
date: 2020/08/22 15:00:00
tag:
	- Chisel
catagory:
	- Chisel学习
---

本节主要记录如何编译Chisel代码，生成Verilog和测试我们的Chisel设计。由于Chisel使用Scala实现，构建Scala的常用工具是`sbt`。在编译和测试的过程中，`sbt`下载对应版本的Scala和Chisel库。

## 使用sbt构建项目

- 在构建项目时，会通过`build.sbt`中指出的库，从Maven仓库中下载相应版本的Chisel。
- 如果在`build.sbt`中使用`latest.release`，则每次都会下载Chisel的最新版本
- 推荐在`build.sbt`中指明Chisel版本，以达到离线使用的目的

### 源文件组织

![image-20200823131332598](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823131332598.png)

推荐的文件组织方式如上图所示。

Package将Chisel代码划分成命名空间，Package还可以包括不同的子Pacakge。

Target文件夹包括了那些类的文件和其他生成好的文件。

generated文件夹包含了生成的Verilog代码。

以下是一个Package的实例：

```scala
pacakge mypack
import chisel3._

class Abc extens Module {
	val io = IO(new Bundle{})
}
```

注意到我们导入了Chisel包。

如果要在其他的环境（不同的命名空间）中使用Abc模块，需要导入mypack包，下划线代表通配符，表示导入`mypack`包中所有的类。

```
import mypack._
```

也可以导入一部分类，此时需要指明包名和类名。

```scala
class AbcUser2 extends Module {
	val io = IO(new Bundle{})
	val abc = new mypack.Abc()
}
```

```scala
import mypack.Abc
class AbcUser3 exteds Module {
	val io = IO(new Bundle{})
	val abc = new Abc()
}
```

### 运行sbt

Chisel项目可以用`sbt`命令编译并运行：

```
sbt run
```

这个命令将搜索Chisel源码树中的满足：拥有包含`main`方法的`object`的类 / 继承了App类的类。如果有多个满足条件，将会列出所有供选择。

当然也可以进行显式的选择。

```
sbt "runMain mypacket.MyObject"
```

Chisel测试程序包含`main`方法，但被放在了`test`文件夹中，并不会被`sbt`扫描到（这是Java/Scala的一个约定，认为test文件夹中仅包含单元测试，并没有main方法）。为了在`tester`文件夹中执行，需要使用如下的命令：

```
sbt "test:runMain mypacket.MyTester"
```

### 工具流

![image-20200823131357855](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200823131357855.png)

电路的RTL文件是`Hello.scala`，Scala编译器将这个文件和Chisel库、Scala库一起编译，生成Java类`Hello.class`，使用JVM进行执行，执行后生成RTL的中间表示（FIRRTL）。

Treadle是一个FIRRTL的解释器，可以用来仿真电路，可以生成波形文件。

Verilog Emitter可以生成Verilog代码。

## 使用Chisel进行测试

DUT: Design Under Test， 被测模块

### PeekPokeTester

Chisel可以使用Scala的很多特性来编写TB，例如，可以**使用软件来编写硬件的功能，并且在仿真时和真正的硬件行为进行比较**。例如，在[Martin Schoeberl. Lipsi: Probably the smallest processor in the world. In Architecture of Computing Systems – ARCS 2018, pages 18–30. Springer International Publishing, 2018.]中，实现一个处理器并进行测试。

需要导入的包有：

```scala
import chisel3._
import chisel3.iotesters._
```

对电路进行测试，需要包含三个元素：

- 用于测试的单元
- 测试逻辑
- 测试对象，包含开始测试的`main`方法

#### DUT（被测对象）

```scala
class DeviceUnderTest extends Module {
	val io = IO(new Bundle {
		val a = Input(UInt(2.W))
		val b = Input(UInt(2.W))
		val out = Output(UInt(2.W))
	} )
	
	io.out := io.a & io.b
}
```

#### 测试逻辑

测试对象需要从`PeekPokeTester`中继承，并且以DUT作为参数传入。

```scala
class TesterSimple(dut: DeviceUnderTest ) extends
    PeekPokeTester(dut) {
    poke(dut.io.a, 0.U)
    poke(dut.io.b, 1.U)
    step (1)
    println("Result is: " + peek(dut.io.out).toString)
    poke(dut.io.a, 3.U)
    poke(dut.io.b, 2.U)
    step (1)
    println("Result is: " + peek(dut.io.out).toString)
}
```

`PeekPokeTester`可以用`poke()`设置输入，并使用`peek()`获取输出。使用`step(1)`来前进一个时钟周期。

#### 测试对象（object）

```scala
object TesterSimple extends App {
    chisel3. iotesters .Driver (() => new DeviceUnderTest ()) { c =>
    	new TesterSimple (c)
    }
}
```

也可以使用`expect`断言，让程序自动检查测试是否通过。

```scala
class Tester(dut: DeviceUnderTest ) extends PeekPokeTester(dut) {
    poke(dut.io.a, 3.U)
    poke(dut.io.b, 1.U)
    step (1)
    expect(dut.io.out , 1)
    poke(dut.io.a, 2.U)
    poke(dut.io.b, 0.U)
    step (1)
    expect(dut.io.out , 0)
}
```

### ScalaTest

ScalaTest是一个用于测试Scala的工具，我们也可以用来运行Chisel测试。

```
libraryDependencies += "org.scalatest" %% "scalatest" % "3.0.5" % "test"
```

使用

```
sbt test
```

就可以运行在`src/test/scala`下面的测试程序。

一个简单的测试程序如下：

```scala
import org. scalatest ._
class ExampleSpec extends FlatSpec with Matchers {
    "Integers" should "add" in {
        val i = 2
        val j = 3
        i + j should be (5)
    }
}
```

事实上，我们可以将Chisel的测试包裹到Scala中去。

```scala
class SimpleSpec extends FlatSpec with Matchers {
	"Tester" should "pass" in {
        chisel3. iotesters .Driver (() => new DeviceUnderTest ()) { c =>
            new Tester(c)
        } should be (true)
    }
}
```

这样做的好处是，可以使用

```
sbt "testOnly SimpleSpec"
```

来运行单个测试。

### 波形图

要生成波形图，应该在进行测试时添加一定的参数。

```scala
class WaveformSpec extends FlatSpec with Matchers {
    "Waveform" should "pass" in {
        Driver.execute(Array("--generate -vcd-output", "on"), () =>
            new DeviceUnderTest ()) { c =>
            new WaveformTester (c)
        } should be (true)
    }
}
```

生成的`vcd`波形图文件可以使用GTKWake或者Modelsim打开。

### printf调试

在Chisel中写printf语句，在时钟的上升沿会触发printf语句。

```scala
class DeviceUnderTestPrintf extends Module {
    val io = IO(new Bundle {
        val a = Input(UInt (2.W))
        val b = Input(UInt (2.W))
        val out = Output(UInt (2.W))
    })
        io.out := io.a & io.b
        printf("dut: %d %d %d\n", io.a, io.b, io.out)
}
```

