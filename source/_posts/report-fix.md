---
title: ventus LLVM 初步 调研 report-fix
---
Q:第一个细节是技术上要注意一下：ventus-llvm的llc调用命令要参考ventus-llvm的readme，加上`-mtriple=riscv32 -mcpu=ventus-gpgpu`；而官方llvm的llc调用命令则是加上`-mtriple=riscv32 -mattr=+v,+zvfh`。这样应该就能解决“承影的产生代码是x86MIR而不是RISC-V MIR”以及“官方llvm跑half.ll会报错”的问题，试一试。
A:加上-mtriple=riscv32 -mcpu=ventus-gpgpu确实产生了riscv的MIR 但是官方执行`/home/wjsun/LLVM/build/bin/llc -march=riscv32 -mattr=+v,+zvfh --debug-only=isel half.ll &> out.log`仍然报错
```
/home/wjsun/LLVM/build/bin/llc: error: /home/wjsun/LLVM/build/bin/llc: half.ll:369:25: error: floating point constant invalid for type
  %fneg = fmul half %b, 0xBFF3333340000000
                        ^

```
Q:
第二个细节是表达上要注意一下：要展示MIR的差异之处的话，可以直接把ventus-llvm和官方llvm在“===== Instruction selection ends:”之后的结果摆在一起，这样再得出“MIR加的指令的表示方式不同”的结论，这样会更加顺畅；而要展示发现有报错时，在贴报错信息的时候可以同步贴上执行命令，方便后续复现。
A：已修改()
Q:
1. 对于float.ll的测试，这里的差别确实是MIR加的指令的表示方式不同，不过更具体来说，是官方llvm会产生标量加指令FADD_S，而ventus-llvm却会出现向量加指令VFADD_VV。是的，同样的代码，官方会得到标量MIR，承影却会得到向量MIR，这是一个重点。
A:是的！完全是这样的。
Q:由于官方llvm对于float.ll和half.ll都只生成标量代码，如果把“-mattr=+v,+zvfh”换成“-mattr=+zfh”，可以看到最终结果跟“支持标量half”文档里官方llvm的结果应该是相似的。
执行/home/wjsun/LLVM/build/bin/llc -march=riscv32 -mattr=+zfh --debug-only=isel half.ll &> out.log

仍然报错
```
/home/wjsun/LLVM/build/bin/llc: error: /home/wjsun/LLVM/build/bin/llc: half.ll:369:25: error: floating point constant invalid for type
  %fneg = fmul half %b, 0xBFF3333340000000
```