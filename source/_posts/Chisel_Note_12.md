---
title: Chisel学习笔记（十二）：面向对象
date: 2020/08/25 22:00:00
tag:
	- Chisel
catagory:
	- Chisel笔记
---

Chisel基于Scala，而Scala支持面向对象的编程范式，也就是说，代码可以被分割成不同的对象。

## Scala中的面向对象

在这一节中，我们将展示Scala是如何实现面向对象的编程范式的。在面向对象的编程中，Scala还拥有以下的特性：

- 抽象类（Abstract class）
- 特质（Traits）
- 对象（Objects）
- 伴生对象（Companion Objects）
- 案例类（Case Classes）

### 抽象类

对于抽象类，熟悉面向对象的人应该不陌生了。在抽象类中，我们定义一些继承他的子类必须实现的字段。

任何对象只能从最多一个抽象类中继承。

以下是抽象类的例子：

```scala
abstract class MyAbstractClass {
  def myFunction(i: Int): Int
  val myValue: String
}
class ConcreteClass extends MyAbstractClass {
  def myFunction(i: Int): Int = i + 1
  val myValue = "Hello World!"
}
// val abstractClass = new MyAbstractClass() // 这段代码不能编译，因为抽象类的部分字段没有实现
val concreteClass = new ConcreteClass()      // 这段代码是合法代码
```

### 特质

特质和抽象类非常相似，在于他们都可以定义“未实现”的字段。特质和抽象类有两点不同：

- 一个类可以从多个特质继承而来
- 一个特质不能拥有构造函数的参数

```scala
trait HasFunction {
  def myFunction(i: Int): Int
}
trait HasValue {
  val myValue: String
  val myOtherValue = 100
}
class MyClass extends HasFunction with HasValue {
  override def myFunction(i: Int): Int = i + 1
  val myValue = "Hello World!"
}
// Uncomment below to test!
// val myTraitFunction = new HasFunction() // Illegal! Cannot instantiate a trait
// val myTraitValue = new HasValue()       // Illegal! Cannot instantiate a trait
val myClass = new MyClass()                // Legal!
```

通常情况下，请多多使用特质，而非抽象类，除非我们能够确保仅仅从抽象类做单继承。

### 对象

对于单例的类（仅有一个对象的类），可以创建一个单独的Object。

Object不能被实例化，我们可以直接引用这个对象。

```scala
object MyObject {
  def hi: String = "Hello World!"
  def apply(msg: String) = msg
}
println(MyObject.hi)
println(MyObject("This message is important!")) // equivalent to MyObject.apply(msg)
```

### 伴生对象

当一个类和一个对象的**名字相同，且位于同一个文件中**时，这个对象可以称为是**伴生对象**。

当在名字前面看到`new`关键字的时候，它将创建一个类的实例；当没有使用`new`的时候，它将指向伴生对象。

```scala
object Lion {
    def roar(): Unit = println("I'M AN OBJECT!")
}
class Lion {
    def roar(): Unit = println("I'M A CLASS!")
}
new Lion().roar()
Lion.roar()
```

我们为什么要使用伴生对象呢？原因有几个：

- 维护那些和类有关的常量
- 在运行类的构造函数之前/之后，运行一些其他的代码
- 创建一个类的很多个构造函数

---

看下面代码的例子：

```scala
object Animal {
    val defaultName = "Bigfoot"
    private var numberOfAnimals = 0
    def apply(name: String): Animal = {
        numberOfAnimals += 1
        new Animal(name, numberOfAnimals)
    }
    def apply(): Animal = apply(defaultName)
}
class Animal(name: String, order: Int) {
  def info: String = s"Hi my name is $name, and I'm $order in line!"
}

val bunny = Animal.apply("Hopper") // Calls the Animal factory method
println(bunny.info)
val cat = Animal("Whiskers")       // Calls the Animal factory method
println(cat.info)
val yeti = Animal()                // Calls the Animal factory method
println(yeti.info)
```

- `Animal`伴生对象定义了一个和`Animal`类有关的常量：

  ```scala
  val defaultName = "Bigfoot"
  ```

- 并且给出了一个可变的字段，来记录创建的实例数量：

  ```scala
  private var numberOfAnimals = 0
  ```

- 然后定义了两个`apply`方法，我们称之为**工厂方法**，能够返回`Animal`类的实例

  - 第一个方法，传入动物名称，并将总计数器+1，返回新的实例
  - 第二个方法，以默认名称创建实例

- 工厂方法可以以以下的形式调用：

  ```scala
  val bunny = Animal.apply("Hopper")
  ```

  也可以省略`apply`：

  ```scala
  val bunny = Animal("Hopper")
  ```

- 工厂方法常常通过伴生对象提供，在我们创建实例时，调用工厂方法的`apply`，可以省略关键字`new`。

  ----

  在Chisel中，经常使用工厂方法，例如：

  ```scala
  val myModule = Module(new MyModule)
  ```

### 案例类

案例类是Scala类的一个特别的类型，在Scala编程中十分常见。具有以下特性：

- 对于类的任何参数，外部都可以访问
- 可以不使用`new`来实例化新的对象
- 自动创建一个`unapply`方法，支持对类所有参数的访问
- 不能被继承

```scala
class Nail(length: Int) // Regular class
val nail = new Nail(10) // Requires the `new` keyword
// println(nail.length) // Illegal! Class constructor parameters are not by default externally visible

class Screw(val threadSpace: Int) // By using the `val` keyword, threadSpace is now externally visible
val screw = new Screw(2)          // Requires the `new` keyword
println(screw.threadSpace)

case class Staple(isClosed: Boolean) // Case class constructor parameters are, by default, externally visible
val staple = Staple(false)           // No `new` keyword required
println(staple.isClosed)
```

`Nail`是一个常规的类，它的参数并不能在外部可见，因为在定义时，并没有使用`val`关键字；同时，实例化`Nail`需要`new`关键字。

`Screw` 的声明类似于`Nail`, 但是在参数列表中使用了 `val` 关键字。这使得其字段 `threadSpace`对外部可见的。

使用案例类声明的 `Staple` 其参数对外部均可见。并且，在创建对象的时候，不需要使用`new`关键字，因为Scala自动为每一个`case class`创建了伴生类。

## 在Chisel中使用继承

我们之前使用过Bundle和Module。事实上，每一个我们创建的Chisel模块，都继承自基类`Module`，每一个IO都继承自基类`Bundle`。Chisel的类型`UInt`或者`Bundle`都以`Data`为超类。

### 模块的继承

当我们在Chisel中创建一个硬件模块的时候，我们都需要从`Module`类继承过来。继承不一定是复用的最佳手段，但仍然是一个非常有利的工具。

#### 例子：格雷码编码器和解码器（待续）

```scala
class NoGlitchCounterIO(bitwidth: Int) extends Bundle {
    val en  = Input(Bool())
    val out = Output(UInt(bitwidth.W))
}

abstract class NoGlitchCounter(val maxCount: Int) extends Module {
    val bitwidth: Int
    val io = IO(new NoGlitchCounterIO(bitwidth))
}

abstract class AsyncFIFO(depth: Int) extends Module {
    val io = IO(new Bundle{
        // write inputs
        val write_clock = Input(Clock())
        val write_enable = Input(Bool())
        val write_data = Input(UInt(32.W))

        // read inputs/outputs
        val read_clock = Input(Clock())
        val read_enable = Input(Bool())
        val read_data = Output(UInt(32.W))

        // FIFO status
        val full = Output(Bool())
        val empty = Output(Bool())
    })
    
    def makeCounter(maxCount: Int): NoGlitchCounter

    // add extra bit to counter to check for fully/empty status
    assert(isPow2(depth), "AsyncFIFO needs a power-of-two depth!")
    val write_counter = withClock(io.write_clock) {
        val count = makeCounter(depth * 2)
        count.io.en := io.write_enable && !io.full
        count.io.out
    }
    val read_counter = withClock(io.read_clock) {
        val count = makeCounter(depth * 2)
        count.io.en := io.read_enable && !io.empty
        count.io.out
    }

    // synchronize
    val sync = withClock(io.read_clock) { ShiftRegister(write_counter, 2) }

    // status logic goes here
}
```

