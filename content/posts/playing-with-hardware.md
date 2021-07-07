---
title: "软件开发者的硬件开发之旅"
date: 2021-07-06T13:12:09+08:00
draft: false
tags:
- Verilog
- SystemVerilog
- hardware
- FPGA
categories:
- hardware
summary: 最近忙于学习硬件开发，这篇post来谈谈一个软件开发者眼中的硬件开发吧。
---

# 背景

最近这半年，没有更新blog，主要是因为忙于学习硬件开发，这篇post就来谈谈一个软件开发者眼中的硬件开发吧。

之前读书的时候也学过电子电路，数字逻辑，计算机组成原理等一些专业课，当时也没有太多关注硬件相关的开发，主要钻研软件开发的技术，入了C++的坑。后来，在做逆向的时候，也算是在汇编的泥潭里面摸爬滚打过，对汇编语言虽然说不上精通，但是也算有了一定的了解。之后，逐渐参与了一些FPGA相关的项目，期间也为FPGA写过driver，对硬件的了解也算是深入到了原理图，寄存器和时序图这个层级。

这一次，由于项目需要实现一个HDMI多屏同步的功能，对实时性要求非常高。虽然我们之前在软件层面已经做到了frame级别（33ms），而这次我们的目标是scanline级别（30us），通过现有的软件根本做不到。为了达成目标，我们最终选择了FPGA。而我们团队中没有人对FPGA开发有任何经验，我只好硬着头皮上了。

经过了半年的学习和开发，目前基本功能已经可以正常使用，还有一些收尾的工作正在进行，终于有时间来总(tu)结(cao)一下了。

# 开发语言的选择

之前做过一些FPGA相关的工作，硬件的同事主要使用的是VHDL，现在Verilog已经是主流了，VHDL的项目也越来越少，再加上Verilog号称与C语言比较类似，一开始就选择了Verilog作为主要开发语言。但是在开发的过程中，发现Verilog基本等同于软件开发的汇编语言了，根本没有数据结构的概念，只有reg和wire，而且还需要手工区分wire和reg类型，作为用惯了高级语言的我来说，感觉实在太别扭了。于是从Verilog转到了SystemVerilog，才感觉从史前到了古代。

为什么说SystemVerilog是古代呢？SystemVerilog虽然比Verilog要稍微好用一点，但是也仅仅也就只是好一点而已了。SystemVerilog大部分改进都只能在simulation的时候使用，synthesis的时候是不支持的。

另外，也简单看了下chisel等号称下一代HDL的语言，发现基本也是没太大改进。

# RTL

之前没有接触数字逻辑开发的时候，RTL（寄存器传输级）这个术语我不太能理解，为什么是“寄存器”传输呢。入门后，我才恍然大悟，RTL的开发也真的就是只能在Register这个层级上捣鼓了。

组合逻辑（combinational logic）电路进行逻辑运算，在时钟边沿将结果寄存到DFF（D flip flop）中，DFF就是寄存器了。FF的输出结果作为下一级组合逻辑电路的输入，在时钟的驱动下，不断向下一级传输，就构成了RTL。

FF用于存储状态，组合逻辑电路用来将输入转换到输出，时钟用于驱动状态的变换，写RTL也就是搭建组合逻辑电路和寄存器构成的数字电路。

在HDL（Hardware Description Language）出现之前，大家都是手工搭建电路的，有种侏罗纪的感觉。出现了HDL后，终于可以不用画图了，综合工具（相当于软件里面的编译器）可以帮你把加法运算直接综合成加法器了，不用手工搭加法器电路了。不过，综合工具也就只能支持加法，位运算这些简单的操作了，乘法和除法还是不支持或者支持有限。

那么，在软件中的乘法和除法是怎么做的呢？乘法可以展开为加法或者移位，每个时钟周期做一次加法或者移位，将中间结果保存在寄存器中，多做几个时钟周期就可以了。

# 状态机

在编写HDL时，除了最简单的组合逻辑情况，比较复杂的设计中都会使用到FSM（有限状态机）。而不同状态的触发条件，基本上都是通过计数器来实现。
RTL的最基本单元就是FSM，不管你是使用计算器的结果作为状态转移的条件，还是使用特定的输入作为条件，本质上都是状态机。

# CDC（Clock Domain Crossing）

如果所有的DFF都工作在同一个时钟域，那就是最简单的情况。但是，考虑到功耗，组合逻辑电路的复杂度不同导致的延迟差异以及不同的传输协议等问题，基本上一个设计中都包含多个不同的时钟，用于驱动不同的部分。比如memory的时钟与HDMI的时钟必然不可能使用同一个时钟。

而多个不同时钟之间的数据传输，有可能出现亚稳态，怎么避免亚稳态就构成了典型的CDC问题。CDC常见的处理方式有三种：打两（N）拍，握手机制和使用FIFO。

CDC导致的第二个问题，就是STA（静态时序分析）是无法对CDC进行正确处理的，因此需要对其进行特殊处理，要么将其路径打断（set_false_path），要么设置为多周期（set_multicycle_path)。

# STA（Static Timing Analysis）

* 为什么一个数字逻辑电路无法跑到更高的频率？
    正如我们上面所述，数字逻辑电路中最重要的组成部分是组合逻辑电路和DFF，组合逻辑电路从输入到稳定输出是需要时间的，越复杂的电路耗时越长，而且在输出稳定前可能会出现毛刺；DFF的记录状态变化和输出之前的状态也是需要时间的，这两部分耗时加起来就约束了数字逻辑电路的最小时钟周期，从而也约束了其运行的最高频率。

* 我们怎么才能得到一个数字逻辑电路的最高频率呢？
    STA就是用来帮助我们分析这个问题的工具，STA工具通过用户提供的约束条件和器件固有的延时，计算出数据传输路径上DFF的setup time和recovery time，当发生时序违例的时候，我们就能够有针对性的调整一些关键路径上的实现，进一步优化时序，提高电路运行的频率。

# FPGA

FPGA怎么做到可编程的呢？学过数字逻辑的同学都知道，所有的组合逻辑可以用LUT（lookup table）来完成。一般来说，FPGA中会将4输入LUT加上一个DFF构成一个基本的LE（逻辑单元），每个LE都可以进行对应的LUT的配置和是否使用DFF，再用连线将LE连接起来，因此就能够将我们前面所述的RTL转换到一个开关表（bitstream），表项对应每个LE内部的LUT/DFF是否使能，LE之间是否连线。

在FPGA开发中，Synthesis将HDL转换为通用的硬件表示，对应到ASIC开发的前端流程，Placement&Routing则将通用的硬件表示映射为特定FPGA硬件中的LE及LE之间的连线，对应到ASIC开发的后端流程。

通用的FPGA Synthesis工具有Synopsys的[Synplify](https://www.synopsys.com/implementation-and-signoff/fpga-based-design/synplify-pro.html)和Mentor的[Precision](https://eda.sw.siemens.com/en-US/ic/precision/rtl/)。Intel/Altera和Xilinx也有自己专用的Synthesis工具。Placement&Routing由于与具体的芯片关系紧密，各家FPGA厂商都是使用的自己的工具。

FPGA中还集成了很多其他硬件，用于方便开发人员实现自己的功能，比如高速收发器用于对接外部的高速串行数据I/O，用于实现10G/40G以太网，HDMI等接口；硬核CPU用于控制；硬核memory controller可以极大的加快内存的访问速度。

数字逻辑电路的Synthesis和P&R的速度实在太慢了，而且对多线程的支持也不完善，我们这个小项目一次完整的综合布局布线流程需要20-30分钟。

# 调试与仿真

RTL开发很难进行调试，常见的拍错方式是进行simulation（仿真）。Simulation就是给予待测试单元（UUT）输入激励，然后通过simulator模拟硬件工作，对激励给出输出信号，simulator可以记录module内部信号的变化，用waveform的形式进行展示。相比于软件的debugger，simulator能够记录所有的信号变化，类似于tracer。对于信号中出现的错误，分析waveform，并定位出root cause是数字逻辑开发人员的基本功。

目前主流的Simulator有Synopsys的VCS+Verdi，Cadence的NCVerilog/Xcelium和Mentor的ModelSim/QuestaSim。另外，Xilinx的Vivado中也有集成自家的Simulator。还有Aldec的ActiveHDL和Riviera Pro这种比较小众的Simulator。

Simulator是EDA软件中为数不多有着开源替代方案的领域。[Verilator](https://www.veripool.org/verilator/)和[Icarus Verilog](http://iverilog.icarus.com/)也是不错的选择，唯一的遗憾是Verilator对SystemVerilog和UVM的支持还不够好，从[issue](https://github.com/verilator/verilator/issues/2446)来看，官方也正在着手改进，希望能够尽快支持最新的SystemVerilog和UVM标准。

Mentor的ModelSim集成在主要的FPGA厂商的IDE中，对于比较小的设计，可以免费使用。但是，作为主流EDA厂家的Simulator，ModelSim实在算不上是好用，工具栏的位置每次切换布局都出问题，拖动窗口的时候反应极为迟缓，怎么都想不通为什么这么个破玩意居然可以买到好几万。

Simulation的速度也是非常慢，很多Simulator只能单线程运行，多线程运行需要额外加钱，而且加速比也不高，甚至需要手工分割任务。免费版的ModelSim按文档说法只有收费版本的1/4的速度，还有零有整的，怀疑里面每个操作都要额外多loop三次。

# Lint

由于无论是仿真还是综合都需要大量时间，Lint工具也成了必不可少的组成部分。很多应该在语言层面加以禁止的问题，只有通过Lint工具才能发现。Lint工具中比较好的有Synopsys的SpyGlass，Aldec的[Alint pro](https://www.aldec.com/en/products/functional_verification/alint-pro)。

# Verification

类似于软件中的TDD（Test Driven Development），RTL设计中Verification也是用来对RTL design做测试的重要步骤，目前主流使用的是UVM，号称是“方法学”，其实就是一个单元测试框架罢了，呵呵。当然，通过Verification也能获取coverage。

# 开发环境

不像软件开发环境的百花齐放，HDL的编辑器要么功能简陋，要么价格昂贵。VIM和vscode加上一些插件后，基本能够满足作为editor的要求。语法高亮的，代码导航，Intellisense等功能虽说也有，但是谈不上好用。[Sigasi](https://www.sigasi.com/)和[DVT](https://dvteclipse.com/)看起来功能不错。Aldec的[ActiveHDL](https://www.aldec.com/en/products/fpga_simulation/active-hdl)内置simulator，看起来也很好用。

# 基础库和IP加密

在软件开发中，一门语言或技术是否有着丰富的标准库，极大的影响着它的流行程度。在HDL中，特别是Verilog中，基本没有官方的标准库，除了基本的算术运算和位运算，没有太多的可以重用的库，延迟一拍这种常见的操作都需要自己实现，只能说目前的HDL语言的抽象程度太低了。

虽然没有基础库，但是各种IP还是不少的，但是IP的价格就不太有亲和力了。IP一般也是被加密过的，目前主流是IEEE-1735标准。之前各家Simulator或者Synthesisor也有自己实现的私有方案，有使用binary格式的，有使用置换密码的，还有代码混淆的。如果出现了问题，也只能再付钱购买额外的服务，或者额外的代码授权了。

后面有时间，我会单独写一篇post展开讲一下IP加密。

# 总结

本文主要是我作为一个软件开发工程师，针对学习并实践RTL开发的过程中所遇到的问题做的一些总结和吐槽，希望能够帮助到对硬件开发有兴趣的朋友。
