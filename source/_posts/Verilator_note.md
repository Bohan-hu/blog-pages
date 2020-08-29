---
title: Verilator仿真器入门
date: 2020/08/30 01:00:00
tag:
	- Verilator
	- Verilog仿真
catagory:
	- 笔记
---

和我们平时熟悉的`xsim`仿真器不同，Verilator是一款高性能的仿真器，其高性能的秘诀就在于将Verilog代码编译成C++模型进行执行，这样可以达到多线程仿真的效果。

**如果想要理解使用模拟器（例如NEMU）进行差分测试的原理，就必须理解Verilator仿真的机制和流程。**

**本文总结了理解差分测试程序需要的Verilator预备知识。**

## Verilator仿真全流程

1. 类似于GCC的，我们调用`verilator`可执行文件，后接相关的Verilog源文件，`verilator`将Verilog源文件编译为C++模型或者System C模型。在手册中，他们把这个叫做`verilating`，输出的模型叫做`verilated`的模型。
2. Verilate完成之后，并不是万事大吉，用户需要编写一个C++的`wrapper`模块，其中包含`main()`函数，实例化了编译好的模型。
3. 使用C++编译器，将Verilator编译好的模型、用户编写的顶层`wrapper`和Verilator提供的库函数编译成可执行的仿真文件。
4. 执行可执行文件，完成仿真。

## 一个简单的例子

`our.v`:

```verilog
module our;
	initial begin $display("Hello World"); $finish; end
endmodule
```

`sim_main.cpp`:

```c
#include "Vour.h"	// Verilog模块会被编译成Vxxx
#include "verilated.h"
int main(int argc, char **argv, char **env){
	Verilated::commandArgs(argc, argv);			// Verilator仿真运行时参数（和编译的参数不一样，详见Verilator手册第6章
	Vour *top = new Vour;
	while (!Verilated::gotFinish()) { top->eval(); }
	delete top;
	exit(0);
}
```

使用相关的命令进行编译：

```shell
verilator -Wall --cc our.v --exe --build sim_main.cpp
```

## 仿真过程

当我们使用C++模型进行仿真的时候，用户编写的顶层函数必须调用`eval()`或者`eval_step()`和`eval_end_step()`。

当我们在C++层面，仅仅实例化一个模型的时候，只需要调用`designPtr->eval()`，事实上，`eval()`和`eval_step()+eval_end_step()`等效。

**当每次`eval()`被调用的时候，就会执行一次`always @(posedge clk)`语句，计算相应的结果，然后计算组合逻辑。**

## 和C++的交互

对于Verilator而言，会对顶层测试模块创建一个`Vxxx.h`和`Vxxx.c`文件，还有其他的模块文件，此处我们并不关注。

在编译的过程中，还会有`Vxxx.mk`文件，目标文件是`Vxxx_ALL.a`文件，会和用户编写的C++主函数中的循环链接，并创建出仿真的可执行文件。

我们来看以下的代码：

```c
#include <verilated.h> // Defines common routines
#include <iostream> // Need std::cout
#include "Vtop.h" // From Verilating "top.v"

Vtop *top; // 模块的实例
vluint64_t main_time = 0; // 当前的仿真时间
// This is a 64-bit integer to reduce wrap over issues and
// allow modulus. This is in units of the timeprecision
// used in Verilog (or from --timescale-override)
double sc_time_stamp() { // Called by $time in Verilog
    return main_time; // 需要被Verilog中的$time调用
}

int main(int argc, char **argv) {
    Verilated::commandArgs(argc, argv); // Remember args
    top = new Vtop; // 创建一个新的实例
    top->reset_l = 0; // 把reset拉低
    while (!Verilated::gotFinish()) {
        if (main_time > 10) {
            top->reset_l = 1; // Deassert reset		// 拉高Reset，结束复位
        }
        if ((main_time % 10) == 1) {
            top->clk = 1; // 时钟翻转
        }
        if ((main_time % 10) == 6) {
            top->clk = 0; // 时钟翻转
        }
        top->eval(); // 计算输出
        cout << top->out << endl; // Read a output
        main_time++; // 时间戳自增
    }
    top->final(); // 仿真结束的时候，需要运行final()来执行所有的final块
// // (Though this example doesn't get here)
    delete top;
}
```

**由上面代码可见，仿真时如果需要访问模块的顶层信号，可以使用`top->signal`的形式进行访问。**

## DPI（直接编程接口）

### 一个简单的例子

如果我们想在Verilog中调用C语言编写的函数，可以在`our.v`的头部声明：

```verilog
import "DPI-C" function int add (input int a, input int b);
initial begin
	$display("%x + %x = %x", 1, 2, add(1,2));
endtask
```

在Verilator编译之后，将会创建一个`Vour__Dpi.h`文件，里面有以下的extern声明：

```c
extern int add (int a, int b);
```

此后，在其他文件中实现这个`add`函数，记住，需要包含两个头文件：

```c
#include "svdpi.h"
#include "Vour__Dpi.h"
int add(int a, int b) { return a+b; }
```

### DPI系统任务/函数

Verilator还支持将外部的C函数作为系统函数，需要使用`$`作为前缀，但注意，`$`前缀需要被转义：

```verilog
export "DPI-C" function integer \$myRand;
initial $display("myRand=%d", $myRand());
```

---

同样地，我们也可以将Verilog中的task导出为C++函数：

```verilog
export "DPI-C" task publicSetBool;
task publicSetBool;
    input bit in_bool;
    var_bool = in_bool;
endtask
```

在"Verilate"之后，Verilator就会创建一个`Vour__Dpi.h`文件，其中有相应函数的原型：

```c
extern void publicSetBool(svBit in_bool);
```

此时，我们可以在`sc_main.cpp`文件中条用相关的函数了。

```c
#include "Vour__Dpi.h"
publicSetBool(value);
```

或者以如下的形式显式调用：

```c
#include "Vour__Dpi.h"
Vour::publicSetBool(value);
// or top->publicSetBool(value);
```

关于更多DPI的使用方法，可以参考Verilator的官方手册第15章：https://www.veripool.org/ftp/verilator_doc.pdf

