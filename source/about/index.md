---
title: 关于我
date: 2020-08-21 19:21:16
---

## 个人介绍

我是胡博涵，目前就读于哈尔滨工业大学（深圳）计算机科学与技术学院。

感兴趣的方向是计算机系统、计算机体系结构、处理器微架构设计、FPGA等。

## 项目经历

### 基于MIPS指令集的超标量CPU的FPGA实现 

Superscalar HIT Core（简称SHIT Core） 是一个基于MIPS指令集的**乱序四发射**处理器。使用SystemVerilog语言编写，在`xc7a200tfbg676-2`平台上运行频率88MHz。

为哈尔滨工业大学（深圳）1队2020年“龙芯杯”竞赛参赛作品。

其主要特性包括但不局限于：

- 10级**动态调度**超标量流水线
- 4KiB I-Cache
- 16KiB **非阻塞** D-Cache
- 取指宽度2，执行宽度4，提交宽度2
- 基于全局历史的GShare分支预测，基于局部历史的分支目标预测
- 支持精确异常
- 基于Architecture State的状态恢复
- 算术指令与乘除法指令全乱序执行，Load/Store指令半乱序执行
- 基于PRF的寄存器重命名
- 实现部分CP0寄存器和特权指令
- 支持AXI总线通信

项目地址：https://github.com/Superscalar-HIT-Core/SHIT-Core-NSCSCC2020

### 基于MIPS指令集的六级流水线CPU的FPGA实现

实现了基于MIPS指令集的六级流水线单发射标量CPU，使用Verilog语言编写，在`xc7a200tfbg676-2`平台上运行频率110MHz。

其主要特性包括但不局限于：

- 六级流水架构，切分出Cache访问级流水，提升整体运行频率
- 2路组相联的L1指令Cache
- 2路组相联、写回式、写分配式的L1数据Cache
- 基于历史信息的双比特分支预测器
- 全相联的TLB
- 实现了AXI总线接口，集成到龙芯官方的SoC测试平台
- 支持LCD、串口等外设

## 技能

- C / Python
- Verilog / SystemVerilog
- Chisel
- LaTeX