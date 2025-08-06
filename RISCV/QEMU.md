# QEMU 相关内容

## 1 固件和外设


### 1.1 程序通过 OpenSBI 调用 UART 接口

在典型的 RISC-V 启动流程中，系统存在不同的特权级：

  - **M-Mode (Machine Mode):** 最高特权级，通常由固件（Firmware）运行，例如 OpenSBI。它直接控制硬件。
  - **S-Mode (Supervisor Mode):** 次高特权级，操作系统内核（如 Linux）运行在此模式。
  - **U-Mode (User Mode):** 最低特权级，用户空间的应用程序运行在此模式。

| 源模式 | 目标模式 | 触发指令 | 异常处理寄存器 | 返回指令 |
| :---: | :---: | :---: | :---: | :---: |
| U-mode | S-mode | ecall | scause, sepc | sret |
| S-mode | M-mode | ecall（未处理异常） | mcause, mepc | mret |
| M-mode | S-mode | mret（需配置 mepc, mstatus） | - | - |

当 Linux 内核作为 S-Mode 的程序运行时，它不能直接访问所有硬件。它需要通过 M-Mode 的 OpenSBI 提供的服务来与底层硬件交互，这是一种安全和标准的做法。这个交互的机制就是 **SBI (Supervisor Binary Interface)** 调用。

**调用流程如下：**

1.  **用户程序 (U-Mode):** 一个 C 程序调用 `printf("hello\n");`。
2.  **C 库 & 系统调用 (Syscall):** `printf` 函数最终会调用内核的 `write` 系统调用，将数据写入标准输出 (stdout)。
3.  **Linux 内核 (S-Mode):**
      * 内核的控制台子系统 (Console subsystem) 接收到这个写请求。
      * 在默认配置下，Linux 内核的早期控制台或主控制台被配置为使用 SBI。它知道自己不能直接写 UART 硬件寄存器。
      * 因此，它会准备 SBI 调用。对于字符输出，它会使用 `SBI_CONSOLE_PUTCHAR` 这个 SBI 扩展。
4.  **ECALL 指令:**
      * 内核执行一条特殊的指令 `ecall` (Environment Call)。这条指令会产生一个陷阱 (trap)，使处理器的执行从 S-Mode 切换到 M-Mode。
      * 在执行 `ecall` 之前，内核会按照 SBI 规范，将要调用的函数 ID（例如 `SBI_CONSOLE_PUTCHAR`）和参数（要打印的字符）放入指定的寄存器中（通常是 a7 和 a0-a6）。
5.  **OpenSBI (M-Mode):**
      * OpenSBI 在 M-Mode 下运行，它预先设置好了陷阱处理程序。
      * 当 `ecall` 发生时，OpenSBI 的陷阱处理程序接管控制权。
      * 它检查传入的寄存器，发现内核想要调用 `sbi_console_putchar` 函数。
      * OpenSBI 内部有自己的、非常基础的 UART 驱动。它会通过直接的内存映射 I/O (MMIO) 方式，**轮询 (Polling)** UART 的状态寄存器，直到可以发送数据，然后将字符写入 UART 的数据寄存器。
6.  **QEMU 模拟的 UART:**
      * QEMU 的 virt 平台模拟了一个标准的 UART 设备（通常是 `ns16550a` 兼容型号）。当 OpenSBI 写入该 UART 的模拟寄存器时，QEMU 会捕捉到这个操作，并将字符打印到宿主机的终端上。
7.  **返回流程:** OpenSBI 完成操作后，执行 `mret` (Machine Return) 指令，将控制权安全地返回给 Linux 内核的 S-Mode，程序继续执行。

这个过程的示意图：
`用户程序 -> C 库 -> 内核 Syscall -> 内核 SBI Console 驱动 -> ECALL -> OpenSBI (M-Mode) -> OpenSBI UART 驱动 (轮询) -> QEMU 模拟的 UART 硬件`

-----

### 1.2 HVC (Hypervisor Call)

#### 1.2.1 概念

  * **HVC (Hypervisor Call)** 是 ARM 架构中，用来从 Guest OS 陷入到 Hypervisor 的指令。
  * 在 RISC-V 中，与此概念对等的指令是 **ECALL (Environment Call)**，用于从低特权级（如 S-Mode）陷入到高特权级（如 M-Mode）。

---

#### 1.2.2 从 HVC 切换到 Linux 内核驱动

> 如何关闭基于 `ECALL` 的 SBI Console 调用，转而使用 Linux 内核自己的 UART 驱动，并将轮询改为中断

**方法：修改设备树 (Device Tree)**

设备树是描述硬件信息的数据结构，内核在启动时会解析它来了解平台的硬件布局。切换 UART 驱动的关键就在于修改设备树，告诉内核应该由谁来管理控制台。

**1. 使用 OpenSBI:**

QEMU 在启动时会动态生成一个设备树，或者你可以提供一个。在默认情况下，设备树的 `/chosen` 节点会指定 OpenSBI 作为标准输出，类似这样：

```dts
// dts snippet: Before modification
/ {
    chosen {
        // 这行告诉内核，标准输出是一个名为 "serial0" 的设备，
        // 由 OpenSBI 通过 SBI 接口提供服务。
        stdout-path = "serial0:115200n8";
    };

    // ... 其他节点 ...

    soc {
        serial@10000000 {
            compatible = "ns16550a";
            // ... 其他属性 ...
            // 注意：status 可能为 "disabled" 或未定义，
            // 或者内核因为 stdout-path 的存在而忽略它。
        };
    };
};
```

**使用原生驱动:**

要让 Linux 内核使用自己的原生驱动（例如 `8250_uart.c`），你需要做以下修改：

  * **移除或修改 `/chosen` 节点的 `stdout-path`**，使其指向真实的 UART 设备节点。
  * **确保 UART 设备节点被正确定义并启用 (`status = "okay"`)**。

修改后的设备树如下：

```dts
// dts snippet: After modification
/ {
    chosen {
        // 将 stdout-path 直接指向 UART 设备节点
        // 这告诉内核：“请使用你自己的驱动来管理这个设备作为控制台”
        stdout-path = &uart0; // 或者直接写路径 "/soc/serial@10000000"
    };

    // ... 其他节点 ...

    soc {
        // 这个节点现在是关键
        uart0: serial@10000000 {
            compatible = "ns16550a";
            reg = <0x0 0x10000000 0x0 0x100>;
            interrupt-parent = <&plic0>;
            interrupts = <10>;
            clock-frequency = <3686400>; // 提供时钟频率，用于计算波特率
            status = "okay"; // 明确启用该设备
        };
    };

    // ... PLIC (中断控制器) 节点定义 ...
    plic0: interrupt-controller@c000000 {
        // ...
    };
};
```

**如何操作？**

当用 QEMU 启动 Linux 时，你可以通过 `-dtb` 选项传入一个你自定义的、已编译的设备树文件 (`.dtb`)，来覆盖 QEMU 的默认设置。

```bash
qemu-system-riscv64 \
   -machine virt \
   -m 2G \
   -nographic \
   -kernel path/to/Image \
   -bios path/to/opensbi/fw_jump.bin \
   -append "root=/dev/vda rw console=ttyS0" \
   -dtb path/to/your-modified-virt.dtb # <-- 在这里指定你的设备树
```

**3. 修改后的区别**

| 特性 | 使用 OpenSBI (ECALL) | 使用原生 Linux 驱动 |
| :--- | :--- | :--- |
| **控制路径** | `App -> Kernel -> ECALL -> OpenSBI -> Hardware` | `App -> Kernel -> Hardware` |
| **性能** | **较低**。每次字符 I/O 都需要 `ECALL` 陷入 M-Mode，有上下文切换开销。 | **较高**。内核直接通过 MMIO 访问硬件，避免了陷入开销。 |
| **功能** | **基础**。OpenSBI 只提供最基本的 `getchar` 和 `putchar` 功能。 | **丰富**。Linux 的 `8250_uart` 驱动支持 FIFO 缓冲、流控、DMA、中断等高级功能。 |
| **驱动位置** | UART 驱动在 M-Mode 的 OpenSBI 固件中。 | UART 驱动在 S-Mode 的 Linux 内核中。 |
| **灵活性** | **低**。你无法在 Linux 运行时动态修改 UART 参数（如流控）。 | **高**。可以使用 `stty` 等标准工具在运行时配置串口。 |
| **资源消耗** | `ECALL` 是一种**同步阻塞**调用，会占用 CPU 等待 I/O 完成（轮询）。 | 如果配置为中断模式，CPU 在等待 I/O 时可以执行其他任务，**效率更高**。 |

**4. 如何修改内核支持基于中断的 UART 调用**

默认情况下，即使你切换到了原生驱动，早期的控制台 (`earlycon`) 和一些默认配置可能仍然使用轮询模式，因为它更简单，不需要中断系统完全初始化。要确保使用高效的**中断驱动模式**，你需要确保以下几点：

**4.1. 设备树 (DTS) 是关键**

这是最重要的一步。你必须在设备树的 UART 节点中正确地声明中断信息，就像上面那个例子一样：

```dts
uart0: serial@10000000 {
    ...
    // 告诉内核这个设备的中断信号连接到哪个中断控制器
    interrupt-parent = <&plic0>;
    // 告诉内核这个设备使用的是该控制器的第 10 号中断线
    interrupts = <10>;
    ...
};
```

  * `interrupt-parent`: 指向平台的中断控制器（在 RISC-V `virt` 平台通常是 PLIC）。
  * `interrupts`: 指定该设备使用的中断号。对于 QEMU `virt` 机器，UART 的中断号固定为 10。

当 `8250_uart` 驱动在初始化时看到这些属性，它会自动请求这个中断，并注册一个中断处理函数 (Interrupt Service Routine, ISR)。

**4.2. 内核配置 (`menuconfig`)**

确保你的内核编译时包含了必要的驱动支持。通常这些都是默认开启的。

```
Device Drivers --->
    Character devices --->
        Serial drivers --->
            <*> 8250/16550 and compatible serial support
            [*]   Console on 8250/16550 and compatible serial port
```

  * `CONFIG_SERIAL_8250=y` (或 `m`)
  * `CONFIG_SERIAL_8250_CONSOLE=y`

驱动本身已经包含了处理中断的逻辑，你不需要修改 C 代码，只需要通过设备树提供正确的信息即可。

**4.3. 验证修改是否生效**

系统启动后，你可以通过以下方式验证 UART 是否工作在中断模式：

  * **查看 `/proc/interrupts`:**
    这是最直接的方法。你会看到一个与 UART 相关的中断计数。

    ```bash 
    # cat /proc/interrupts
               CPU0
     10:     12345   riscv-plic-intc  10  uart
    ```

    这里的 `10:` 是中断号，`12345` 是中断发生的次数。如果你在终端上疯狂敲键盘，这个数字会快速增加，这证明了中断正在被正确处理。如果这里没有 uart 的条目，或者数字从不变化，那么它可能仍在轮询模式。

  * **检查内核启动日志 (`dmesg`)**:
    在启动日志中，你应该能看到类似这样的信息，表明 8250 驱动已经接管了设备并将其注册为 `ttyS0`。

    ```
    [    0.512345] 10000000.serial: ttyS0 at MMIO 0x10000000 (irq = 10, base_baud = 230400) is a 16550A
    ```

    `irq = 10` 这部分明确表示驱动已经识别并使用了中断号。

**总结**

从依赖 OpenSBI 的轮询控制台，迁移到内核原生的中断驱动控制台，核心是**修改设备树**。这不仅能带来显著的性能提升和功能增强，也是理解现代嵌入式 Linux 系统中硬件与软件如何解耦和交互的一个绝佳范例。


## 1.3 QEMU UART 模拟器

- 对应的设备文件: 
  - `qemu/hw/char/serial.c`
  - `qemu/include/hw/char/serial.h`

- 包含的寄存器内容
  - `divider` (uint16_t):
    - 波特率除数锁存器 。这个 16 位的值用于设置串口通信的波特率。波特率的计算公式通常是 input_clock / (16 * divider) 。要访问这个寄存器，必须先将线路控制寄存器（`LCR`）中的 `DLAB` (Divisor Latch Access Bit) 位置为 1。
  
  - `rbr` (uint8_t):
    - 接收缓冲寄存器 (Receive Buffer Register) 。当串口接收到数据时，数据被存放在这里。CPU 通过读取这个寄存器来获取接收到的字符。读取 `RBR` 会从接收 FIFO (First-In, First-Out) 队列中取出一个字节。
  
  - `thr` (uint8_t):
    - 发送保持寄存器 (Transmit Holding Register) 。CPU 想要发送数据时，会将数据写入这个寄存器。写入 `THR` 的数据会被放入发送 FIFO 队列，等待被发送。
  
  - `tsr` (uint8_t):
    - 发送移位寄存器 (Transmit Shift Register) 。这是一个内部寄存器，不直接对 CPU 可见。它保存当前正在通过串行线按位发送的字符。当 `tsr` 发送完一个字符后，会从 `thr` (或发送 FIFO) 中加载下一个字符。
  
  - `ier` (uint8_t):
    - 中断使能寄存器 (Interrupt Enable Register) 。这个寄存器用来控制 UART 在特定事件发生时是否产生中断。例如，可以使能“接收到数据”、“发送保持寄存器为空”等中断。
  
  - `iir` (uint8_t):
    - 中断识别寄存器 (Interrupt Identification Register) 。这是一个只读寄存器，用于识别当前挂起的中断源。当中断发生时，CPU 可以读取 `IIR` 来确定中断的原因。
  
  - `lcr` (uint8_t):
    - 线路控制寄存器 (Line Control Register) 。用于配置串行通信的参数，如数据位长度（5, 6, 7, or 8 bits）、停止位数（1, 1.5, or 2 bits）和奇偶校验方式（无、奇、偶等）。它还包含 `DLAB` 位，用于控制对波特率除数锁存器的访问。
  
  - `mcr` (uint8_t):
    - 调制解调器控制寄存器 (Modem Control Register) 。用于控制与调制解调器（Modem）的接口信号，如 `DTR` (Data Terminal Ready) 和 `RTS` (Request to Send)。它也包含一个用于测试的环回模式开关。
  
  - `lsr` (uint8_t):
    - 线路状态寄存器 (Line Status Register) 。这是一个只读寄存器，提供了数据传输的状态信息。例如，它会指示接收缓冲器中是否有数据（Data Ready）、发送保持寄存器是否为空（Transmitter Holding Register Empty），以及是否发生了溢出、奇偶校验或帧错误。
  
  - `msr` (uint8_t):
    - 调制解调器状态寄存器 (Modem Status Register) 。这是一个只读寄存器，反映了调制解调器接口信号的当前状态，如 `CTS` (Clear to Send)`、DSR` (Data Set Ready) 等。

  - `scr` (uint8_t):
    - 暂存寄存器 (Scratch Register) 。一个通用的 8 位读/写寄存器，可以被软件用于任何临时存储目的，硬件本身不使用它。
  
  - `fcr` (uint8_t):
    - FIFO 控制寄存器 (FIFO Control Register) 。用于启用和管理 16550A UART 的 FIFO 缓冲。可以设置接收 FIFO 的触发级别（即接收多少字节后产生中断）、清除发送和接收 FIFO 等。

  - `fcr_vmstate` (uint8_t):
    - 用于虚拟机状态迁移的 `FCR` 。直接写入 FCR 寄存器会产生副作用（例如，清除 FIFO）。为了在 QEMU 进行虚拟机迁移（save/load state）时能正确保存和恢复 FCR 的状态而不触发这些副作用，QEMU 使用这个 `fcr_vmstate` 字段来临时存储 FCR 的值。在加载状态后， post_load 函数会负责将这个值安全地写入真实的 FCR。