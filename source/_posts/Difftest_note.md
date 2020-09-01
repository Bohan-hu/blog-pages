---
title: Nutshell仿真机制解析
date: 2020/08/31 00:00:00
tag:
	- 测试
catagory:
	- 测试
---

近期接触了国科大的Nutshell项目，深感其差分测试思想的精妙之处，非常感谢国科大的教学团队为CPU测试提供了一个创新的思路。

本文记录了笔者学习差分测试源代码时的一些收获与想法。

要理解差分测试的源代码，首先需要理解Verilator仿真器的仿真机制。笔者在学习Verilator的过程中，也记录了相关的笔记：https://hubohan.space/2020/08/30/Verilator_note/

### 差分测试的总体思想

在参加“龙芯杯”竞赛的时候，我们接触了基于`trace`比对的测试机制，让DUT和黄金模型同样执行一段程序，捕捉黄金模型处理器状态改变的轨迹，即哪条指令(PC)在何时(trace中的顺序)往哪个寄存器(`wnum`)写入了什么值(`wbdata`)，然后将DUT执行的轨迹和黄金模型执行的轨迹逐条比对，遇到出错点即停止执行，这其中其实也用到了差分测试的思路。

Easydiff是Nutshell采用的测试框架，它将这个差分测试的思想更进了一步，`trace`比对机制是将处理器状态改变的**轨迹**进行比对，而Easydiff是将处理器的**全状态**进行比对，包括GPR、控制寄存器和PC。毫无疑问地，这个比对的粒度更细，能够更精确地定位到错误点。

在Easydiff中，DUT是我们自己实现的CPU，黄金模型是南京大学的NEMU模拟器。Easydiff在DUT状态改变的时候，立即让NEMU模拟器执行和DUT相同的指令，并且比对其状态，若有差异，立刻报错停止。

在Easydiff框架中，采用的仿真器是Verilator，其将Verilog源码构建为C++描述的仿真模型，而C++语言相较Verilog，具有更大的灵活性。项目提供的NEMU模拟器为动态链接库的形式，能够在运行时灵活地调用其API，进行动态链接。

为了屏蔽NEMU的实现细节，将NEMU提供的API摘录如下：

| API                     | 说明                   |
| ----------------------- | ---------------------- |
| ref_difftest_getregs    | 从NEMU中获取寄存器状态 |
| ref_difftest_setregs    | 设置NEMU的寄存器状态   |
| ref_difftest_exec       | NEMU执行n条指令        |
| ref_difftest_raise_intr | 触发NEMU的中断         |

在仿真阶段，NEMU的API通过动态链接的形式在仿真驱动代码`main.cpp`以及其下层函数中被调用，使用者无需关注其实现细节。

### 观察生成的Verilog顶层模型

- 使用`make verilog`生成相应的Verilog文件，最终生成的代码在`build/Topmain.v`里面

  ![image-20200829214644319](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200829214644319.png)

其中可以看到，除了NutShell的顶层模块之外，还有`SDHelper.v`，`UARTGetc.v`和`FBHelper.v`三个文件，这三个文件是使用了Verilog与C语言的DPI接口，使用C语言描述相关的仿真行为，并在Verilog中进行调用，我们可以在文件中找到相关的细节：

```verilog
import "DPI-C" function void uart_getc(output byte ch);
```

为什么要这样写呢？因为仿真时可能会用到串口输入，而我们的控制台明显无法仿真串口，需要用C语言编写一个“假”的串口来进行仿真，而这个“串口”可以读取用户的键盘输入，并且将DUT的输出显示在控制台上。

关于DPI-C的接口使用，稍后会进行介绍。

同时注意到，仿真所使用的顶层模块`NutShellSimTop`，其暴露给外部有以下接口：

```verilog
module NutShellSimTop(
  input         clock,
  input         reset,
  output [63:0] io_difftest_r0,
  output [63:0] io_difftest_r1,
  output [63:0] io_difftest_r2,
  // ...........................中间略 ............................
  output [63:0] io_difftest_r30,
  output [63:0] io_difftest_r31,
  output        io_difftest_commit,
  output        io_difftest_isMultiCommit,
  output [63:0] io_difftest_thisPC,
  output [31:0] io_difftest_thisINST,
  output        io_difftest_isMMIO,
  output        io_difftest_isRVC,
  output        io_difftest_isRVC2,
  output [63:0] io_difftest_intrNO,
  output [1:0]  io_difftest_priviledgeMode,
  output [63:0] io_difftest_mstatus,
  output [63:0] io_difftest_sstatus,
  output [63:0] io_difftest_mepc,
  output [63:0] io_difftest_sepc,
  output [63:0] io_difftest_mcause,
  output [63:0] io_difftest_scause,
  input  [63:0] io_logCtrl_log_begin,
  input  [63:0] io_logCtrl_log_end,
  input  [63:0] io_logCtrl_log_level,
  output        io_difftestCtrl_enable
);
```

这些接口是暴露给编写的`difftest`函数使用的，目的在于进行对比。


### `main.c` - 开始的地方

```c++
int main(int argc, const char** argv) {
  auto emu = Emulator(argc, argv);					// <--------------- PAY ATTENTION TO IT

  get_sc_time_stamp = [&emu]() -> double {			// 匿名函数，lambda表达式
    return emu.get_cycles();
  };

  emu.execute();

  extern uint32_t uptime(void);
  uint32_t ms = uptime();

  int display_trapinfo(uint64_t max_cycles);
  int ret = display_trapinfo(emu.get_max_cycles());
  eprintf(ANSI_COLOR_BLUE "Guest cycle spent: %" PRIu64 "\n" ANSI_COLOR_RESET, emu.get_cycles());
  eprintf(ANSI_COLOR_BLUE "Host time spent: %dms\n" ANSI_COLOR_RESET, ms);

  return ret;
}
```

我们看到，主循环中，构造了一个模拟器`Emulator`，并且根据Verilator的要求，需要自行构造一个`get_cycle`的函数，以供Verilog中的`$time`任务调用。

那么需要熟悉的就是`Emulator`和里面的`execute`函数了。

### `emu.h` - 解构`Emulator`类

```cpp
class Emulator {
  const char *image;                            // 镜像路径
  std::shared_ptr<VNutShellSimTop> dut_ptr;     // Verilator模型
  VerilatedVcdC* tfp;							// Verilator波形（由宏定义进行开关）

  // 仿真器控制的相关变量
  uint32_t seed;
  uint64_t max_cycles, cycles;
  uint64_t log_begin, log_end, log_level;
  // 部分辅助函数，此处无需关注
  std::vector<const char *> parse_args(int argc, const char *argv[]);
  static const struct option long_options[];
  static void print_help(const char *file);
  
  void read_emu_regs(rtlreg_t *r);   // 从DUT顶层模块读取相关寄存器

  public:
  // Emulator的构造函数
  Emulator(int argc, const char *argv[]);
  // n个周期的复位
  void reset_ncycles(size_t cycles);
  // 步进单个周期
  void single_cycle();
  // 执行n个周期
  void execute_cycles(uint64_t n);
  // 测试Cache的函数
  void cache_test(uint64_t n);
  // 执行，同时进行difftest
  void execute();
  // 返回当前的周期数，用于Verilog的$time任务
  uint64_t get_cycles();
  // 返回最大的周期数，是通过构造函数设置的，防止卡死
  uint64_t get_max_cycles();
};
```

#### `read_emu_regs`

```cpp
  void read_emu_regs(rtlreg_t *r) {   // 从DUT顶层模块读取相关寄存器
#define macro(x) r[x] = dut_ptr->io_difftest_r_##x
    macro(0); macro(1); macro(2); macro(3); macro(4); macro(5); macro(6); macro(7);
    macro(8); macro(9); macro(10); macro(11); macro(12); macro(13); macro(14); macro(15);
    macro(16); macro(17); macro(18); macro(19); macro(20); macro(21); macro(22); macro(23);
    macro(24); macro(25); macro(26); macro(27); macro(28); macro(29); macro(30); macro(31);
    r[DIFFTEST_THIS_PC] = dut_ptr->io_difftest_thisPC;
#ifndef __RV32__                    // 读取CSR寄存器，我们在顶层编译选项中定义的是RV64
    r[DIFFTEST_MSTATUS] = dut_ptr->io_difftest_mstatus;
    r[DIFFTEST_SSTATUS] = dut_ptr->io_difftest_sstatus;
    r[DIFFTEST_MEPC   ] = dut_ptr->io_difftest_mepc;
    r[DIFFTEST_SEPC   ] = dut_ptr->io_difftest_sepc;
    r[DIFFTEST_MCAUSE ] = dut_ptr->io_difftest_mcause;
    r[DIFFTEST_SCAUSE ] = dut_ptr->io_difftest_scause;
#endif
  }
```

将宏定义展开之后，我们看到，这个函数其实是依次将相关的寄存器dump出来，按顺序放入了一个`rtlreg_t`类型的变量中。（事实上，`rtlreg_t`变量也就是`uint64`的数组。如果是RV32，那么就是`uint32`，这个根据宏定义来开关，而宏定义是在编译的时候，通过编译选项指定的，详见Makefile）。

其中，剩下的例如`DIFFTEST_THIS_PC`等下标，在`difftest.h`中定义如下，PC作为32号，其余以此类推：

```cpp
enum {
  DIFFTEST_THIS_PC = 32,
#ifndef __RV32__
  DIFFTEST_MSTATUS,
  DIFFTEST_MCAUSE,
  DIFFTEST_MEPC,
  DIFFTEST_SSTATUS,
  DIFFTEST_SCAUSE,
  DIFFTEST_SEPC,
#endif
  DIFFTEST_NR_REG
};
```

#### 初始化 - 构造函数

```cpp
    auto args = parse_args(argc, argv);

    // srand
    srand(seed);
    srand48(seed);
	// 随机复位
    Verilated::randReset(2);

    // 设置日志的起始时间和日志级别
    dut_ptr->io_logCtrl_log_begin = log_begin;
    dut_ptr->io_logCtrl_log_end = log_end;
    dut_ptr->io_logCtrl_log_level = log_level;

    // 使用镜像来初始化ram
    extern void init_ram(const char *img);
    init_ram(image);

    // 初始化外部设备
    extern void init_device(void);
    init_device();

    // init core
    reset_ncycles(10);
```

#### `reset_ncycles` - 重置n个周期

```
    for(int i = 0; i < cycles; i++) { // 重置n个周期
      dut_ptr->reset = 1;
      dut_ptr->clock = 0;
      dut_ptr->eval();
      dut_ptr->clock = 1;
      dut_ptr->eval();
      dut_ptr->reset = 0;
    }
```

#### `single_cycle()` - 前进一个时钟周期（单步）

```cpp
    dut_ptr->clock = 0;
    dut_ptr->eval();

    dut_ptr->clock = 1;
    dut_ptr->eval();

#if VM_TRACE
    tfp->dump(cycles);
#endif

    cycles ++;
```

- 此处使用了Verilator手册中提到的`eval()`。

- 当设置了时钟下降沿之后，调用`eval()`计算组合逻辑，设置了上升沿之后，调用`eval()`更新时序逻辑。

- 这里维护的`cycles`变量是为了给Verilog的`$time`任务使用，Verilator规定这个需要由用户维护。

#### `execute_cycles` - 最核心的部分

```cpp
  void execute_cycles(uint64_t n) {
    extern bool is_finish();
    extern void poll_event(void);
    extern uint32_t uptime(void);
    extern void set_abort(void);
    uint32_t lasttime = 0;
    uint64_t lastcommit = n;
    int hascommit = 0;
    const int stuck_limit = 2000;			// 超过2000周期没有指令提交，可能会是卡住了

#if VM_TRACE
    Verilated::traceEverOn(true);	// Verilator must compute traced signals
    VL_PRINTF("Enabling waves...\n");
    tfp = new VerilatedVcdC;
    dut_ptr->trace(tfp, 99);	// Trace 99 levels of hierarchy
    tfp->open("vlt_dump.vcd");	// Open the dump file
#endif

    while (!is_finish() && n > 0) {	// 主执行循环
      single_cycle();
      n --;

      if (lastcommit - n > stuck_limit && hascommit) { // 如果太久都没有指令提交，那说明有可能卡住了，强制停止
        eprintf("No instruction commits for %d cycles, maybe get stuck\n"
            "(please also check whether a fence.i instruction requires more than %d cycles to flush the icache)\n",
            stuck_limit, stuck_limit);
#if VM_TRACE
        tfp->close();
#endif
        set_abort();
      }

      if (!hascommit && (uint32_t)dut_ptr->io_difftest_thisPC == 0x80000000) {			// 开始执行，初始PC=80000000
        // 
        hascommit = 1;
        extern void init_difftest(rtlreg_t *reg);
        rtlreg_t reg[DIFFTEST_NR_REG];
        read_emu_regs(reg);
        init_difftest(reg);
      }

      // difftest
      if (dut_ptr->io_difftest_commit && hascommit) {		// 有提交的时候，就需要进行差分测试了
        // 将里面的所有reg都提取出来
        rtlreg_t reg[DIFFTEST_NR_REG];						
        read_emu_regs(reg);

        extern int difftest_step(rtlreg_t *reg_scala, uint32_t this_inst,
          int isMMIO, int isRVC, int isRVC2, uint64_t intrNO, int priviledgeMode, int isMultiCommit);		
          // 这里有一个参数，如果是双提交的话，需要检测两次
        if (dut_ptr->io_difftestCtrl_enable) {
          if (difftest_step(reg, dut_ptr->io_difftest_thisINST,				// <----------- PAY ATTENTION TO THIS FUNCTION !!!
              dut_ptr->io_difftest_isMMIO, dut_ptr->io_difftest_isRVC, dut_ptr->io_difftest_isRVC2,
              dut_ptr->io_difftest_intrNO, dut_ptr->io_difftest_priviledgeMode, 
              dut_ptr->io_difftest_isMultiCommit)) {
#if VM_TRACE
            tfp->close();
#endif
            set_abort();
          }
        }
        lastcommit = n;
      }

      uint32_t t = uptime();
      if (t - lasttime > 100) {
        poll_event();
        lasttime = t;
      }
    }
  }
```

- `difftest_step()`
- `poll_event()`

- 在开始仿真的时候，其实NEMU的状态是不确定的，需要在Reset之后，**用我们编写的CPU的初始状态去校准NEMU的初始状态，他们一致之后，才会开始真正的仿真。**

#### `difftest_step()` - 灵魂

```cpp
int difftest_step(rtlreg_t *reg_scala, uint32_t this_inst,
  int isMMIO, int isRVC, int isRVC2, uint64_t intrNO, int priviledgeMode, 
  int isMultiCommit
  ) {

  // Note:
  // reg_scala[DIFFTEST_THIS_PC] is the first PC commited by CPU-WB
  // ref_r[DIFFTEST_THIS_PC] is NEMU's next PC
  // To skip the compare of an instruction, replace NEMU reg value with CPU's regfile value,
  // then set NEMU's PC to next PC to be run

  #define DEBUG_RETIRE_TRACE_SIZE 16

  rtlreg_t ref_r[DIFFTEST_NR_REG];
  rtlreg_t this_pc = reg_scala[DIFFTEST_THIS_PC];
  // ref_difftest_getregs() will get the next pc,
  // therefore we must keep track this one
  // 需要注意的是：NEMU的PC是NEMU将要执行的下一条指令的PC
  // ref_difftest_getregs() 拿到的PC是NEMU计算出的下一个PC
  // 为了避免对比出错，我们需要用当前CPU执行的PC去覆盖读出的PC
  // 为了避免丢失原有的PC，我们需要将NEMU拿出来的下一个PC进行保存，保存到nemu_this_pc中
  static rtlreg_t nemu_this_pc = 0x80000000;
  // 这个循环队列，是用于记录之前的轨迹用的
  static rtlreg_t pc_retire_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t inst_retire_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t multi_commit_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t skip_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static int pc_retire_pointer = 7;
  static int need_copy_pc = 0;
  #ifdef NO_DIFFTEST
  return 0;
  #endif
  // Copy PC是什么机制？
  // 这个代码块的作用，猜测是仅在双提交+MMIO的情况下有用
  // 通过在代码中添加assert断言，确定其确实是在双提交的时候才有用
  // 在双提交，且为MMIO+跳转的模式下，下一个PC不应当是PC+8，而应当是跳转的目标
  // 我们假设DUT计算的跳转的目标永远是正确的，直接将其拷贝给NEMU
  if (need_copy_pc) {
    need_copy_pc = 0;
    ref_difftest_getregs(&ref_r);
    nemu_this_pc = reg_scala[DIFFTEST_THIS_PC];
    ref_r[DIFFTEST_THIS_PC] = reg_scala[DIFFTEST_THIS_PC];
    ref_difftest_setregs(ref_r);
  }
  if (isMMIO) {				
    // 对于MMIO，直接不对比，NEMU中的MMIO关系我们暂时无法干涉，所以直接跳过去，并且将执行完毕MMIO指令的CPU状态，拷贝到NEMU里面
    // printf("diff pc: %x isRVC %x\n", this_pc, isRVC);
    // MMIO accessing should not be a branch or jump, just +2/+4 to get the next pc
    int pc_skip = 0;
    // 我们为什么很肯定地能认为NEMU的下一个PC是PC+4呢？
    // 在单提交的情况下，MMIO相关的不可能是跳转指令，所以PC+4一定是下一个PC
    // 所以直接将当前DUT的状态复制给NEMU，并且将NEMU的PC+4
    // 双提交的情况下，一定会是PC+8吗？这里需要多加考虑。
    pc_skip += isRVC ? 2 : 4;
    pc_skip += isMultiCommit ? (isRVC2 ? 2 : 4) : 0;
    reg_scala[DIFFTEST_THIS_PC] += pc_skip;
    nemu_this_pc += pc_skip;
    // to skip the checking of an instruction, just copy the reg state to reference design
    ref_difftest_setregs(reg_scala);        // 把状态拷贝到里面去
    // 设置相关的队列
    pc_retire_pointer = (pc_retire_pointer+1) % DEBUG_RETIRE_TRACE_SIZE;
    pc_retire_queue[pc_retire_pointer] = this_pc;
    inst_retire_queue[pc_retire_pointer] = this_inst;
    multi_commit_queue[pc_retire_pointer] = isMultiCommit;
    skip_queue[pc_retire_pointer] = isMMIO;
    // 标记下一次需要覆盖PC（仅针对MMIO+跳转的双提交情形）
    need_copy_pc = 1;
    return 0;																														// NEMU并不执行，直接return
  }


  if (intrNO) {																// 如果产生了中断或者异常，则NEMU也需要获知相关的信息
    ref_difftest_raise_intr(intrNO);
  } else {
    ref_difftest_exec(1);
  }

  if (isMultiCommit) {    
    ref_difftest_exec(1);
    // 如果是多提交，需要再运行一步
  }

  ref_difftest_getregs(&ref_r);

  rtlreg_t next_pc = ref_r[32];
  pc_retire_pointer = (pc_retire_pointer+1) % DEBUG_RETIRE_TRACE_SIZE;
  pc_retire_queue[pc_retire_pointer] = this_pc;
  inst_retire_queue[pc_retire_pointer] = this_inst;
  multi_commit_queue[pc_retire_pointer] = isMultiCommit;
  skip_queue[pc_retire_pointer] = isMMIO;
  
  int isCSR = ((this_inst & 0x7f) ==  0x73);
  int isCSRMip = ((this_inst >> 20) == 0x344) && isCSR;
  if (isCSRMip) {
    // We can not handle NEMU.mip.mtip since it is driven by CLINT,
    // which is not accessed in NEMU due to MMIO.
   // Just sync the state of NEMU from NutCore.
    reg_scala[DIFFTEST_THIS_PC] = next_pc;
    nemu_this_pc = next_pc;
    ref_difftest_setregs(reg_scala);
    return 0;
  }

  // replace with "this pc" for checking
  // 同步NEMU的PC
  ref_r[DIFFTEST_THIS_PC] = nemu_this_pc;
  // 保存下一个PC
  nemu_this_pc = next_pc;

  ref_r[0] = 0;
	
    
  // 检查上面获得的结果，如果有误，打印出最近DEBUG_RETIRE_TRACE_SIZE条
  if (memcmp(reg_scala, ref_r, sizeof(ref_r)) != 0) {
    printf("\n==============Retire Trace==============\n");
    int j;
    for(j = 0; j < DEBUG_RETIRE_TRACE_SIZE; j++){
      printf("retire trace [%x]: pc %010lx inst %08x %s %s %s\n", j, 
        pc_retire_queue[j], 
        inst_retire_queue[j], 
        (multi_commit_queue[j])?"MC":"  ", 
        (skip_queue[j])?"SKIP":"    ", 
        (j==pc_retire_pointer)?"<--":""
      );
    }
    printf("\n==============  Reg Diff  ==============\n");
    ref_isa_reg_display();
    printf("priviledgeMode = %d\n", priviledgeMode);
    puts("");
    int i;
    for (i = 0; i < DIFFTEST_NR_REG; i ++) {
      if (reg_scala[i] != ref_r[i]) {
        printf("%s different at pc = 0x%010lx, right= 0x%016lx, wrong = 0x%016lx\n",
            reg_name[i], this_pc, ref_r[i], reg_scala[i]);
      }
    }
    return 1;
  }
  return 0;
}

```

**需要注意的有以下几个逻辑：**

- NEMU中的PC寄存器，指向的是**下一个**将要执行的指令的PC
- 进入`difftest_step()`函数时，NEMU的状态比DUT的状态**滞后**一条指令，但是**NEMU的PC和DUT是相同的**
- 在NEMU单步执行一个周期之后，NEMU的通用寄存器和控制寄存器应当和DUT相同，但是NEMU的PC较DUT超前了一条指令。

---

- 最重要的是需要理解**PC跳过的逻辑**，具体内容在代码中有详尽的注释。
- 对比一般流程如下：
  - CPU提交一条指令
  - 调用`difftest_step()`
  - 保存NEMU的PC(nemu_this_pc)
  - NEMU单步执行
  - 对比
- 如果是有需要跳过的指令
  - CPU提交一条指令
  - 调用`difftest_step()`
  - 拷贝CPU的状态给NEMU
  - 更新NEMU的PC
  - 返回

##### `need_copy_pc`到底有啥用？

笔者在阅读源代码时，一直不理解这个`need_copy_pc`标志有什么用。

`need_copy_pc`标志仅在`MMIO`这个分支内会被设置。

某天突然想到，之前实现的乱序处理器中，MMIO和跳转指令可能会同时提交，如果跳转指令发生了跳转，那么接下来，不可能会在PC+8的地方继续取指。那么在这种情况下，在`MMIO`这个分支中为NEMU设置的PC就不对了！

因为CPU在提交完MMIO和跳转指令这两条指令之后，接下来的PC不再是PC+8，而是会跳到目标地址，此时，`need_copy_pc`就派上用场了。根据作者的注释，在这种情况下，认为CPU计算的目标地址是正确的。

> 笔者认为，此处的对比似乎不是那么严谨，如果需要跳过指令，并且双提交的情况，一个改进的措施是去除`need_copy_pc`这个标志，转而在CPU暴露的接口中指明跳转指令所在的PC，进而让NEMU也去执行对应的跳转指令，这样能够将测试覆盖的更加全面。
>
> 并且如果DUT为顺序标量处理器，是否可以使用宏来关闭这个`need_copy_pc`机制？因为在顺序标量的核中，这个选项确实是多余的，但是每次碰到MMIO，都会去执行这个分支，会不会造成效率的下降？

###### 猜测与验证

猜测：MMIO和跳转双提交的时候，会进入到这个条件之中，因为NEMU的PC需要校正。

为了验证自己的想法，笔者在关键部分添加了断言：

![](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200901004921682.png)

- 通过在此处添加断言，发现`need_copy_pc`这个条件似乎是多余的，那么事实是这样吗？
- 虽然每一次进入这个分支的时候，都复制了PC给NEMU，但是这个复制似乎是“多余”的，因为每次`assert`中的条件都成立。

- 在后面添加断言语句，断言是否有双提交，发现确实没有双提交

![](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200901004321667.png)

---

进一步地，修改Nutshell的配置，配置为顺序双发射和乱序多发射。

- 在顺序双发射的核中，也不存在MMIO与跳转指令同时提交的情况，唯有在乱序核中会有这种情况，所以当将参数调到如下情形时，断言才有不成立的可能。

![](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200901010424899.png)


![image-20200901010449629](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200901010449629.png)

---

**结论：`need_copy_pc`仅在双提交，并且双提交为跳转指令+MMIO时才起作用。**



### `device.cpp, ram.cpp, sdcard.cpp, uart.cpp, vga.cpp等` - DPI-C接口

Nutshell的仿真事实上也接近于**全系统仿真**，也就是说不仅仅仿真CPU核心，还需要同步仿真SoC上面的外设，例如内存、I/O设备。因此，Nutshell的源代码中提供了相应的C++编写的仿真模型。

一个很好的例子是串口模型。使用Verilog编写串口仿真模型，往往会比较困难，并且在命令行下难以与用户进行交互。而Verilog提供了DPI-C接口，可以在Verilog中调用编写好的C语言函数。

在`src/main/scala/device`中，是Nutshell提供的AXI外设模型，我们以串口模型为例：

```scala
class UARTGetc extends BlackBox with HasBlackBoxInline {
  val io = IO(new Bundle {
    val clk = Input(Clock())
    val getc = Input(Bool())
    val ch = Output(UInt(8.W))
  })

  setInline("UARTGetc.v",
    s"""
      |import "DPI-C" function void uart_getc(output byte ch);
      |
      |module UARTGetc (
      |  input clk,
      |  input getc,
      |  output reg [7:0] ch
      |);
      |
      |  always@(posedge clk) begin
      |    if (getc) uart_getc(ch);
      |  end
      |
      |endmodule
     """.stripMargin)
}
```

可以看到的是，在这个`UARTGetc`模块中，关联了外部的`uart_getc`函数，这个函数在控制台中能够接收用户输入，将输入存放在缓冲区中，当调用`uart_getc`函数的时候，能够返回缓冲区首部的字符，这里在Verilog中调用了C语言函数，能够直接将返回的字符传送给上层的硬件模块进行仿真。

```scala
class AXI4UART extends AXI4SlaveModule(new AXI4Lite) {
  val rxfifo = RegInit(0.U(32.W))
  val txfifo = Reg(UInt(32.W))
  val stat = RegInit(1.U(32.W))
  val ctrl = RegInit(0.U(32.W))

  val getcHelper = Module(new UARTGetc)
  getcHelper.io.clk := clock
  getcHelper.io.getc := (raddr(3,0) === 0.U && ren)

  def putc(c: UInt): UInt = { printf("%c", c(7,0)); c }
  def getc = getcHelper.io.ch
```

那么串口输出就比较简单了，直接调用`scala`的输出函数就可以实现。

有关于DPI-C以及相关仿真模型的细节，由于时间所限，此处不一一描述。