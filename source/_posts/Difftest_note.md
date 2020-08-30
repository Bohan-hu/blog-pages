## Nutshell仿真流程解析

- 生成Verilog
- 自己编写的main函数，调用Verilator生成的仿真模型，并动态链接已经编译好的二进制版本的NEMU



### 观察生成的Verilog顶层模型

- 使用`make verilog`生成相应的Verilog文件，最终生成的代码在`build/Topmain.v`里面

  ![image-20200829214644319](https://cdn.jsdelivr.net/gh/Bohan-Hu/img/images/image-20200829214644319.png)

其中可以看到，除了NutShell的顶层模块之外，还有`SDHelper.v`，`UARTGetc.v`和`FBHelper.v`三个文件，这三个文件是使用了Verilog与C语言的DPI接口，使用C语言描述相关的仿真行为，并在Verilog中进行调用，我们可以在文件中找到相关的细节：

```verilog
import "DPI-C" function void uart_getc(output byte ch);
```

为什么要这样写呢？因为仿真时可能会用到串口输入，而我们的控制台明显无法仿真串口，正如yzh老师所说，需要用C语言编写一个“假”的串口来进行。

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




### `main.c` - 梦开始的地方

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

我们看到，主循环中，构造了一个模拟器`Emulator`，并且根据Verilator的要求，需要自行构造一个`get_cycle`的函数。

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

#### `single_cycle()` - 前进一个时钟周期

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

此处使用了Verilator中提到的`eval()`，当设置了时钟下降沿之后，调用`eval()`计算组合逻辑，设置了上升沿之后，调用`eval()`更新时序逻辑。

这里维护的`cycles`变量是为了给Verilog的`$time`任务使用，Verilator规定这个需要由用户维护。

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
          int isMMIO, int isRVC, int isRVC2, uint64_t intrNO, int priviledgeMode, int isMultiCommit);		// 这里有一个参数，如果是双提交的话，需要检测两次
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
  static rtlreg_t nemu_this_pc = 0x80000000;
  static rtlreg_t pc_retire_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t inst_retire_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t multi_commit_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static uint32_t skip_queue[DEBUG_RETIRE_TRACE_SIZE] = {0};
  static int pc_retire_pointer = 7;
  static int need_copy_pc = 0;
  #ifdef NO_DIFFTEST
  return 0;
  #endif

  if (need_copy_pc) {
    need_copy_pc = 0;
    ref_difftest_getregs(&ref_r); 							// 将参考实现中的寄存器们存进来
    nemu_this_pc = reg_scala[DIFFTEST_THIS_PC]; 			// 然后将原来的PC保存
    ref_r[DIFFTEST_THIS_PC] = reg_scala[DIFFTEST_THIS_PC];	// 然后把我们实现中的pc复制过去
    ref_difftest_setregs(ref_r);							// 最后把相应reference的状态更改
  }

  if (isMMIO) {
    // printf("diff pc: %x isRVC %x\n", this_pc, isRVC);
    // MMIO accessing should not be a branch or jump, just +2/+4 to get the next pc
    int pc_skip = 0;
    pc_skip += isRVC ? 2 : 4;
    pc_skip += isMultiCommit ? (isRVC2 ? 2 : 4) : 0;
    reg_scala[DIFFTEST_THIS_PC] += pc_skip;
    nemu_this_pc += pc_skip;
    // to skip the checking of an instruction, just copy the reg state to reference design
    // 如果需要跳过某个指令的检查，那么直接将寄存器的状态复制到reference里面去就可以了
    ref_difftest_setregs(reg_scala);
    pc_retire_pointer = (pc_retire_pointer+1) % DEBUG_RETIRE_TRACE_SIZE;
    pc_retire_queue[pc_retire_pointer] = this_pc;
    inst_retire_queue[pc_retire_pointer] = this_inst;
    multi_commit_queue[pc_retire_pointer] = isMultiCommit;
    skip_queue[pc_retire_pointer] = isMMIO;
    need_copy_pc = 1;
    return 0;
  }


  if (intrNO) {
    ref_difftest_raise_intr(intrNO);
  } else {
    ref_difftest_exec(1);
  }

  if (isMultiCommit) {    
    ref_difftest_exec(1);
    // exec 1 more cycle
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
  ref_r[DIFFTEST_THIS_PC] = nemu_this_pc;
  nemu_this_pc = next_pc;

  ref_r[0] = 0;
	
    
  // 检查上面获得的结果
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

- 这个函数，最重要的是需要理解**上面PC跳过的逻辑（TBC）**。
- 理解几个API的实现，那么还是需要去参考NEMU的手册（TODO）。



### `device.cpp, ram.cpp, sdcard.cpp, uart.cpp, vga.cpp等` - 使用C++编写的外设仿真模型

TBC