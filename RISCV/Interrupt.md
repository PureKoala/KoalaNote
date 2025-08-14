# RISC-V 中断相关笔记

## Resource

- [详解RISCV中断](https://www.cnblogs.com/harrypotterjackson/p/17548837.html)
- [RISC-V Advanced Core Local Interruptor Specification](https://github.com/riscvarchive/riscv-aclint/blob/main/riscv-aclint.adoc#3-machine-level-software-interrupt-device-mswi)
- [RISC-V Platform-Level Interrupt Controller Specification](https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc)


## 1 CLINT / ACLINT

- CLINT (Core Local Interrupt, 核心本地中断器) 是最初的版本，其包含 `MSWI` 和 `MTIME` 相关寄存器
  - 默认情况下，所有 trap 都在机器模式下处理。机器模式软件可以通过在 `mideleg` ( 委托中断 ) 和 `medeleg` ( 委托异常 ) CSR 寄存器 中设置相应的位，有选择地将中断和异常委托给 S 模式。此时通过 S 模式寄存器 `sip`,`sie`,`scause`,`stvec` 来管理 S 中断。
- ACLINT 规范采用了一种更模块化的方法，为 IPI 和定时器功能定义了单独的内存映射设备，为 supervisor-level IPI 定义了一个专用的内存映射设备 `SSWI`
  - 向后兼容 CLINT

## 1.1 Overview

| Name | Privilege Level | Functionality |
|:----:|:---------------:|:-------------:|
| **MTIMER** | Machine       | Fixed-frequency counter and timer events |
| **MSWI**   | Machine       | Inter-processor (or software) interrupts |
| **SSWI**   | Supervisor    | Inter-processor (or software) interrupts |


```
┌────────────────────────────────────────────────────────────┐
│                    Socket 0 CLINT区域                       │
│                 Base: 0x2000000, Size: 0x10000             │
├────────────────────────────────────────────────────────────┤
│ MSWI (Machine Software Interrupt)                          │
│ Base: 0x2000000                                            │
│ Size: RISCV_ACLINT_SWI_SIZE (0x4000 = 16KB)                │
├────────────────────────────────────────────────────────────┤ 
│ MTIMER区域                                                  │
│ Base: 0x2004000 (0x2000000 + RISCV_ACLINT_SWI_SIZE)        │
│ Size: 0xC000 (0x10000 - 0x4000)                            │
│ ├─ MTIMECMP[0]: 0x2004000 + 0x4000 = 0x2008000             │
│ ├─ MTIMECMP[1]: 0x2004000 + 0x4008 = 0x2008008             │
│ ├─ MTIMECMP[n]: 0x2004000 + 0x4000 + n*8                   │
│ └─ MTIME:       0x2004000 + 0xBFF8 = 0x200FFF8             │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    Socket 0 SSWI区域                        │
│                 Base: 0x2F00000, Size: 0x4000              │
├────────────────────────────────────────────────────────────┤
│ SSWI (Supervisor Software Interrupt)                       │
│ 每个hart一个SSWI寄存器                                       │
└────────────────────────────────────────────────────────────┘
```


## 1.2 Machine-level Timer Device (MTIMER)

- **MTIME**（8B）：固定频率的唯一全局计时器
  - 全局也可以有很多，但是必须是在相同的时钟下，每个MTIME寄存器对应一组MTIMECMP寄存器；在此情况下如果进行了power management action，需要软件保证 re-synchronize
- **MTIMECMP**（8B）：每个核心（HART）的独立计数器

## 1.3 Machine-level Software Interrupt Device (MSWI)

- 每个 CPU 都包含一个连接到 MSWI 器件的 **IPI Register (MSIP)**
- 这些 IPI（Inter-Processor Interrupt） 功能是 machine-level 的
- 每个 MSWI 器件最多支持控制 4095 个 MSIP 寄存器（4B）
- MSIP reset 时所有 IPI 寄存器也会 reset
- **寄存器功能**：
  - Each MSIP register is a 32-bit wide WARL register where the upper 31 bits are wired to zero
  - The least significant bit is reflected in MSIP of the `mip` CSR
  - A machine-level software interrupt for a HART is pending or cleared by writing `1` or `0` respectively to the corresponding `MSIP` in the `mip` CSR


## 1.4 Supervisor-level Software Interrupt Device (SSWI)

- 每个 CPU 都包含一个连接到 SSWI 器件的 **IPI Register (SETSSIP)**
- 这些 IPI（Inter-Processor Interrupt） 功能是 supervisor-level 的
- 每个 SSWI 器件最多支持控制 4095 个 SETSSIP 寄存器（4B）
- **寄存器功能**：
  - Each SSIP register is a 32-bit wide WARL register where the upper 31 bits are wired to zero
  - Writing `0` to the least significant bit of a `SETSSIP` register has no effect whereas writing `1` to the least significant bit sends an edge-sensitive interrupt signal to the corresponding HART causing the HART to set `SSIP` in the `mip` CSR

- **Note**：
  - The RISC-V Privileged Architecture defines `SSIP` in `mip` and `sip` CSRs as a writeable bit so the M-mode or S-mode software can directly clear SSIP.

## 1.5 Address Mapping Example (ACLINT)

- 每个socket的地址计算：
  - Socket N的`CLINT`基地址: 0x2000000 + socket_id * 0x10000
  - Socket N的`SSWI`基地址: 0x2F00000 + socket_id * 0x4000
- 每个CPU的`MSWI`寄存器：
  - 0x2000000 + cpu_id * 4
- 每个CPU的`MTIMECMP`寄存器：
  - 0x2000000 + RISCV_ACLINT_SWI_SIZE + MTIMECMP_OFFSET + socket_id * 0x10000 + M * 8
- 每个CPU的`SSWI`寄存器
  - 0x2F00000 + cpu_id * 4


## 2 PLIC 逻辑与寄存器说明

简单的讲，PLIC 的逻辑分为 gateway 和 core 两大部分：
- **gateway** 控制着外设的中断信号能否进入 PLIC。
- **core** 是 PLIC 具体的处理逻辑。

大的逻辑上看，外设的中断信息可能被送到任意 hart 的外部中断输入管脚上，但是具体会送到哪个 hart 的哪个外部中断输入管脚上，就要看 PLIC 的配置。

PLIC 上的几个关键定义有：
- **中断 ID (Interrupt Identifier)**：区分不同外设的中断，不同外设的中断会固定接到不同的中断 ID 上，这个定义会在机器硬件描述信息中体现，比如描述在 DTS 里。
- **中断通知 (Interrupt Notification)**：PLIC 为它支持的每个中断设置一个 pending bit，用这个 pending bit 表示对应外设触发了相关的中断。PLIC spec 里叫这个是 Interrupt Notification。注意，每个外设的中断在 PLIC 里也只能 pending 一个，这个设计就有点简单了。
- **target context** 概念：简单看就是一个 core 上，M mode 外部中断是一个 target context，S mode 外部中断是一个 target context。理论上每个输入 PLIC 的中断都可以输出到任何一个 core 的任何一个 target context 上。PLIC 可以通过下面的 enable register 使能和关闭相关的输出路径，可见 enable register 是一个中断源和 target context 相乘之积大小的配置表。

PLIC 中的配置全部通过 MMIO 接口进行，所支持的中断的 pending 和 enable 也都全部在 MMIO 配置，这种设计真是简单粗暴啊。

下面把每个寄存器的定义展开，进一步说明 PLIC：

- **priority register**
  - 针对每个中断源的优先级配置，每个中断源都可以配置一个优先级。
- **pending bits register**
  - 针对每个中断源的配置。
- **enable register**
  - 如上提到的，每个中断源在每个 target context 下都有一个 enable bit 去配置。
- **threshold register**
  - 针对每个 target context 的配置，当中断源的 priority 数值大于 target context 的数值时，这个中断源的中断才能报给对应的 target context。
- **claim register**
  - 针对每个 target context 的配置，target context 从 claim 寄存器中读到中断 ID 号。
- **complete register**
  - 针对每个 target context 的配置，target context 通过这个寄存器告诉 PLIC 中断已经处理，PLIC 在这之后才会打开 gateway，让后面的中断进来。




