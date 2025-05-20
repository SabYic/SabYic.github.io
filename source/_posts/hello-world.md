---
title: ventus LLVM 初步 调研
---

# Pretask-J142-PLCT-20250510

## 乘影 LLVM 与官方 LLVM 的 RISC-V ZVFH 扩展支持对比及指令选择阶段分析
### 概述
主要对比乘影 LLVM（ventus-llvm）与官方 LLVM 在 ZVFH 扩展支持方面的实现差异，特别关注指令选择阶段。需要同时在 float 和 half 数据类型上做测试。

### 前置知识
学习前置知识了解了zvfh和zvfhmin是什么

### 1. 环境搭建与验证
- 克隆乘影编译器源码仓库 ventus-llvm（J142 岗位描述中提供），编译并成功构建 ventus-llvm。跑通 README 中给出的示例。
    成功编译产生llc，但是没有成功编译产生vecadd，原因出在gcc版本和clang的版本上。

### 2. 乘影 LLVM 向量 float 测试
这里需注意的，应该在build ventus-llvm的时候开启debug选项，不然编译出来release版本就没有办法看debug信息了！
#### -参考 Pretask-20250424 中“附录2：开发提示 - 标量 half 类型支持”的写法，使用乘影编译器的 `llc` 对 Pretask-20250424 的“调研提示”中提到的 `float.ll` 进行编译。
我使用命令`/home/sunwenjia/ventus/llvm-project/install/bin/llc --debug-only=isel float.ll &> ~/output2.log`
观察产生MIR的代码，发现是x86MIR。
#### 添加编译选项 `--debug-only=isel`，打印出指令选择阶段的 log 文件。
生成log如下
```
=== fadd_v
Initial selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


Optimized lowered selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


Type-legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


Legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


Optimized legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


===== Instruction selection begins: %bb.0 'entry'

ISEL: Starting selection on root node: t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1
ISEL: Starting pattern match
  Morphed node: t8: ch = PseudoRET # D:1 Register:f32 $v0, t7, t7:1
ISEL: Match complete!

ISEL: Starting selection on root node: t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5

ISEL: Starting selection on root node: t5: f32 = fadd # D:1 t2, t4
ISEL: Starting pattern match
  Initial Opcode index to 28599
  Match failed at index 28602
  Continuing at 28637
  Match failed at index 28640
  Continuing at 28661
  Match failed at index 28663
  Continuing at 28685
  Match failed at index 28691
  Continuing at 28725
  TypeSwitch[f32] from 28728 to 28731
  Morphed node: t5: f32 = VFADD_VV nofpexcept # D:1 t2, t4
ISEL: Match complete!

ISEL: Starting selection on root node: t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1

ISEL: Starting selection on root node: t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0

ISEL: Starting selection on root node: t6: f32 = Register $v0

ISEL: Starting selection on root node: t3: f32 = Register %1

ISEL: Starting selection on root node: t1: f32 = Register %0

ISEL: Starting selection on root node: t0: ch,glue = EntryToken

===== Instruction selection ends:
Selected selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = VFADD_VV nofpexcept # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = PseudoRET # D:1 Register:f32 $v0, t7, t7:1


Total amount of phi nodes to update: 0
*** MachineFunction at end of ISel ***
# Machine code for function fadd_v: IsSSA, TracksLiveness
Function Live Ins: $v0 in %0, $v1 in %1

bb.0.entry:
  liveins: $v0, $v1
  %1:vgpr = COPY $v1
  %0:vgpr = COPY $v0
  %2:vgpr = nofpexcept VFADD_VV %0:vgpr, %1:vgpr, implicit $frm
  $v0 = COPY %2:vgpr
  PseudoRET implicit $v0

# End machine code for function fadd_v.

.......



```
产生了riscv代码
阅读并理解 log 文件中输出的内容，了解指令是如何从 SelectionDAG 节点映射到 MIR 机器指令的。相关资料博客[Legalizations in LLVM Backend](https://myhsu.xyz/llvm-codegen-legalization/)
那么它总共有这几个流程
```
Initial selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1


```
这是开始的dag
```
Optimized lowered selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1

对这个DAG图使用pass优化,比如死代码消除等
```
```

Type-legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1
这里是类型合法化，让当前代码映射到指定架构目标上
```
```
Legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1
  这里应该还进行了操作合法化
```
```
Optimized legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = fadd # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1
再进行一遍优化？
```

```
===== Instruction selection begins: %bb.0 'entry'

ISEL: Starting selection on root node: t8: ch = RISCVISD::RET_FLAG # D:1 t7, Register:f32 $v0, t7:1
ISEL: Starting pattern match
  Morphed node: t8: ch = PseudoRET # D:1 Register:f32 $v0, t7, t7:1
ISEL: Match complete!

ISEL: Starting selection on root node: t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5

ISEL: Starting selection on root node: t5: f32 = fadd # D:1 t2, t4
ISEL: Starting pattern match
  Initial Opcode index to 28599
  Match failed at index 28602
  Continuing at 28637
  Match failed at index 28640
  Continuing at 28661
  Match failed at index 28663
  Continuing at 28685
  Match failed at index 28691
  Continuing at 28725
  TypeSwitch[f32] from 28728 to 28731
  Morphed node: t5: f32 = VFADD_VV nofpexcept # D:1 t2, t4
ISEL: Match complete!

ISEL: Starting selection on root node: t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1

ISEL: Starting selection on root node: t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0

ISEL: Starting selection on root node: t6: f32 = Register $v0

ISEL: Starting selection on root node: t3: f32 = Register %1

ISEL: Starting selection on root node: t1: f32 = Register %0

ISEL: Starting selection on root node: t0: ch,glue = EntryToken

===== Instruction selection ends:

这里是匹配MIR代码
```
```
===== Instruction selection ends:
Selected selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg # D:1 t0, Register:f32 %0
      t4: f32,ch = CopyFromReg # D:1 t0, Register:f32 %1
    t5: f32 = VFADD_VV nofpexcept # D:1 t2, t4
  t7: ch,glue = CopyToReg # D:1 t0, Register:f32 $v0, t5
  t8: ch = PseudoRET # D:1 Register:f32 $v0, t7, t7:1

这里可以看到add指令已经被替换成MIR风格的了
```
```
Total amount of phi nodes to update: 0
*** MachineFunction at end of ISel ***
# Machine code for function fadd_v: IsSSA, TracksLiveness
Function Live Ins: $v0 in %0, $v1 in %1

bb.0.entry:
  liveins: $v0, $v1
  %1:vgpr = COPY $v1
  %0:vgpr = COPY $v0
  %2:vgpr = nofpexcept VFADD_VV %0:vgpr, %1:vgpr, implicit $frm
  $v0 = COPY %2:vgpr
  PseudoRET implicit $v0

# End machine code for function fadd_v.
MIR代码如上
```

### 3. 官方 LLVM 向量 float 测试

```
FastISel is disabled



=== fadd_v

Initial selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = fadd t2, t4
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1



Optimized lowered selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = fadd t2, t4
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1



Type-legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = fadd t2, t4
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1



Legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = fadd t2, t4
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1



Optimized legalized selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 9 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = fadd t2, t4
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1


===== Instruction selection begins: %bb.0 'entry'

ISEL: Starting selection on root node: t8: ch = RISCVISD::RET_GLUE t7, Register:f32 $f10_f, t7:1
ISEL: Starting pattern match
  Morphed node: t8: ch = PseudoRET Register:f32 $f10_f, t7, t7:1
ISEL: Match complete!

ISEL: Starting selection on root node: t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5

ISEL: Starting selection on root node: t5: f32 = fadd t2, t4
ISEL: Starting pattern match
  Initial Opcode index to 1411309
  TypeSwitch[f32] from 1411314 to 1411317
  Morphed node: t5: f32 = FADD_S nofpexcept t2, t4, TargetConstant:i64<7>
ISEL: Match complete!

ISEL: Starting selection on root node: t4: f32,ch = CopyFromReg t0, Register:f32 %1

ISEL: Starting selection on root node: t2: f32,ch = CopyFromReg t0, Register:f32 %0

ISEL: Starting selection on root node: t6: f32 = Register $f10_f

ISEL: Starting selection on root node: t3: f32 = Register %1

ISEL: Starting selection on root node: t1: f32 = Register %0

ISEL: Starting selection on root node: t0: ch,glue = EntryToken

===== Instruction selection ends:

Selected selection DAG: %bb.0 'fadd_v:entry'
SelectionDAG has 10 nodes:
  t0: ch,glue = EntryToken
      t2: f32,ch = CopyFromReg t0, Register:f32 %0
      t4: f32,ch = CopyFromReg t0, Register:f32 %1
    t5: f32 = FADD_S nofpexcept t2, t4, TargetConstant:i64<7>
  t7: ch,glue = CopyToReg t0, Register:f32 $f10_f, t5
  t8: ch = PseudoRET Register:f32 $f10_f, t7, t7:1


Total amount of phi nodes to update: 0
*** MachineFunction at end of ISel ***
# Machine code for function fadd_v: IsSSA, TracksLiveness
Function Live Ins: $f10_f in %0, $f11_f in %1

bb.0.entry:
  liveins: $f10_f, $f11_f
  %1:fpr32 = COPY $f11_f
  %0:fpr32 = COPY $f10_f
  %2:fpr32 = nofpexcept FADD_S %0:fpr32, %1:fpr32, 7, implicit $frm
  $f10_f = COPY %2:fpr32
  PseudoRET implicit $f10_f

# End machine code for function fadd_v.
```
发现MIR是不同的，主要的不同是寄存器符号的不同和MIR加的指令的表示方式不同

ventus的MIR如下
```
bb.0.entry:
  liveins: $v0
  %0:vgpr = COPY $v0
  %1:gpr = LUI target-flags(riscv-hi) @global_val
  %2:gpr = LW killed %1:gpr, target-flags(riscv-lo) @global_val :: (dereferenceable load (s32) from @global_val)
  %3:gprf32 = COPY %2:gpr
  %5:vgpr = COPY %3:gprf32
  %4:vgpr = nofpexcept VFADD_VV %0:vgpr, killed %5:vgpr, implicit $frm
  $v0 = COPY %4:vgpr
  PseudoRET implicit $v0
```
官方LLVM是这样
```
bb.0.entry:
  liveins: $f10_f, $f11_f
  %1:fpr32 = COPY $f11_f
  %0:fpr32 = COPY $f10_f
  %2:fpr32 = nofpexcept FADD_S %0:fpr32, %1:fpr32, 7, implicit $frm
  $f10_f = COPY %2:fpr32
  PseudoRET implicit $f10_f

```
确实，经过对比发现同样的代码，官方会得到标量MIR，承影却会得到向量MIR，可以清晰的通过上面的对比发现，官方是标量加，ventus是向量加。
### 4. 官方 LLVM 向量 half 测试
#### - 将 `float.ll` 中的所有 `float` 类型替换为 `half`，得到新的 `half.ll`。
#### - 同样使用官方 LLVM 运行 `half.ll` 得到MIR指令和 log 文件，观察它们跟运行 `float.ll` 的差异。
#### - 最后，尝试也用乘影 LLVM 运行 `half.ll` ，看是否会有报错，如有则记录报错情况并尝试猜测原因，如没有则记录MIR结果和 log 文件并评估是否符合预期。
```
 NOTE: Assertions have been autogenerated by utils/update_llc_test_checks.py
; RUN: llc -mtriple=riscv32 -mcpu=ventus-gpgpu -verify-machineinstrs < %s \
; RUN:   | FileCheck -check-prefix=VENTUS %s

define half @fadd_v(half noundef %a, half noundef %b) {
; VENTUS-LABEL: fadd_v:
; VENTUS:       # %bb.0: # %entry
; VENTUS-NEXT:    vfadd.vv v0, v0, v1
; VENTUS-NEXT:    ret
entry:
  %add = fadd half %a, %b
  ret half %add
}

define half @fadd_f(half noundef %a) {
; VENTUS-LABEL: fadd_f:
; VENTUS:       # %bb.0: # %entry
; VENTUS-NEXT:    lui t0, %hi(global_val)
; VENTUS-NEXT:    lw t0, %lo(global_val)(t0)
; VENTUS-NEXT:    vmv.v.x v1, t0
; VENTUS-NEXT:    vfadd.vv v0, v0, v1
; VENTUS-NEXT:    ret
entry:
  %val = load half, ptr @global_val, align 4
  %add = fadd half %a, %val
  ret half %add
}

define half @fsub_v(half noundef %a, half noundef %b) {
; VENTUS-LABEL: fsub_v:
; VENTUS:       # %bb.0: # %entry
; VENTUS-NEXT:    vfsub.vv v0, v0, v1
; VENTUS-NEXT:    ret
entry:
  %add = fsub half %a, %b
  ret half %add
}
```
先用官方llvm编译,但是修改后仍然报错
但是官方执行`/home/wjsun/LLVM/build/bin/llc -march=riscv32 -mattr=+v,+zvfh --debug-only=isel half.ll &> out.log`仍然报错
```
/home/wjsun/LLVM/build/bin/llc: error: /home/wjsun/LLVM/build/bin/llc: half.ll:369:25: error: floating point constant invalid for type
  %fneg = fmul half %b, 0xBFF3333340000000
```
都是因为不符合half的存储长度造成的。那么用ventusllvm会好吗？
执行命令`/home/sunwenjia/ventus/llvm-project/install/bin/llc  -mtriple=riscv32 -mcpu=ventus-gpgpu --debug-only=isel float.ll &> out3.log`
也没有改变
```
/home/sunwenjia/ventus/llvm-project/install/bin/llc: error: /home/sunwenjia/ventus/llvm-project/install/bin/llc: float.ll:207:14: error: floating point constant invalid for type
  store half 0x3FF3333340000000, ptr addrspace(5) %b, align 4
             ^

             ^
```
