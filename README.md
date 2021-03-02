# RISC V 指令仿真器 Spike 内部原理分析笔记

## About

    本仓库中的文档为本人对 RISC-V ISA SIM（或 Spike），这一 RISC-V 基金会官方指定的指令集模拟器的代码笔记。
    由于 Spike 的文档和文献大多过时且分散，因此这些文档均为我直接阅读其代码进行的代码解析和程序原理分析，不一定准确无误，如有纰漏还请指正。
    
## Content

    + cpu_virtulization.md  Spike 中CPU虚拟化的说明，介绍了Spike中描述虚拟CPU的类，以及虚拟CPU对象的初始化过程，虚拟CPU如何处理CSR寄存器的读写和虚拟CPU如何处理中断。对于具体的指令集模拟（或者说 RISC-V 指令的解释执行）会在另外的文章中进行更详细的说明。
    
    + bus_and_device.md  Spike 中虚拟总线和设备的说明，介绍了Spike中总线和设备的概念，通过具体分析 Spike 中已经实现的总线和少许虚拟设备对象，尝试说明总线的作用，设备的注册方法，目标程序如读写设备。

## Plan
  
    + Spike 模拟器的运行流程
    + 指令解释执行原理和优化
    + 虚拟MMU和虚拟内存的实现
    + SPIKE主机&目标机接口（HTIF）的介绍
