# QEMU 相关内容

## 目录
1. 概述
2. RISC-V 特权级与 SBI/ECALL
   2.1 特权级简介
   2.2 SBI 调用流程 (Console_putchar)
3. QEMU UART 模拟器
   3.1 UART 寄存器一览
   3.2 FIFO 与中断使能
   3.3 虚拟机迁移状态保存
4. RISC-V 中断模型 (CLINT / PLIC)
   4.1 CLINT/ACLINT
   4.2 SSWI
   4.3 PLIC 结构与 MMIO
   4.4 PLIC 状态保存
5. 设备树与中断驱动
   5.1 设备树修改示例
   5.2 内核配置 (8250/16550)
   5.3 驱动验证 (`/proc/interrupts`, `dmesg`)
6. 总结

## 1 概述
> 本文档汇总 QEMU 中 RISC-V UART 与中断模拟的原理，涵盖特权级、SBI 调用、UART 设备、CLINT/PLIC 中断模型，以及通过设备树配置中断驱动的方法。

## 2 RISC-V 特权级与 SBI/ECALL

### 2.1 特权级简介
在典型的 RISC-V 启动流程中，存在以下特权级：
- **M-Mode (Machine Mode):** 最高特权级，由固件如 OpenSBI 运行，直接控制硬件。
- **S-Mode (Supervisor Mode):** 操作系统内核运行模式，如 Linux。
- **U-Mode (User Mode):** 用户应用程序运行模式。

特权级切换表：

| 源模式 | 目标模式 | 触发指令 | 异常处理寄存器 | 返回指令 |
| :---: | :---: | :---: | :---: | :---: |
| U-mode | S-mode | ecall | scause, sepc | sret |
| S-mode | M-mode | ecall（未处理异常） | mcause, mepc | mret |
| M-mode | S-mode | mret（需配置 mepc, mstatus） | - | - |

### 2.2 SBI 调用流程 (Console_putchar)
当 Linux 内核需要打印字符时，会通过 SBI 扩展调用 OpenSBI 的 `sbi_console_putchar`：

**详细调用流程：**

1. **用户程序 (U-Mode):** C 程序调用 `printf("hello\n");`。
2. **C 库 & 系统调用 (Syscall):** `printf` 最终调用内核的 `write` 系统调用，写入标准输出。
3. **Linux 内核 (S-Mode):**
   * 内核控制台子系统接收写请求。
   * 默认配置下使用 SBI，准备 `SBI_CONSOLE_PUTCHAR` 调用。
4. **ECALL 指令:**
   * 内核执行 `ecall`，从 S-Mode 陷入 M-Mode。
   * 按 SBI 规范，在寄存器 a7 放置函数 ID，a0 放参数（字符）。
5. **OpenSBI (M-Mode):**
   * 陷阱处理程序接管控制权。
   * 检查寄存器，调用 `sbi_console_putchar`。
   * 通过 MMIO **轮询** UART 状态寄存器，写入数据寄存器。
6. **QEMU 模拟的 UART:**
   * QEMU virt 平台模拟 `ns16550a` 兼容 UART。
   * OpenSBI 写入模拟寄存器时，QEMU 捕捉并打印到宿主机终端。
7. **返回流程:** OpenSBI 执行 `mret`，返回 Linux 内核 S-Mode。

流程示意图：
`用户程序 -> C 库 -> 内核 Syscall -> 内核 SBI Console 驱动 -> ECALL -> OpenSBI (M-Mode) -> OpenSBI UART 驱动 (轮询) -> QEMU 模拟的 UART 硬件`

## 3 QEMU UART 模拟器

### 3.1 UART 寄存器一览

QEMU 的 UART 模拟器对应文件：
- `qemu/hw/char/serial.c`
- `qemu/include/hw/char/serial.h`

主要寄存器包含：

- `divider` (uint16_t):
  - **波特率除数锁存器**。这个 16 位的值用于设置串口通信的波特率。波特率的计算公式通常是 `input_clock / (16 * divider)`。要访问这个寄存器，必须先将线路控制寄存器（`LCR`）中的 `DLAB` (Divisor Latch Access Bit) 位置为 1。
- `rbr` (uint8_t):
  - **接收缓冲寄存器 (Receive Buffer Register)**。当串口接收到数据时，数据被存放在这里。CPU 通过读取这个寄存器来获取接收到的字符。读取 `RBR` 会从接收 FIFO (First-In, First-Out) 队列中取出一个字节。
- `thr` (uint8_t):
  - **发送保持寄存器 (Transmit Holding Register)**。CPU 想要发送数据时，会将数据写入这个寄存器。写入 `THR` 的数据会被放入发送 FIFO 队列，等待被发送。
- `tsr` (uint8_t):
  - **发送移位寄存器 (Transmit Shift Register)**。这是一个内部寄存器，不直接对 CPU 可见。它保存当前正在通过串行线按位发送的字符。当 `tsr` 发送完一个字符后，会从 `thr` (或发送 FIFO) 中加载下一个字符。
- `ier` (uint8_t):
  - **中断使能寄存器 (Interrupt Enable Register)**。这个寄存器用来控制 UART 在特定事件发生时是否产生中断。例如，可以使能“接收到数据”、“发送保持寄存器为空”等中断。
- `iir` (uint8_t):
  - **中断识别寄存器 (Interrupt Identification Register)**。这是一个只读寄存器，用于识别当前挂起的中断源。当中断发生时，CPU 可以读取 `IIR` 来确定中断的原因。
- `lcr` (uint8_t):
  - **线路控制寄存器 (Line Control Register)**。用于配置串行通信的参数，如数据位长度（5, 6, 7, or 8 bits）、停止位数（1, 1.5, or 2 bits）和奇偶校验方式（无、奇、偶等）。它还包含 `DLAB` 位，用于控制对波特率除数锁存器的访问。
- `mcr` (uint8_t):
  - **调制解调器控制寄存器 (Modem Control Register)**。用于控制与调制解调器（Modem）的接口信号，如 `DTR` (Data Terminal Ready) 和 `RTS` (Request to Send)。它也包含一个用于测试的环回模式开关。
- `lsr` (uint8_t):
  - **线路状态寄存器 (Line Status Register)**。这是一个只读寄存器，提供了数据传输的状态信息。例如，它会指示接收缓冲器中是否有数据（Data Ready）、发送保持寄存器是否为空（Transmitter Holding Register Empty），以及是否发生了溢出、奇偶校验或帧错误。
- `msr` (uint8_t):
  - **调制解调器状态寄存器 (Modem Status Register)**。这是一个只读寄存器，反映了调制解调器接口信号的当前状态，如 `CTS` (Clear to Send)`、DSR` (Data Set Ready) 等。
- `scr` (uint8_t):
  - **暂存寄存器 (Scratch Register)**。一个通用的 8 位读/写寄存器，可以被软件用于任何临时存储目的，硬件本身不使用它。
- `fcr` (uint8_t):
  - **FIFO 控制寄存器 (FIFO Control Register)**。用于启用和管理 16550A UART 的 FIFO 缓冲。可以设置接收 FIFO 的触发级别（即接收多少字节后产生中断）、清除发送和接收 FIFO 等。
- `fcr_vmstate` (uint8_t):
  - **用于虚拟机状态迁移的 `FCR`**。直接写入 FCR 寄存器会产生副作用（例如，清除 FIFO）。为了在 QEMU 进行虚拟机迁移（save/load state）时能正确保存和恢复 FCR 的状态而不触发这些副作用，QEMU 使用这个 `fcr_vmstate` 字段来临时存储 FCR 的值。在加载状态后， post_load 函数会负责将这个值安全地写入真实的 FCR。

### 3.2 FIFO 与中断使能

16550A UART 支持 FIFO 缓冲和多种中断模式：
- 接收数据可用中断
- 发送保持寄存器空中断  
- 接收线路状态中断
- 调制解调器状态中断

### 3.3 虚拟机迁移状态保存

QEMU 在虚拟机迁移时需保存 UART 的完整状态，包括所有寄存器值、FIFO 内容和中断状态，确保迁移后 UART 功能正常恢复。

## 4 RISC-V 中断模型 (CLINT / PLIC)

### 4.1 CLINT/ACLINT

- [riscvarchive/riscv-aclint](https://github.com/riscvarchive/riscv-aclint/blob/main/riscv-aclint.adoc#3-machine-level-software-interrupt-device-mswi)

CLINT (Core Local Interrupt) 包含软件中断和定时器功能：
- **MSWI (Machine Software Interrupt)**: 处理器间中断
- **MTIMER**: 定时器中断
- **SSWI (Supervisor Software Interrupt)**: S 模式软件中断

ACLINT 是 CLINT 的模块化改进版，向后兼容 CLINT。

地址映射：
```
Socket 0 CLINT 区域: Base 0x2000000, Size 0x10000
├─ MSWI: 0x2000000, Size 0x4000  
├─ MTIMER: 0x2004000, Size 0xC000
│  ├─ MTIMECMP[n]: 0x2008000 + n*8
│  └─ MTIME: 0x200FFF8
└─ SSWI: 0x2F00000, Size 0x4000
```

#### MSWI 功能
- 每个 CPU 包含一个 IPI Register (MSIP)
- 32 位 WARL 寄存器，高 31 位为零
- 最低位反映在 `mip` CSR 的 MSIP 位
- 写 1 或 0 分别挂起或清除机器级软件中断

#### MTIMER 功能
- **MTIME**: 固定频率的全局计时器（8 字节）
- **MTIMECMP**: 每个核心的独立计数器（8 字节）

### 4.2 SSWI

SSWI 设备为每个 CPU 提供 S 模式软件中断寄存器 (SETSSIP)：
- 写 1 到 SETSSIP 最低位发送边沿敏感中断信号
- 写 0 无效果
- **Note**：
  - The RISC-V Privileged Architecture defines `SSIP` in `mip` and `sip` CSRs as a writeable bit so the M-mode or S-mode software can directly clear SSIP.

#### ACLINT 地址映射示例
- 每个socket的地址计算：
  - Socket N的`CLINT`基地址: 0x2000000 + socket_id * 0x10000
  - Socket N的`SSWI`基地址: 0x2F00000 + socket_id * 0x4000
- 每个CPU的`MSWI`寄存器：
  - 0x2000000 + cpu_id * 4
- 每个CPU的`MTIMECMP`寄存器：
  - 0x2000000 + RISCV_ACLINT_SWI_SIZE + MTIMECMP_OFFSET + socket_id * 0x10000 + M * 8
- 每个CPU的`SSWI`寄存器
  - 0x2F00000 + cpu_id * 4

### 4.3 PLIC 结构与 MMIO

- [详解RISC v中断 - LightningStar - 博客园](https://www.cnblogs.com/harrypotterjackson/p/17548837.html)
- [https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc](https://github.com/riscv/riscv-plic-spec/blob/master/riscv-plic.adoc)
- [riscv plic基本逻辑分析](https://wangzhou.github.io/riscv-plic%E5%9F%BA%E6%9C%AC%E9%80%BB%E8%BE%91%E5%88%86%E6%9E%90/)


PLIC (Platform-Level Interrupt Controller) 管理外部设备中断：
- 支持最多 1023 个中断源
- 支持多 hart、多上下文 (M/S 模式)
- 所有配置通过 MMIO 寄存器管理

核心概念：
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

PLIC 内存映射：

| 寄存器区域 | 偏移量 | 描述 |
|:---|:---|:---|
| **中断源优先级** | `0x000004 - 0x000FFC` | 每个中断源的 32 位优先级寄存器（源 1-1023） |
| **中断挂起位** | `0x001000 - 0x00107C` | 只读，反映中断源等待处理状态 |
| **中断使能位** | `0x002000 - 0x1F1FFC` | 每个 context 的中断使能位 |
| **上下文配置** | `0x200000 - 0x3FFFFFC` | 每个 context 的配置区域 |

关键地址计算：
- Enable 寄存器：`base + 0x002000 + context_id * 0x80`
- Context 寄存器：`base + 0x200000 + context_id * 0x1000`

### 4.4 PLIC 状态保存

QEMU PLIC 状态结构体包含：
```c
struct SiFivePLICState {
    /*< private >*/
    SysBusDevice parent_obj; // QEMU 设备基类

    /*< public >*/
    MemoryRegion mmio; // MMIO 区域
    uint32_t num_addrs; // 上下文数量（context/hart）
    uint32_t num_harts; // 处理器核心数量
    uint32_t bitfield_words; // enable/pending bitfield 的字数
    uint32_t num_enables; // enable bit 总数
    PLICAddr *addr_config; // 地址配置
    uint32_t *source_priority; // 每个中断源的优先级
    uint32_t *target_priority; // 每个 context 的阈值
    uint32_t *pending; // 每个中断源的 pending 状态
    uint32_t *claimed; // 每个 context 的 claim 状态
    uint32_t *enable; // 每个 context 的 enable bit

    /* config */
    char *hart_config; // hart 配置字符串
    uint32_t hartid_base; // hartid 起始值
    uint32_t num_sources; // 中断源数量
    uint32_t num_priorities; // 优先级等级数量
    uint32_t priority_base; // 优先级寄存器基地址
    uint32_t pending_base; // pending 寄存器基地址
    uint32_t enable_base; // enable 寄存器基地址
    uint32_t enable_stride; // enable 寄存器步长
    uint32_t context_base; // context 寄存器基地址
    uint32_t context_stride; // context 寄存器步长
    uint32_t aperture_size; // MMIO 区域大小

    qemu_irq *m_external_irqs; // M 模式外部中断线
    qemu_irq *s_external_irqs; // S 模式外部中断线
};
```

状态保存时只保存配置信息（优先级、阈值、使能），不保存运行时状态（pending/claimed），确保迁移时硬件状态一致。

**状态保存逻辑示例:**
```c
SiFivePLICState *plic = SIFIVE_PLIC(obj);

// 获取设备配置
uint32_t num_sources = CPT_RESTORE_IRQS; // 只保存有意义的中断源
uint32_t num_contexts = plic->num_addrs; // context 数量
uint32_t bitfield_words = (CPT_IRQCHIP_NUM_SOURCES + 31) / 32; // enable bitfield 字数

// 保存每个中断源的优先级
for (uint32_t i = 1; i <= num_sources; i++) {
    uint32_t priority = plic->source_priority[i];
    // ... add to restorer ...
}

// 保存每个 context 的 enable bits
for (uint32_t context = 0; context < num_contexts; context++) {
    for (uint32_t i = 0; i < bitfield_words; i++) {
        uint32_t enable = plic->enable[context * bitfield_words + i];
        // ... add to restorer ...
    }
}

// 保存每个 context 的优先级阈值
for (uint32_t context = 0; context < num_contexts; context++) {
    uint32_t threshold = plic->target_priority[context];
    // ... add to restorer ...
}
```

**关键参数说明:**
- `CPT_RESTORE_IRQS`: 定义了需要保存状态的关键中断源数量
- `CPT_IRQCHIP_NUM_SOURCES`: 平台总中断源数
- `bitfield_words` 必须用平台总中断源数计算，否则 enable bitfield 保存不完整。

## 5 设备树与中断驱动

### 5.1 设备树修改示例

**使用 OpenSBI (默认):**
```dts
/ {
    chosen {
        stdout-path = "serial0:115200n8";  // 通过 SBI 接口
    };
    soc {
        serial@10000000 {
            compatible = "ns16550a";
            // status 可能为 "disabled" 或被忽略
        };
    };
};
```

**切换到原生驱动:**
```dts
/ {
    chosen {
        stdout-path = &uart0;  // 直接指向 UART 设备节点
    };
    soc {
        uart0: serial@10000000 {
            compatible = "ns16550a";
            reg = <0x0 0x10000000 0x0 0x100>;
            interrupt-parent = <&plic0>;
            interrupts = <10>;
            clock-frequency = <3686400>;
            status = "okay";  // 明确启用
        };
    };
    plic0: interrupt-controller@c000000 {
        // PLIC 配置
    };
};
```

**QEMU 启动命令:**
```bash
qemu-system-riscv64 \
   -machine virt \
   -m 2G \
   -nographic \
   -kernel path/to/Image \
   -bios path/to/opensbi/fw_jump.bin \
   -append "root=/dev/vda rw console=ttyS0" \
   -dtb path/to/your-modified-virt.dtb
```

### 5.2 内核配置 (8250/16550)

确保内核编译时包含必要的驱动支持：

```
Device Drivers --->
    Character devices --->
        Serial drivers --->
            <*> 8250/16550 and compatible serial support
            [*]   Console on 8250/16550 and compatible serial port
```

配置选项：
- `CONFIG_SERIAL_8250=y`
- `CONFIG_SERIAL_8250_CONSOLE=y`

**对比表:**

| 特性 | OpenSBI (ECALL) | 原生 Linux 驱动 |
|:---|:---|:---|
| **控制路径** | `App -> Kernel -> ECALL -> OpenSBI -> Hardware` | `App -> Kernel -> Hardware` |
| **性能** | 较低（上下文切换开销） | 较高（直接 MMIO 访问） |
| **功能** | 基础（getchar/putchar） | 丰富（FIFO、流控、DMA、中断） |
| **灵活性** | 低（无法运行时配置） | 高（支持 stty 等工具） |
| **资源消耗** | 同步阻塞（轮询） | 异步中断（CPU 可执行其他任务） |

### 5.3 驱动验证 (`/proc/interrupts`, `dmesg`)

**验证中断模式生效:**

1. **查看 `/proc/interrupts`:**
```bash
# cat /proc/interrupts
           CPU0
 10:     12345   riscv-plic-intc  10  uart
```
如果看到 uart 条目且计数在键盘输入时增加，说明中断正在正常工作。

2. **检查内核启动日志:**
```bash
# dmesg | grep serial
[0.512345] 10000000.serial: ttyS0 at MMIO 0x10000000 (irq = 10, base_baud = 230400) is a 16550A
```
`irq = 10` 表示驱动已识别并使用中断号。

3. **设备树相关信息:**
- `interrupt-parent = <&plic0>`: 指向中断控制器
- `interrupts = <10>`: QEMU virt 平台 UART 固定使用中断号 10
- 8250_uart 驱动会自动请求中断并注册 ISR

## 6 总结

从依赖 OpenSBI 的轮询控制台迁移到内核原生的中断驱动控制台，核心在于修改设备树并确保内核配置支持中断驱动。结合 QEMU 模拟和 Linux 驱动，实现高效的 UART I/O 与中断处理。

关键要点：
1. **特权级理解**: M/S/U 模式分工明确，SBI 提供 M-Mode 服务
2. **设备树配置**: 正确配置 `stdout-path`、中断属性是关键
3. **中断模型**: CLINT 处理本地中断，PLIC 处理外部中断
4. **性能优化**: 从轮询模式切换到中断驱动显著提升效率
5. **验证方法**: 通过 `/proc/interrupts` 和 `dmesg` 确认中断工作正常
