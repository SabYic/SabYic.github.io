---
title: 读target目录
---
## target 目录内容大致解析
### ASMparser 
RISCVAsmParser.cpp - Parse RISCV assembly to MCInst instructions
MCInst 是machine code instruction的意思，它是在汇编和机器码之间的等级。
### 问题：
为什么要有这个MCInst，为什么不直接汇编到机器码就好了？

A: 整个后端流水线用到了四种不同层次的指令表示：内存中的LLVM IR，SelectionDAG节点，MachineInstr，和MCInst。那么machineInstr和MCinst之间干了什么呢

### Disassembler
这就是反汇编器，好理解

### GIsel
Global instruction select 全局指令选择，就是ISEL的一部分，但是我感觉代码不太多。是都写在其他地方了吗？

### MCTargetDesc
MC to 机器码 关于生成elf文件。这个明天也要问一下。

### TargetInfo 
指定三元组信息

### RISCV.H
高层次接口。
定义了各种pass和一些栈类型

### RISCV.td
Subtarget Features && Instruction Predicates
某个具体的目标 CPU 子类型（subtarget）所支持的可选扩展或特性。
Instruction Predicates 是基于 Subtarget Features 的条件判断，用来决定：

- 某条指令是否可以在当前目标上合法生成；

- 是否在 .td 中启用某个指令定义；

- 或用于指令选择（Instruction Selection）阶段条件匹配。
### 插播一条：
td会生成inc然后被include进cpp里面

### RISCVAsmPrinter.cpp         
汇编转移二进制？
### RISCVInstrInfoV.td           
这个要详细看一下：分部分
Operand and SDNode transformation definitions.
1. LLVM 生成 SelectionDAG：
   LLVM 会将 LLVM IR 转换成 SelectionDAG，里面的节点是各种 SDNode 表示的操作（如 add, load, zext 等）。

2. 通过 Pat 匹配规则识别 SDNode 组合：

   ```tablegen
   def : Pat<(add GPR:$rs1, GPR:$rs2), (ADD $rs1, $rs2)>;
这段表示：当看到 DAG 中的 add 节点，其操作数来自寄存器时，用目标机器指令 ADD 替换。

使用 Operand 规则解析寄存器、立即数、内存操作数等：

tablegen
```
def simm5 : Operand<XLenVT>, ImmLeaf<XLenVT, [{ return isInt<5>(Imm); }]>;
```

3. 它告诉编译器，某个操作数是一个 5 位有符号立即数，编码时要验证这个约束。

4. 生成机器指令（MachineInstr）：
DAG 经过 ISelDAGToDAG 的匹配和替换后，会转为 MachineInstr，并使用 Operand 信息生成二进制编码。

Scheduling definitions.
这一部分是干嘛
Instruction class templates
比如：
// strided segment load vd, (rs1), rs2, vm
成段的load，store。
以及向量和标量的结合计算啥的
Combination of instruction classes.
这里是一些指令子集啥的，比如VF，F就是两个不同的子集，同时根据不同的predict实例化不同子集
tablegen
```
let Predicates = [HasVInstructions] in {
def VLM_V : VUnitStrideLoadMask<"vlm.v">,
             Sched<[WriteVLDM, ReadVLDX]>;
def VSM_V : VUnitStrideStoreMask<"vsm.v">,
             Sched<[WriteVSTM, ReadVSTM, ReadVSTX]>;
def : InstAlias<"vle1.v $vd, (${rs1})",
                (VLM_V VR:$vd, GPR:$rs1), 0>;
def : InstAlias<"vse1.v $vs3, (${rs1})",
                (VSM_V VR:$vs3, GPR:$rs1), 0>;

```
RISCVInstrInfoVPseudos.td
### RISCVRegisterInfo.td    
1           
### VentusInstrFormatsV.td
### RISCVCallingConv.td 
中断恢复
### RISCVInstrInfoVVLPatterns.td  
### RISCVSchedRocket.td                
### VentusInstrInfoA.td
原子化拓展(关于ventus)
下面开始关注部分文件，关注一些更有借鉴意义的。
### RISCVCodeGenPrepare.cpp

### RISCVInstrInfoXVentana.td   
### RISCVSchedSiFive7.td               
### VentusInstrInfoC.td
### RISCVExpandAtomicPseudoInsts.cpp
### RISCVInstrInfoZb.td           
### RISCVSchedule.td                 
### VentusInstrInfoF.td
### RISCVExpandPseudoInsts.cpp     
### RISCVInstrInfoZfh.td          
### RISCVScheduleV.td                  
### VentusInstrInfoM.td
RISCVFrameLowering.cpp            RISCVInstrInfoZicbo.td        RISCVScheduleZb.td                 VentusInstrInfo.td
RISCVFrameLowering.h              RISCVInstrInfoZk.td           RISCVSearchableTables.td           VentusInstrInfoVPseudos.td
RISCV.h                           RISCVISelDAGToDAG.cpp         RISCVSExtWRemoval.cpp              VentusInstrInfoVSDPatterns.td
RISCVInstrFormatsC.td             RISCVISelDAGToDAG.h           RISCVSubtarget.cpp                 VentusInstrInfoV.td
RISCVInstrFormats.td              RISCVISelLowering.cpp         RISCVSubtarget.h                   VentusInstrInfoVVLPatterns.td
RISCVInstrFormatsV.td             RISCVISelLowering.h           RISCVSystemOperands.td             VentusLegalizeLoad.cpp
RISCVInstrInfoA.td                RISCVMachineFunctionInfo.cpp  RISCVTargetMachine.cpp             VentusPrintfRuntimeBinding.cpp
RISCVInstrInfo.cpp                RISCVMachineFunctionInfo.h    RISCVTargetMachine.h               VentusProgramInfo.h
RISCVInstrInfoC.td                RISCVMacroFusion.cpp          RISCVTargetObjectFile.cpp          VentusRegextInsertion.cpp
RISCVInstrInfoD.td                RISCVMacroFusion.h            RISCVTargetObjectFile.h            VentusRegisterInfo.td
RISCVInstrInfoF.td 