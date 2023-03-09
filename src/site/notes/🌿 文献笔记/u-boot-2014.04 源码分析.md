---
{"dg-publish":true,"permalink":"/🌿 文献笔记/u-boot-2014.04 源码分析/","tags":["Language/c","Language/assembly","源码分析"]}
---


由于要移植一个 kernel 和 u-boot 到 smart210 平台，所以就有了该篇记录。

本文使用的版本是 ~~[u-boot-2022.04-rc4.tar.bz2](ttps://ftp.denx.de/pub/u-boot/u-boot-2022.04-rc4.tar.bz2)~~ [u-boot-2014.04](https://github.com/Gonglja/u-boot-smart210)（因作者能力不够，遂从 2014.04 开始）

## s5pv210

### 芯片启动流程

由三星官方手册可知，整个启动分为三阶段，BL0、BL1、BL2，其中 BL0 运行在内部 iROM 中，BL1 和 BL2 运行在 SRAM 中，但在实际项目中使用的比较通用的启动流程与官方流程有较大差异，为什么呢？因为编译的 u-boot 过大，超过最大 80k 限制。所以 BL2 在 SDRAM 中运行，而三星官方是放在内部 SRAM 中运行。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204082006861.png)

#### BL0

- 运行在 **iROM** 上
- 代码固定在 s5pv210 的 IROM 中，无法修改。上电后直接从 IROM 中开始执行
- 主要工作
  - 初始化系统时钟、特殊设备控制器、启动设备、看门狗、SRAM 等
  - 验证 BL1 镜像
  - 从存储介质上（比如 SD/eMMC/nand flash）等加载 BL1 镜像到内部 SRAM 中
  - 跳转到 BL1 镜像所在地址

#### BL1

- 运行在 **SRAM** 上
- BL1 代码在 BL0 段被加载到 SRAM 中
- 主要工作
  - 初始化部分时钟（SDRAM 相关）
  - 初始化 DDR（外部 SDRAM）
  - 从存储介质上将 BL2 镜像加载到 SDRAM 中
  - 验证 BL2 镜像合法性
  - 跳转到 BL2 镜像所在的地址上

#### BL2

- 运行在 **SDRAM** 上
- BL2 代码在 BL1 段被加载到 SDRAM 中
- BL2 就是传统意义上的 bootloader，主要负责加载 OS 和启动 OS

### 地址映射

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204082025672.png)

| 起始地址        | 结束地址        | 长度       | 映射区域描述                    |
| --------------- | --------------- | ---------- | ------------------------------- |
| **0x0000_0000** | **0x1FFF_FFFF** | **512MB**  | **Boot area（取决于启动模式）** |
| **0x2000_0000** | **0x3FFF_FFFF** | **512MB**  | **DRAM 0**                      |
| **0x4000_0000** | **0x7FFF_FFFF** | **1024MB** | **DRAM 1**                      |
| 0x8000_0000     | 0x87FF_FFFF     | 128MB      | SROM Bank 0                     |
| 0x8800_0000     | 0x8FFF_FFFF     | 128MB      | SROM Bank 1                     |
| 0x9000_0000     | 0x97FF_FFFF     | 128MB      | SROM Bank 2                     |
| 0x9800_0000     | 0x9FFF_FFFF     | 128MB      | SROM Bank 3                     |
| 0xA000_0000     | 0xA7FF_FFFF     | 128MB      | SROM Bank 4                     |
| 0xA800_0000     | 0xAFFF_FFFF     | 128MB      | SROM Bank 5                     |
| 0xB000_0000     | 0xBFFF_FFFF     | 256MB      | OneNAND/NAND Controller and SFR |
| 0xC000_0000     | 0xCFFF_FFFF     | 256MB      | MP3_SRAM output buffer          |
| **0xD000_0000** | **0xD000_FFFF** | **64KB**   | **IROM**                        |
| 0xD001_0000     | 0xD001_FFFF     | 64KB       | Reserved                        |
| **0xD002_0000** | **0xD003_7FFF** | **96KB**   | **IRAM**                        |
| 0xD800_0000     | 0xDFFF_FFFF     | 128MB      | DMZ ROM                         |
| 0xE000_0000     | 0xFFFF_FFFF     | 512MB      | SFR region                      |

s5pv210 芯片上电之后，CPU 会直接从 0x0 地址取指令，也就是直接运行 BL0。

| 起始地址    | 结束地址    | 长度 | 映射区域描述 |
| ----------- | ----------- | ---- | ------------ |
| 0x0000_0000 | 0x0000_FFFF | 64KB | Internal ROM |

BL1 运行在 IRAM 中，IRAM 空间如下

| 起始地址    | 结束地址    | 长度 | 映射区域描述 |
| ----------- | ----------- | ---- | ------------ |
| 0xD002_0000 | 0xD003_7FFF | 96KB | IRAM         |

但需注意：BL1 运行在 IRAM 上，但并不意味着就从 0xD002_0000 开始。其实 0xD002_0000 开头的 16B 被用做 BL1 的 header，BL1 真正是从 **0xD002_0010** 开始运行的。前 16 字节被用来验证 BL1 镜像的完整性，其格式如下：

| 地址        | 数据                         |
| ----------- | ---------------------------- |
| 0xD002_0000 | BL1 镜像包括 header 的长度   |
| 0xD002_0004 | 保留，设置为 0               |
| 0xD002_0008 | BL1 镜像除去 header 的校验和 |
| 0xD002_000c | 保留，设置为 0               |

BL2 运行地址在 SDRAM 中，smart210 使用的是 DRAM0，所以地址为

| 起始地址    | 结束地址    | 长度  | 映射区域描述 |
| ----------- | ----------- | ----- | ------------ |
| 0x2000_0000 | 0x3FFF_FFFF | 512MB | DRAM 0       |

## u-boot

由以上可知，上电后首先运行 BL0 阶段（BL0 段代码起始地址为 `0`）。

BL0 阶段运行的代码为 IROM 中自带的，其主要功能为 初始化系统时钟、特殊设备控制器、启动设备、看门狗、SRAM 等，验证 BL1 镜像是否完整（`0xD002_0000` 处存在长度和校验）然后跳转到 BL1 处（`0xD002_0010`）执行。

BL1 阶段运行在 IRAM 中，其主要功能是初始化部分时钟和 DDR，拷贝 BL2 段代码到 SDRAM 中，验证 BL2 并跳转执行。

> 那么，BL1 段代码是怎么被拷贝到 SRAM 中的？
>
> 答：在 BL0 段将 BL1 的代码拷贝到 SRAM 中（参考 `S5PV210_iROM_ApplicationNote_Preliminary_20091126.pdf` 中 `2.2 iROM(BL0) boot-up sequence (Refer 2.3 V210 boot-up diagram)`），由于 BL0 三星不开源，所以我我们只能按照三星的要求存放 BL1 的位置（参考 `S5PV210_iROM_ApplicationNote_Preliminary_20091126.pdf` 中 `2.8 Boot Block Assignment Guide`），如果从 SD 卡启动，也即从 SD 卡的第 `1` 个块开始；如果从 NAND Flash 启动，就放到 Flash 中的 `0` 块处。拷贝多少呢？这个不知道，有人说是 `16k` 也有人说是 `8k`，尽可能将 BL1 的代码控制在 `8K` 以内。

BL2 阶段运行在 SDRAM 中，就是传统意义上的 bootloader，主要负责加载 OS 和启动 OS

u-boot 代码中有两部分：u-boot-spl、u-boot，其中 u-boot-spl 与芯片启动流程中的 BL1 对应、u-boot 与 BL2 对应。

那么问题来了，u-boot-spl 是如何生成的？u-boot 又是怎么生成的?

spl 的编译时编译 uboot 的一部分，和 uboot.bin 走的是两条编译流程。

一般来说，会先编译主体 uboot，也就是 uboot.bin，在编译 uboot-spl，也就是 uboot-spl.bin，两个流程。

编译成功后有几个文件名需要注意下，

| 文件                 | 说明                                                                                   |
| -------------------- | -------------------------------------------------------------------------------------- |
| u-boot-spl           | 初步链接后得到的 spl 文件                                                              |
| u-boot-spl-nodtb.bin | 在 u-boot-spl 的基础上，经过 objcopy 去除符号表信息之后的可执行程序                    |
| u-boot-spl.bin       | 在不需要 dtb 的情况下，直接由 u-boot-spl-nodtb.bin 复制而来，也就是编译 spl 的最终目标 |
| smart210-spl.bin     | 由 s5pv210 平台决定，需要在 u-boot-spl.bin 的基础上加上 16B 的 header 用作校验         |
| u-boot-spl.lds       | spl 的连接脚本                                                                         |
| u-boot-spl.map       | 连接之后的符号表文件                                                                   |
| u-boot-spl.cfg       | 由 spl 配置生成的文件                                                                  |

**框架**

> 一般情况下，u-boot 采用 "board-->machine-->arch-->cpu" 框架，如图：

> ![](http://www-x-wowotech-x-net.img.abc188.com/content/uploadfile/201605/29bd3da4b061810a74093c33d3292b4320160519144243.gif)

> 基于这个架构，u-boot 和平台有关的初始化流程就很清晰了。
>
> 1. u-boot 启动后，会先执行 CPU（如 armv8）的初始化代码
> 2. CPU 相关的代码，会调用 ARCH 的公共代码（如 arch/arm）
> 3. ARCH 的公共代码，在适当的时候，调用 board 有关的接口。u-boot 的功能逻辑，大多是由 common 代码实现，部分和平台有关的部分，则由公共代码声明，由 board 代码实现。
> 4. board 代码在需要的时候，会调用 machine(arch/arm/mach-xxx) 提供的接口，实现特定的功能。因此 machine 的定位是提供一些基础的代码支持，不会直接参与到 u-boot 的逻辑功能中去。

由此可知，当 u-boot 上电后，首先执行的是 CPU 的初始化代码。板卡型号为 smart210，CPU 型号为 s5pv210，armv7 指令集。

通过 `u-boot-spl.lds`（`arch/arm/cpu/u-boot-spl.lds`）和 `u-boot.lds`（arch/arm/cpu/u-boot.lds）中 `ENTRY(_start)` 可知，不管是 u-boot-spl 还是 u-boot 都是从 `_start` 开始的。

## 代码分析

首先解决编译问题，其次是分析代码，在分析代码的过程中遇到不会的指令和寄存器及时记录下来，防止下次还是不会。

### 编译问题

#### 错误 1

/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found

dirname: missing operand

具体如下

```bash
glj0@glj0-ubuntu21:~/worksapce/os/smart210/u-boot-2014.04$ ./make.sh
make: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: No such file or directory
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
dirname: missing operand
Try 'dirname --help' for more information.
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
dirname: missing operand
Try 'dirname --help' for more information.
make: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: No such file or directory
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
dirname: missing operand
Try 'dirname --help' for more information.
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
dirname: missing operand
Try 'dirname --help' for more information.
Configuring for smart210 board...
make: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: No such file or directory
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
dirname: missing operand
Try 'dirname --help' for more information.
  GEN     include/autoconf.mk.dep
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
  GEN     include/autoconf.mk
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
  CHK     include/config/uboot.release
  CHK     include/generated/timestamp_autogenerated.h
  UPD     include/generated/timestamp_autogenerated.h
  UPD     include/config/uboot.release
  HOSTCC  scripts/basic/fixdep
  CHK     include/generated/version_autogenerated.h
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-ld: not found
  UPD     include/generated/version_autogenerated.h
  CC      lib/asm-offsets.s
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
make[1]: *** [/home/glj0/worksapce/os/smart210/u-boot-2014.04/./Kbuild:35: lib/asm-offsets.s] Error 127
make[1]: *** Waiting for unfinished jobs....
  CC      arch/arm/lib/asm-offsets.s
/bin/sh: 1: /opt/FriendlyARM/toolschain/4.5.1/bin/arm-none-linux-gnueabi-gcc: not found
make[1]: *** [/home/glj0/worksapce/os/smart210/u-boot-2014.04/./Kbuild:84: arch/arm/lib/asm-offsets.s] Error 127
make: *** [Makefile:999: prepare0] Error 2

```

缺少 32 位库，安装就可以了。

```shell
sudo apt-get install libgl1-mesa-dri:i386
```

### 寄存器

> 32 位处理器能同时处理 32 位的数据，所以对应寄存器为 32 位的
>
> 64 位处理器能同时处理 64 位的数据，所以对应寄存器为 64 位的
>
> ARM 处理器用到的指令集分为 ARM 和 THUMB 两种。
>
> ARM 指令集长度固定为 32bit，THUMB 指令集长度固定为 16bit。ARM64 指令集长度也是 32bit。

| 类别                    | 寄存器 | APCS(ARM 过程调用标准)                    | 说明                                                                                                                         |
| ----------------------- | ------ | ----------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 通用寄存器/不分组寄存器 | r0     | a1                                        | 用作传入函数参数，传出函数返回值。在子程序调用之间，可以将 r0-r3 用于任何用途                                                |
|                         | r1     | a2                                        | 同上                                                                                                                         |
|                         | r2     | a3                                        | 同上                                                                                                                         |
|                         | r3     | a4                                        | 同上                                                                                                                         |
|                         | r4     | v1                                        | **存放函数的局部变量** 如果被调用函数使用了这些寄存器，它在返回之前必须恢复这些寄存器的值                                     |
|                         | r5     | v2                                        | 同上                                                                                                                         |
|                         | r6     | v3                                        | 同上                                                                                                                         |
|                         | r7     | v4                                        | 同上                                                                                                                         |
| 通用寄存器/分组寄存器   | r8     | v5                                        | 同上                                                                                                                         |
|                         | r9     | v6                                        | 同上                                                                                                                         |
|                         | r10    | sl (stack limit)                          | 同上                                                                                                                         |
|                         | r11    | fp ()                                     | 同上                                                                                                                         |
|                         | r12    | ip (intra-prpcedure-call scratch regiser) | 内部调用暂时寄存器。|
|                         | r13    | sp (stack pointer)                        | 栈指针 sp，存放的值在退出被调用函数时必须与进入时的值相同。|
|                         | r14    | lr (link register)                        | 通常被用作子程序链接寄存器，也称 lr,指向函数的返回地址。通常用来保存子程序执行的下一条指令。|
| 通用寄存器/程序计数器   | r15    | pc (program counter)                      | 程序计数器，保留下一条 CPU 即将执行的指令。不能用于其它用途。|
|                         |        |                                           |                                                                                                                              |
| 程序状态字寄存器        | cpsr   |                                           |                                                                                                                              |
|                         | spsr   |                                           |                                                                                                                              |
|                         |        |                                           |                                                                                                                              |
| 协处理器                | cp15   |                                           | 在基于 ARM 的嵌入式应用系统中，存储系统的操作通常是由协处理器 CP15 完成的。CP15 包含 16 个 32 位的寄存器，其编号为 0～15。|

**CP15** 的寄存器列表如表所示：

| 寄存器编号 | 基本作用         | 在 MMU 中的作用      | 在 PU 中的作用       |
| ---------- | ---------------- | -------------------- | -------------------- |
| 0          | ID 编码（只读）| ID 编码和 cache 类型 |                      |
| 1          | 控制位（可读写）| 各种控制位           |                      |
| 2          | 存储保护和控制   | 地址转换表基地址     | Cachability 的控制位 |
| 3          | 存储保护和控制   | 域访问控制位         | Bufferablity 控制位  |
| 4          | 存储保护和控制   | 保留                 | 保留                 |
| 5          | 存储保护和控制   | 内存失效状态         | 访问权限控制位       |
| 6          | 存储保护和控制   | 内存失效地址         | 保护区域控制         |
| 7          | 高速缓存和写缓存 | 高速缓存和写缓存控制 |                      |
| 8          | 存储保护和控制   | TLB 控制             | 保留                 |
| 9          | 高速缓存和写缓存 | 高速缓存锁定         |                      |
| 10         | 存储保护和控制   | TLB 锁定             | 保留                 |
| 11         | 保留             |                      |                      |
| 12         | 保留             |                      |                      |
| 13         | 进程标识符       | 进程标识符           |                      |
| 14         | 保留             |                      |                      |
| 15         | 因不同设计而异   | 因不同设计而异       | 因不同设计而异       |

### 指令

**指令集二进制编码**

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204271909384.png)

**条件码**

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204131656507.png)

**指令**

| 类别                 | 指令                                                 | 说明                                                                                                                                                                                                                                                                                                                                                                                                  | 示例                                                                                                                                                        |
| -------------------- | ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 条件（有符号数）| `GE `                                                | 大于等于                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                             |
|                      | `LE `                                                | 小于等于                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                             |
|                      | `GT`                                                 | 大于                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                             |
|                      | `LT `                                                | 小于                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                             |
| 条件（用于无符号数）| `HS`                                                 | 大于等于                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                             |
|                      | `LS `                                                | 小于等于                                                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                             |
|                      | `HI `                                                | 大于                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                             |
|                      | `LO`                                                 | 小于                                                                                                                                                                                                                                                                                                                                                                                                  |                                                                                                                                                             |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `STR{cond} 源寄存器，<存储器地址>`                   | 从源寄存器中将一个 32 位的字数据传送到存储器中                                                                                                                                                                                                                                                                                                                                                        | `STR r1, [r0]` 将 r1 里面值复制到以 r0 里面的值作为地址的内存中                                                                                              |
|                      | `STRLO 源寄存器，<存储器地址>`                       | 当满足条件小于时，从源寄存器中将一个 32 位的字数据传送到存储器中                                                                                                                                                                                                                                                                                                                                      | `CMP r0，r1;STRLO r2，[r0];` 首先 r0=r0-r1,当满足条件 r0<r1 时，将 r2 中的值复制到以 r0 里面的值作为地址的内存中。|
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
| 地址读取伪指令       | `ADR{cond}目的寄存器，相对地址或标签 `               | 将基于 PC 相对偏移的地址值或基于寄存器相对地址值传送到目的寄存器中（小范围）| `ADR r0, here;here: ....` 将标签 here 处的相对地址传递给寄存器 r0                                                                                            |
|                      | `LDR{cond}目的寄存器，<存储器地址>`                  | 从存储器中将一个 32 位的字数据传送到目的寄存器中（大范围）| `LDR r0, [r1]` 将 r1 里面的值作为地址，将地址里面的值复制给寄存器 r0                                                                                         |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
| 跳转指令             | `B{cond} 目标地址 `                                  | 跳转指令                                                                                                                                                                                                                                                                                                                                                                                              | `B Label` 程序无条件跳转到标号 Label 处执行                                                                                                                  |
|                      | `BL{cond} 目标地址 `                                 | 带返回的跳转指令（跳转之前，将 PC 内容存到 R14 中）| `BL Label` 程序将当前 PC 值存到 R14 中，程序无条件跳转至标号 Label 中执行                                                                                    |
|                      | `BX{cond} Rm `                                       | 带状态切换的跳转指令，最低位为 1 时，切换到 Thumb 指令执行，为 0 时，解释为 ARM 指令执行（最低位指的是 Rm 的第 0 位）|                                                                                                                                                             |
|                      | `BLX                                                 | 带返回和状态切换的跳转指令                                                                                                                                                                                                                                                                                                                                                                            |                                                                                                                                                             |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
| 算术运算指令         | `SUB 寄存器 1，寄存器 2，寄存器 3`                      | 寄存器 1=寄存器 2-寄存器 3                                                                                                                                                                                                                                                                                                                                                                            | `SUB r0，r1,#1` r0=r1-1                                                                                                                                     |
|                      | `SUBS 寄存器 1，寄存器 2，寄存器 3 `                    | 寄存器 1=寄存器 2-寄存器 3;S 表示并将进位结果写到 CPSR 中                                                                                                                                                                                                                                                                                                                                             | `SUBS r0，r1,#1` r0=r1-1，并将进位结果写道 CPSR                                                                                                             |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `CMP 寄存器 1，寄存器 2 `                              | 寄存器 1=寄存器 1-寄存器 2                                                                                                                                                                                                                                                                                                                                                                            | `CMP r0，r1` 也就是`r0=r0-r1`                                                                                                                               |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
| 多数据传输条件       | `IA（Increase After）`                              | 每次传送后地址加 4,其中的寄存器**从左到右执行**,例如:STMIA R0,{R1,LR} 先存 R1,再存 LR                                                                                                                                                                                                                                                                                                                 |                                                                                                                                                             |
|                      | `IB（Increase Before）`                             | 每次传送前地址加 4,同上                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                                             |
|                      | `DA（Decrease After）`                               | 每次传送后地址减 4,其中的寄存器**从右到左执行**,例如:STMDA R0,{R1,LR} 先存 LR,再存 R1                                                                                                                                                                                                                                                                                                                 |                                                                                                                                                             |
|                      | `DB（Decrease Before）`                             | 每次传送前地址减 4,同上                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                                             |
|                      | `FD`                                                 | 满递减堆栈 (每次传送前地址减 4)                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `FA`                                                 | 满递增堆栈 (每次传送后地址减 4)                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `ED`                                                 | 空递减堆栈 (每次传送前地址加 4)                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `EA`                                                 | 空递增堆栈 (每次传送后地址加 4)                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
| 多数据传输（加载）| `LDM{cond} Rn{!},reglist{^} `                        | 多数据加载，将地址上的值加载到寄存器中。Rn：基址寄存器，装有传送数据的起始地址，Rn 不允许为 R15；！：表示最后的地址写回到 Rn 中；reglist：可包含多于一个寄存器范围，用“，”隔开，如{R1，R2，R6-R9}，寄存器由小到大顺序排列；^：不允许在用户模式和系统模式下运行                                                                                                                                        | `LDR R0,=0x100000;LDMIA R0!,{R1-R8}`也就是从左往右加载，首先将 0x100000 中的数据加载到 R1,然后 R0=R0+4，然后将 0x1000004 中的数据加载到 R2,然后 R0=R0+4.... |
| 多数据传输（存储）| `STM{cond} mode Rn{!}, reglist{^} `                  | 多数据存储，将寄存器的值存到地址上。同上                                                                                                                                                                                                                                                                                                                                                              |                                                                                                                                                             |
| 协处理器 CP15 操作   | `MRC{cond} p15,<Opcode1>,<Rd>,<CRn>,<CRm>,<Opcode2>` | 协处理器寄存器到 ARM 处理器寄存器的数据传送指令（读出协处理器寄存器）。cond 为指令执行条件码，当忽略时无条件执行。Opcode1:协处理器的特性操作码，对于 CP15 来说，其为 0。Rd：源寄存器的 ARM 寄存器，其值被传送到协处理器中或者将协处理器的值传送到该寄存器中。CRn:目标寄存器的的协处理器寄存器，C~C15。CRm：协处理器中附件的目标寄存器或源操作数寄存器，默认为 c0。Opcode2：可选的协处理器特定操作码。| `mrc p15, 0, r0, c1, c0, 0` 将 CP15 的寄存器 C1 的值读到 r0 中                                                                                              |
|                      | `MCR{cond} p15,<Opcode1>,<Rd>,<CRn>,<CRm>,<Opcode2>` | ARM 处理器寄存器到协处理器寄存器的数据传送指令（写入协处理器寄存器）| `mcr p15, 0, r0, c7, c7, 0` 关闭 ICaches 和 DCachesmcr <br />`p15, 0, r0, c8, c7, 0` 使无效整个数据 TLB 和指令 TLB                                          |
|                      |                                                      |                                                                                                                                                                                                                                                                                                                                                                                                       |                                                                                                                                                             |
|                      | `.globl`                                             | 伪指令，声明一个全局变量                                                                                                                                                                                                                                                                                                                                                                              | `.globl _start`声明\_start，下面调用时才不会出现链接错误。类似 c 语言中的 extern。|

### 源码分析

#### ❎~~2022 中的\_start 不看~~

~~所以首先会执行 armv7 中的 start.S 代码，但是呢，通过代码，我们没有发现 start.S 中有 `_start` 相关的定义，而且也没有一些向量表的处理，这就很奇怪了，代码放哪呢了呢？没有 `_start` 又要怎么启动呢？带着问题，我们接着往下走。~~

由于我们的代码是最新的，但有印象在 2014 版本中是有这一部分的，所以我们直接去查找 git 上这个文件的 history

于是有了找到了以下

https://source.denx.de/u-boot/u-boot/-/commit/41623c91b09a0c865fab41acdaff30f060f29ad6

在这个链接右上角可以看到，删除了 2426 行，新增了 313 行。看了下发现删除的都是 下面这种代码

```ass
.global _start
_start: b    reset
...
	.balignl 16,0xdeadbeef
```

那这些代码去哪了呢？

接着搜索 armv7，发现也是这种代码被删除。

接着走，看看新增了什么，在 u-boot-spl.lds 和 u-boot.lds 中 代码中新增了一个 \*(vectors) 并且位置比 start.o 靠前，什么意思？中断向量表存到这个地方了嘛？是的，接着往下看，一切都会真相大白

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202203291016583.png)

终于看到 vector.S 了，这部分代码是存在 arch/arm/lib 下，有没有发现很熟悉，没错，就是上面删掉的代码，而且 `.global _start` 也在这，由此可以猜测，**开发人员将各个处理器的公共代码抽离，最终以 vector.S 体现**。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202203291030263.png)

借助 u-boot.lds（arch/arm/cpu/u-boot.lds）可知，`ENTRY(_start)`，整个 u-boot 的开始依然是 `_start`，由以上分析可知，`_start` 在 vectors.S 中定义，所以整个 U-boot 的入口为 vector.S 中的 `_start`，接着 ARM_VECTORS 是一个宏，展开就是 `.macro/.endm` 之间包裹的。接着往下走，就会走入 `b reset`，调用 reset。

```c
/* arch/arm/lib/vectors.S */
.globl _start

	.section ".vectors", "ax"

#if defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK)
#include <asm/arch/boot0.h>
#else   /*走这个分支 (怎么确认的呢？在此处随便加个乱七八糟的字符，然后编译，如果编译过，则说明走的不是此分支)*/
_start: /*U-boot程序入口*/
#ifdef CONFIG_SYS_DV_NOR_BOOT_CFG
	.word	CONFIG_SYS_DV_NOR_BOOT_CFG
#endif
	ARM_VECTORS /* 一个宏*/
#endif /* !defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK) */
	/* 在.macro 和 .endm 中的为*/
    .macro ARM_VECTORS
#ifdef CONFIG_ARCH_K3
	ldr     pc, _reset
#else
	b	reset	// 走这个分支
#endif
	ldr	pc, _undefined_instruction
	ldr	pc, _software_interrupt
	ldr	pc, _prefetch_abort
	ldr	pc, _data_abort
	ldr	pc, _not_used
	ldr	pc, _irq
	ldr	pc, _fiq
	.endm


```

reset 标签在哪呢？`arch/arm/cpu/armv7/start.S` 中

```c
/* arch/arm/cpu/armv7/start.S */

reset:
	/* Allow the board to save important registers */
	b	save_boot_params
save_boot_params_ret:

/*
 * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
 * except if in HYP mode already
 * 禁用中断，同时设置CPU模式，除非已经处于HYP模式下
 */
    mrs		r0, cpsr			;读取寄存器cpsr中的值，并保存到r0寄存器中。
    and		r1, r0, #0x1f		@ mask mode bits
    teq		r1, 	#0x1a		@ test for HYP mode
    bicne	r0, r0, #0x1f		@ clear all mode bits
    orrne	r0, r0, #0x13		@ set SVC mode
    orr		r0, r0, #0xc0		@ disable FIQ and IRQ
    msr		cpsr,r0

ENTRY(save_boot_params)
	b	save_boot_params_ret		@ back to my caller
ENDPROC(save_boot_params)
```

`mrs r0, cpsr` ; 读取寄存器 cpsr 中的值，并保存到 r0 寄存器中。

`and r1, r0, #0x1f` 寄存器 r0 中的值与 0X1F 进行与运算，结果保存到 r1 寄存器中，目的就是提取 cpsr 的 bit0~bit4 这 5 位，这 5 位为 M4 M3 M2 M1 M0，M[4:0] 这五位用来设置处理器的工作模式

以上为 u-boot-2022.04 中的内容

---

#### **代码函数调用图**

https://www.zhixi.com/view/18da9b99

#### \_start

由于我们的处理器使用的指令集为 armv7，所以**\_start**的路径为 `arch/arm/cpu/armv7/start.S`

其实对于 u-boot 的 `start.S`，主要做的几件事就是系统各方面的初始化。

从大的方面分，可分为以下部分

- 设置 CPU 模式

- 关闭看门狗

- 关闭中断

- 设置堆栈 sp 指针

- 清除 bss 段

- 异常中断处理

接着我们看代码，

`.globl _start` 声明 `_start` 标号对全局可见，类似于 c 语言中的 `extern`

`_start: b reset` `_start` 后加一个冒号，表示其实一个 Label。而同时 `_start` 的值，也就是代码的最开始的位置，相对是 `0`。在 u-boot-spl 中其地址是 `0`，在 u-boot 中，由于经过了 relocate 之后，代码的运行地址是我们定义的基地址（也就是重定位后的偏移地址 `gd->relocaddr`）

```c
; arch/arm/cpu/armv7/start.S
.globl _start   				/* .globl指示告诉汇编器，_start是一个全局符号*/
_start: b	reset				/* 跳转到reset符号处*/
	ldr	pc, _undefined_instruction	/* 未定义指令异常*/
	ldr	pc, _software_interrupt		/* 软中断，Linux系统调用*/
	ldr	pc, _prefetch_abort		/* 预取址中止，取不到下一条指令*/
	ldr	pc, _data_abort
	ldr	pc, _not_used
	ldr	pc, _irq			/* 中断*/
	ldr	pc, _fiq			/* 快中断*/
...
reset:
        bl	save_boot_params       //跳转到save_boot_params符号处
	/*
	 * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
	 * except if in HYP mode already
	 */
        mrs		r0, cpsr			//将cpsr寄存器的值读到r0中
        and		r1, r0, #0x1f		 @ mask mode bits
        teq		 r1,      #0x1a		    @ test for HYP mode
        bicne	 r0, r0, #0x1f		   @ clear all mode bits
        orrne	 r0, r0, #0x13		  @ set SVC mode 			//更改处理器模式为SVC模式
        orr		 r0, r0, #0xc0		  @ disable FIQ and IRQ	//将I、F位（7、6位）置1 也即0xc0，关闭快中断和中断
        msr	cpsr,r0				      //将r0的内容写入cpsr寄存器


ENTRY(save_boot_params)
	bx	lr			@ back to my caller
ENDPROC(save_boot_params)
	.weak	save_boot_params
```

这一段，首先会跳转到 **save_boot_params** 标号处，而 `ENTRY(save_boot_params)` 表示 `save_boot_params` 入口，进入后就一句 `bx lr`，其作用为 跳转到 `lr` 中存放的地址处。而 `lr` 的用途：当通过 BL 或者 BLX 调用 **子程序** 时，硬件将自动将子程序返回地址保存在 R14 寄存器中，在子程序返回时，把 LR 的值复制到程序计数器就可以实现子程序返回。所以这段代码的意思就是跳转到调用者处。也就是接着执行 `bl save_boot_params` 后面的代码。

`mrs r0, cpsr` 读取寄存器 cpsr 中的值，并保存到 `r0` 寄存器中。

`and r1, r0, #0x1f` 寄存器 `r0` 中的值与 `0X1F` 进行与运算，结果保存到 `r1` 寄存器中，目的就是提取 `cpsr` 的 bit0~bit4 这 5 位，这 5 位为 M4 M3 M2 M1 M0，M[4:0] 这五位用来设置处理器的工作模式。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204101725592.png)

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204101726017.png)

如果 `r1` 和 `0X1A(0b11010)` 不相等，也就是 CPU 不处于 Hyp 模式的话就将 `r0` 寄存器的 bit0~5 进行清零，其实就是清除模式位。

如果处理器不处于 Hyp 模式的话就将 r0 的寄存器的值与 `0x13` 进行或运算，`0x13(0b10011)`，也就是设置处理器进入管理模式（SVC）。

`r0` 寄存器的值再与 `0xC0(0b1100_0000)` 进行或运算，那么 r0 寄存器此时的值就是 `0xD3(0b1101_0000)`，cpsr 的 I 位和 F 位分别控制 IRQ 和 FIQ 这两个中断的开关，设置为 1 就关闭了 FIQ 和 IRQ！

将 `r0` 寄存器写回到 `cpsr` 寄存器中。完成设置 CPU 处于 SVC32 模式，并且关闭 FIQ 和 IRQ 这两个中断。

```c
; arch/arm/cpu/armv7/start.S
/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	/* Set V=0 in CP15 SCTRL register - for VBAR to point to vector */
	mrc	p15, 0, r0, c1, c0, 0	@ Read CP15 SCTRL Register
	bic	r0, #CR_V		@ V = 0
	mcr	p15, 0, r0, c1, c0, 0	@ Write CP15 SCTRL Register

	/* Set vector address in CP15 VBAR register */
	ldr	r0, =_start
	mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
#endif
```

这部分代码通过对协处理器 CP15 进行操作，设置了处理器的异常向量入口地址为 `_start`。

这是因为 ARM 默认的异常向量表入口在 0x0 地址，然而 S5PV210 中 0x0 地址存放的是 IROM，不可修改，自然不可能存放异常向量表，所以需要修改异常向量表入口，将它们映射到其他位置上去。

接着是三个跳转，每个跳转后都会返回到当前位置的下一条地址处。

```c
; arch/arm/cpu/armv7/start.S
	/* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif

	bl	_main
```

#### cpu_init_cp15

首先跳转到 **cpu_init_cp15** 中，我们可以看到在 `ENDPROC(cpu_init_cp15)` 之前，有一条 `mov pc, lr`，此命令作用和 `bx lr` 相同，跳转回子函数调用的地方。在 `cpu_init_cp15` 中，主要是失效 L1、I/D、关闭 MMU 等

```c
; arch/arm/cpu/armv7/start.S
ENTRY(cpu_init_cp15)
	...
	mov	pc, lr			@ back to my caller //子过程运行结束，跳转回去
ENDPROC(cpu_init_cp15)
```

#### cpu_init_crit

接着跳转到 **cpu_init_crit** 中，里面又一个跳转 `lowlevel_init`

```c
; arch/arm/cpu/armv7/start.S
ENTRY(cpu_init_crit)
	b	lowlevel_init		@ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
```

---

#### :x:~~lowlevel_init(分析错误)~~

**注意：以下分析错误，请直接看下一个 lowlevel_init**

~~**lowlevel_init** 位于 `arch/arm/cpu/armv7/lowlevel_init.S`~~，主要是设置栈顶指针为 CONFIG_SYS_INIT_SP_ADDR，然后栈顶指针 8 字节对齐。如果在 SPL，设置 r9 数据为 gdata 地址，否则为 GD 数据分配空间，并将分配后的栈顶指针给 r9。

```c
ENTRY(lowlevel_init)
	/*
	 * Setup a temporary stack
	 */
	ldr	sp, =CONFIG_SYS_INIT_SP_ADDR
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
#ifdef CONFIG_SPL_BUILD
	ldr	r9, =gdata
#else
	sub	sp, sp, #GD_SIZE
	bic	sp, sp, #7
	mov	r9, sp
#endif
	/*
	 * Save the old lr(passed in ip) and the current lr to stack
	 */
	push	{ip, lr}

	/*
	 * go setup pll, mux, memory
	 */
	bl	s_init
	pop	{ip, pc}
ENDPROC(lowlevel_init)
```

**CONFIG_SYS_INIT_SP_ADDR** 在 `include/configs/smart210.h`，也就是说设置栈顶指针为 SDRAM 结束的高地址处。

```c
#define CONFIG_SYS_INIT_SP_ADDR	(CONFIG_SYS_LOAD_ADDR + PHYS_SDRAM_1_SIZE)
...
#define CONFIG_SYS_LOAD_ADDR		CONFIG_SYS_SDRAM_BASE
...
/* DRAM Base */
#define CONFIG_SYS_SDRAM_BASE		0x20000000
...
#define PHYS_SDRAM_1_SIZE	(512 << 20)				/* 0x2000_0000, 512 MB Bank #1 */
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204101814781.png)

其中 gddata 定义在 `arch/arm/lib/spl.c` 中，这里需要补充一个高级用法。具体用法查看 https://gcc.gnu.org/onlinedocs/gcc/Variable-Attributes.html，这里则是将 gdata 位置存放到 .data 段中。

> `__attribute__`
>
> 该关键字允许您指定变量、函数参数或结构体、联合体以及类成员的特殊属性
>
> 而 `__attribute__ ((section(".data")))`
>
> 则说明将 前者定义的放入 `.data` 段中

```c
/* Pointer to as well as the global data structure for SPL */
DECLARE_GLOBAL_DATA_PTR;
gd_t gdata __attribute__ ((section(".data")));
...
// 在 arch/arm/include/asm/global_data.h 中
#define DECLARE_GLOBAL_DATA_PTR		register volatile gd_t *gd asm ("r9")
```

此时需要注意下 DECLARE_GLOBAL_DATA_PTR 这个宏，这个宏的意思是定义了全局数据结构 gd（**寄存器 r9 来表示全局数据结构 gd**）

`register volatile gd_t *gd asm ("r9")` 其中 register 关键字是必须的，asm ("r9") 为嵌入式汇编，表示用 r9 寄存器存储 gd 指针，r9 和 CPU 体系结构相关。寄存器变量。

所以不管 `ldr r9, =gdata` 还是 `mov r9, sp` 都是更新 gd 的地址，不同的是在 SPL 下，gd 结构体空间被分配在 `.data` 段中，在 u-boot 中，gd 结构体空间被分配在 `CONFIG_SYS_INIT_SP_ADDR - GD_SIZE`（需要 8 字节对齐）处。

接着往下走，`push {ip, lr}` 用 stack 的形式保存 `lr`(返回指针)，将 ip 和 lr 入栈

然后一个跳转到 `bl s_init` 中。

下面 `pop {ip, pc}` 恢复到 `ip`、`pc`，意思是改变 pc 的指向，将 lr 内容恢复到 pc 中，也即返回到 lr 指向的地方，在此处应为 `bl cpu_init_crit` 的下方，也即 `bl _main`。

接着分析 `s_init`，在与 smart210 相关的任何地方找不到与 s_init 相关的定义，也找不到与 s5pv210 任何相关的，所以此处应该是有问题的，通过在指令间添加一些异常数据，编译，编译通过，得知在调用 lowlevel_init 时并不是找的此处。

**以上分析错误，请直接查看下一个 lowlevel_init**

---

#### lowlevel_init

**lowlevel_init** 位于 `board/samsung/smart210/lowlevel_init.S`，这个相对简单，但还是说一下，首先将 `lr` 返回地址保存到 `r9` 中（此处不应该使用 r9，或者说在使用 r9 前先保存 r9 的值），如果在 SPL 下，则需要初始化时钟和 ddr，并配置串口寄存器 `PA0CON` 值为 `0x00002222`（查看手册 `S5PV210_UM_REV1.1.pdf` 中 `2.2.2.1 Port Group GPA0 Control Register (GPA0CON, R/W, Address = 0xE020_0000)`），也即配置 `GPA0CON` 功能为串口 UART1。之后，恢复 `pc` 值为 `lr`，返回到调用 `lowlevel_init` 的地方。

```c
; board/samsung/smart210/lowlevel_init.S

.globl lowlevel_init
lowlevel_init:
	mov	r9, lr

#ifdef CONFIG_SPL_BUILD
	bl clock_init                   /* clock init */
	bl ddr_init                     /* DDR init */

	/* add by Flinn, for uart */
	ldr r0, =0xE0200000     		/* GPA0_CON */
	ldr r1, =0x22222222
	str r1, [r0]

#endif
	mov pc, r9                		/* return */
```

返回到 `bl cpu_init_crit` 的下一句指令，也即执行 `bl _main`。

```c
; arch/arm/cpu/armv7/start.S

	/* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif
	bl	_main
```

#### \_main

**\_main** 位于 `arch/arm/lib/crt0.S` 中，具体代码一会回来分析。

终于来到 `_main` 中了，首先如果定义了 `CONFIG_SPL_BUILD` 和 `CONFIG_SPL_STACK`，也即 SPL 时，sp 值为 `CONFIG_SPL_STACK`，但经过验证没有走这个分支；所以 sp 的值只可能为 `CONFIG_SYS_INIT_SP_ADDR`。

```c
; arch/arm/lib/crt0.S
/*
 * entry point of crt0 sequence
 */

ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
	ldr	sp, =(CONFIG_SPL_STACK)
#else
	ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
	...
ENDPROC(_main)
```

**CONFIG_SYS_INIT_SP_ADDR** 在 `include/configs/smart210.h`，也就是说设置栈顶指针为 SDRAM 结束的高地址处。

```c
/* include/configs/smart210.h */
#define CONFIG_SYS_INIT_SP_ADDR		   (CONFIG_SYS_LOAD_ADDR + PHYS_SDRAM_1_SIZE)
...
#define CONFIG_SYS_LOAD_ADDR		CONFIG_SYS_SDRAM_BASE
...
/* DRAM Base */
#define CONFIG_SYS_SDRAM_BASE		0x20000000
...
#define PHYS_SDRAM_1_SIZE			 (512 << 20)				/* 0x2000_0000, 512 MB Bank #1 */
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204101814781.png)

接着往下走，是给 `gd` 数据结构分配空间。gd 是什么？

```c
// board/samsung/smart210/smart210.c
DECLARE_GLOBAL_DATA_PTR;
...
// arch/arm/include/asm/global_data.h
#define DECLARE_GLOBAL_DATA_PTR		register volatile gd_t *gd asm ("r9")
```

> gd 是一个寄存器变量。此时需要注意下 DECLARE_GLOBAL_DATA_PTR 这个宏，这个宏的意思是定义了全局数据结构 gd（**寄存器 r9 来表示全局数据结构 gd**）
> `register volatile gd_t *gd asm ("r9")` 其中 `register` 关键字是必须的，`asm ("r9")` 为嵌入式汇编，表示用 `r9` 寄存器存储 `gd` 指针，`r9` 和 CPU 体系结构相关。

先 8 字节对齐，然后给 `gd` 结构分配大小为 `GD_SIZE` 的空间，然后继续 8 字节对齐。`GD_SIZE` 是一个宏，在 `include/generated/generic-asm-offsets.h` 中定义为 `160`（`sizeof(struct global_data)`），该文件也是生成的。

更新 `r9` 值为 `sp`，此时 `r9 = sp = [CONFIG_SYS_INIT_SP_ADDR - GD_SIZE]`（`[CONFIG_SYS_INIT_SP_ADDR - GD_SIZE]` 为对应地址处的值），接着给 `r0` 清零

```c
; arch/arm/lib/crt0.S
	bic	sp, sp, #7			/* 8-byte alignment for ABI compliance */
	sub	sp, sp, #GD_SIZE	/* allocate one GD above SP */
	bic	sp, sp, #7			/* 8-byte alignment for ABI compliance */
	mov	r9, sp				/* GD is above SP */
	mov	r0, #0
```

接着往下走，分为两种情况，第一种 SPL 下，跳转到 `copy_bl2_to_ram`（完成 BL2 阶段镜像校验并拷贝到 SDRAM 中），执行结束后返回，更改 PC 指针为 `CONFIG_SYS_SDRAM_BASE`，也即跳转到 BL2 阶段（SDRAM 中）。

```c
; arch/arm/lib/crt0.S
#ifdef CONFIG_SPL_BUILD
        bl       copy_bl2_to_ram
        ldr pc,=CONFIG_SYS_SDRAM_BASE
#else
        bl       board_init_f
#endif
-----------------------------------------------
; include/configs/smart210.h
#define CONFIG_SYS_SDRAM_BASE		0x20000000
```

#### copy_bl2_to_ram

`copy_bl2_to_ram` 函数在 `board/samsung/smart210/smart210.c` 中

首先是几个宏定义，让我们看一下是什么东东？

```c
/* board/samsung/smart210/smart210.c */
/*
 * 从SD/MMC拷贝（加载）块到内存中的函数
** ch:  通道号
** sb:  起始块号
** bs:  块数量
** dst: 要拷贝到内存的什么位置上
** i:   是否需要初始化
*/
#define CopySDMMCtoMem(ch, sb, bs, dst, i) \
        (((u8(*)(int, u32, unsigned short, u32*, u8))\
        (*((u32 *)0xD0037F98)))(ch, sb, bs, dst, i))

#define MP0_1CON  (*(volatile u32 *)0xE02002E0)
#define MP0_3CON  (*(volatile u32 *)0xE0200320)
#define MP0_6CON  (*(volatile u32 *)0xE0200380)

#define NF8_ReadPage_Adv(a,b,c) (((int(*)(u32, u32, u8*))(*((u32 *)0xD0037F90)))(a,b,c))
```

```c
#define CopySDMMCtoMem(ch, sb, bs, dst, i) \
(((u8(*)(int, u32, unsigned short, u32*, u8))(*((u32 *)0xD0037F98)))(ch, sb, bs, dst, i))
```

这个宏定义分为三段来看，分别对应着 **函数返回值数据类型（\* 指针变量名）（函数的实际参数或者函数参数的类型）**

第一段：`((u8 (*)(int, u32, unsigned short, u32*, u8))` 是一个函数类型强制类型转换，其中 `u8` 为返回值，`(int, u32, unsigned short, u32*, u8)` 为传入参数类型。

第二段：`(*((u32 *)0xD0037F98)))`，在地址 `0xD0037F98` 中存放了一个名字叫 `CopySDMMCtoMem` 的函数，armv7 为 32 位 cpu，所以先转成 **指针类型为 u32\***，然后再把这个地址 **解引用**，就得到了地址中存在的值也就是 CopySDMMCtoMem

第三段：将函数的实际参数传入即可 `(ch, sb, bs, dst, i)`，最外层加上一个大括号。

接着三个宏表示 将对 **对应地址处的值** 操作。

```c
#define MP0_1CON  (*(volatile u32 *)0xE02002E0) /* 查看手册S5PV210_UM_REV1.1.pdf中 2.2.25.1*/
#define MP0_3CON  (*(volatile u32 *)0xE0200320) /* 查看手册S5PV210_UM_REV1.1.pdf中 2.2.27.1*/
#define MP0_6CON  (*(volatile u32 *)0xE0200380) /* 查看手册S5PV210_UM_REV1.1.pdf中 2.2.30.1*/
```

`NF8_ReadPage_Adv(a,b,c)` 与 `CopySDMMCtoMem(ch, sb, bs, dst, i)` 一致，不在讲解。

| 宏                                 | 作用                                                                                                                            |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| CopySDMMCtoMem(ch, sb, bs, dst, i) | 从 SD/MMC 拷贝（加载）块到内存中的函数（从通道 ch 的 sb 块处，加载 bs 块内存到以 dst 起始的地方）|
| NF8_ReadPage_Adv(a,b,c)            | 从 nand flash 拷贝（加载）页到内存中的函数（a：要复制的源块地址号，b：要复制的源页地址号，c：目标缓冲区指针，返回值成功或失败）|

接着就进入到 `copy_bl2_to_ram`，整个 `copy_bl2_to_ram` 框架如下。也就是上来先定义 bl2 的大小为 250k（编译结束后生成 u-boot.bin 大小为 240k），所以此处最好大小调大一些（建议 512k）。OM 为寄存器 `0xE0000004` 中存储的值，通过手册（`SIAP` 中 `3 Boot configuration`）可知 OM 低 5 位控制着从哪里启动，所以下面两个分支也就是选择从哪边启动。0x2（0b0010）也即 `Nand 2KB,5cycle X-TAL`；0xc（0b1100）也即 `SD/MMC X-TAL`。

```c
/* board/samsung/smart210/smart210.c */
void copy_bl2_to_ram(void)
{
    u32 bl2Size = 250 * 1024;       		  // 250K
    u32 OM = *(volatile u32 *)(0xE0000004); // OM Register
    OM &= 0x1F;                             	       // 取出低5位OM[4:0]
    if (OM == 0x2) {				       // Nand 2KB,5cycle X-TAL
        ...
    } else if (OM == 0xC) {			           // SD/MMC X-TAL
        ...
    }
}
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204112027872.png)

先分析 SD/MMC 下的代码（SD 卡下的代码），首先取出当前 boot 的 `V210_SDMMC_BASE` 基地址（查看手册 `SIAP` 中 `2.6 Global Variable`），接着看手册 `S5PV210_UM_REV1.1.pdf` 中的 `7.9.1 REGISTER MAP` 找出对应地址的 channel，然后从对应 channel 的 SD 卡中将以 `32块开始的250k数据` 拷贝到 SDRAM 中的 `CONFIG_SYS_SDRAM_BASE`（`0x20000000`）中去。

```c
/* board/samsung/smart210/smart210.c */
	else if (OM == 0xC) {
	    u32 V210_SDMMC_BASE = *(volatile u32 *)(0xD0037488);    // V210_SDMMC_BASE
            u8 ch = 0;

            /* 7.9.1 SD/MMC REGISTER MAP */
            if (V210_SDMMC_BASE == 0xEB000000)
                    ch = 0;
            else if (V210_SDMMC_BASE == 0xEB200000)
                    ch = 2;
            // 将BL2 从SD卡（32块开始的250k区域数据）拷贝到SDRAM中
            CopySDMMCtoMem(ch, 32, bl2Size / 512, (u32 *)CONFIG_SYS_SDRAM_BASE, 0);
        }
```

接着分析如果 u-boot 在 nand 中是怎样将 BL2 拷贝到内存中的？

首先初始化配置，将配置写入到 `nand_reg->nfconf` 和 `nand_reg->nfcont` 中，接着配置 GPIO。`pages = bl2Size/2048`，看一下 BL2 一共有多少页。下一个 `offset = 0x4000/2048`，意思是 BL2 在 nand 的页偏移。`0x4000/512 = 32blocks`，也就是在 nand 中第 32 个 block 开始，后 256k

> `void writel(unsigned char data, unsigned short addr)` 往内存映射的 IO 空间上写数据

```c
/* board/samsung/smart210/smart210.c */
	if (OM == 0x2) {
		u32 cfg = 0;
		struct s5pv210_nand *nand_reg = (struct s5pv210_nand *)(struct s5pv210_nand *)samsung_get_base_nand();

		/* initialize hardware */
		/* HCLK_PSYS=133MHz(7.5ns) */
		cfg =   (0x1 << 23) |   /* Disable 1-bit and 4-bit ECC */
				   (0x3 << 12) |   /* 7.5ns * 2 > 12ns tALS tCLS */
				   (0x2 << 8) |    /* (1+1) * 7.5ns > 12ns (tWP) */
				   (0x1 << 4) |    /* (0+1) * 7.5 > 5ns (tCLH/tALH) */
				   (0x0 << 3) |    /* SLC NAND Flash */
				   (0x0 << 2) |    /* 2KBytes/Page */
				   (0x1 << 1);     /* 5 address cycle */

		writel(cfg, &nand_reg->nfconf);

		writel((0x1 << 1) | (0x1 << 0), &nand_reg->nfcont);
		/* Disable chip select and Enable NAND Flash Controller */

		/* Config GPIO */
		MP0_1CON &= ~(0xFFFF << 8);
		MP0_1CON |= (0x3333 << 8);
		MP0_3CON = 0x22222222;
		MP0_6CON = 0x22222222;

		int i = 0;
		int pages = bl2Size / 2048;             //
		int offset = 0x4000 / 2048;             // u-boot.bin
		u8 *p = (u8 *)CONFIG_SYS_SDRAM_BASE;
		for (; i < pages; i++, p += 2048, offset += 1)
			NF8_ReadPage_Adv(offset / 64, offset % 64, p);
	}
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204112111124.png)

#### board_init_f

`board_init_f` 在 `arch/arm/lib/board.c` 中，天哪，这是个什？这么多代码。

一点点分析吧，首先定义一堆变量，指针等。将 `gd`（`global data`）中清零，什么，gd 又是个什？gd 是在什么时候赋值？什么时候分配空间的呢？

在 [\_main](#_main) 中通过 `sub sp,sp,#GD_SIZE` 给 gd 分配了空间，后面通过 `mov r9，sp` 更新 gd 指针，使之指向分配的空间。所以此处直接清零没有任何问题。

##### global_data

> 在某些情况下，uboot 运行在某些只读存储器上，在 uboot 被重定向到 RAM（可读可写）之前，我们都无法写入数据，更无法通过全局变量来传递数据。而 global_data 则是为了解决这个问题。

```c
/* include/asm-generic/global_data.h */
typedef struct global_data {
	bd_t *bd;
	unsigned long flags;
	unsigned int baudrate;
	unsigned long cpu_clk;	/* CPU clock in Hz!		*/
	unsigned long bus_clk;
	/* We cannot bracket this with CONFIG_PCI due to mpc5xxx */
	unsigned long pci_clk;
	unsigned long mem_clk;
#if defined(CONFIG_LCD) || defined(CONFIG_VIDEO)
	unsigned long fb_base;	/* Base address of framebuffer mem */
#endif
#if defined(CONFIG_POST) || defined(CONFIG_LOGBUFFER)
	unsigned long post_log_word;  /* Record POST activities */
	unsigned long post_log_res; /* success of POST test */
	unsigned long post_init_f_time;  /* When post_init_f started */
#endif
#ifdef CONFIG_BOARD_TYPES
	unsigned long board_type;
#endif
	unsigned long have_console;	/* serial_init() was called */
#ifdef CONFIG_PRE_CONSOLE_BUFFER
	unsigned long precon_buf_idx;	/* Pre-Console buffer index */
#endif
#ifdef CONFIG_MODEM_SUPPORT
	unsigned long do_mdm_init;
	unsigned long be_quiet;
#endif
	unsigned long env_addr;	/* Address  of Environment struct */
	unsigned long env_valid;	/* Checksum of Environment valid? */

	unsigned long ram_top;	/* Top address of RAM used by U-Boot */

	unsigned long relocaddr;	/* Start address of U-Boot in RAM */
	phys_size_t ram_size;	/* RAM size */
	unsigned long mon_len;	/* monitor len */
	unsigned long irq_sp;		/* irq stack pointer */
	unsigned long start_addr_sp;	/* start_addr_stackpointer */
	unsigned long reloc_off;
	struct global_data *new_gd;	/* relocated global data */

#ifdef CONFIG_DM
	struct device	*dm_root;	/* Root instance for Driver Model */
	struct list_head uclass_root;	/* Head of core tree */
#endif

	const void *fdt_blob;	/* Our device tree, NULL if none */
	void *new_fdt;		/* Relocated FDT */
	unsigned long fdt_size;	/* Space reserved for relocated FDT */
	void **jt;		/* jump table */
	char env_buf[32];	/* buffer for getenv() before reloc. */
#ifdef CONFIG_TRACE
	void		*trace_buff;	/* The trace buffer */
#endif
#if defined(CONFIG_SYS_I2C)
	int		cur_i2c_bus;	/* current used i2c bus */
#endif
	unsigned long timebase_h;
	unsigned long timebase_l;
	struct arch_global_data arch;	/* architecture-specific data */
} gd_t;
```

重点说明

- `bd_t *bd`：board info 数据结构定义,位于文件 include/asm-arm/u-boot.h 定义,主要是保存开发板的相关参数。
- `unsigned long env_addr`：环境变量的地址。
- `unsigned long ram_top`：RAM 空间的顶端地址
- `unsigned long relocaddr`：UBOOT 重定向后地址
- `phys_size_t ram_size`：物理 ram 的 size
- `unsigned long irq_sp`：中断的堆栈地址
- `unsigned long start_addr_sp`：堆栈地址
- `unsigned long reloc_off`：uboot 的 relocation 的偏移
- `struct global_data *new_gd`：重定向后的 struct global_data 结构体
- `const void *fdt_blob`：设备的 dtb 地址
- `void *new_fdt`：relocation 之后的 dtb 地址
- `unsigned long fdt_size`：dtb 的长度
- `struct udevice *cur_serial_dev`：当前使用的串口设备。

计算监视区（monitor）长度（`__bss_end` - `_start`）并给 `gd->mon_len`

从环境变量中查找 `fdtcontroladdr`，如果存在则更新 `gd->fdt_blob`

> `ulong getenv_ulong(const char *name, int base, ulong default_val)` 从环境变量中查找 name，存在返回 name 对应的数值，不存在返回 default_val。其中 base 为进制，例如 10、16 等。
>
> 环境变量的存储用 hash 表实现

这一段的意思遍历 `init_sequence`，`init_sequence` 存放的是一系列的函数首地址，通过循环不断将函数地址取出 `解引用` 并调用。如果函数返回不为 0，表示有错误则一直在此循环。

```c
/* arch/arm/lib/board.c */
void board_init_f(ulong bootflag){
	...
	for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
		if ((*init_fnc_ptr)() != 0) {
			hang ();
		}
	}
    ...
 }

...

init_fnc_t *init_sequence[] = {
	arch_cpu_init,		/* basic arch cpu dependent setup */
	mark_bootstage,
#ifdef CONFIG_OF_CONTROL
	fdtdec_check_fdt,
#endif
#if defined(CONFIG_BOARD_EARLY_INIT_F)
	board_early_init_f,
#endif
	timer_init,		/* initialize timer */
#ifdef CONFIG_BOARD_POSTCLK_INIT
	board_postclk_init,
#endif
#ifdef CONFIG_FSL_ESDHC
	get_clocks,
#endif
	env_init,			/* initialize environment */
	init_baudrate,		/* initialze baudrate settings */
	serial_init,		  /* serial communications setup */
	console_init_f,		  /* stage 1 init of console */
	display_banner,		/* say that we are here */
	print_cpuinfo,		/* display cpu info (and speed) */
#if defined(CONFIG_DISPLAY_BOARDINFO)
	checkboard,		/* display board info */
#endif
#if defined(CONFIG_HARD_I2C) || defined(CONFIG_SYS_I2C)
	init_func_i2c,
#endif
	dram_init,		/* configure available RAM banks */
	NULL,
};
```

让我们一个个的看看这些函数都是干什么的。

##### arch_cpu_init

这个函数做的是针对特定 CPU 的初始化，u-boot 支持很多 CPU，不同 CPU 的初始化也不尽相同，因此 u-boot 提供了 arch_cpu_init 用于 CPU 初始化。这个函数由移植者根据自己的硬件（CPU）的情况来提供，如果不提供，则默认的是一个仅返回 0 的函数。

```c
/* arch/arm/lib/board.c */
int arch_cpu_init(void) __attribute__((weak, alias("__arch_cpu_init")));

int __arch_cpu_init(void)
{
	return 0;
}
```

##### mark_bootstage

其中 `bootstage_mark_name（id,name）` 用于标记当前运行的 id 和名字。这个函数也就是用来标记当前运行的函数为 `board_init_f`

```c
/* arch/arm/lib/board.c */
static int mark_bootstage(void)
{
	bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_F, "board_init_f");
	return 0;
}
```

##### fdtdec_check_fdt

由于未配置 `CONFIG_OF_CONTROL`，所以不会检查 fdt。

这个函数的功能是，在 控制台未准备好之前必须有一个 FDT，否则会 panic。

```c
/* lib/fdtdec.c */
int fdtdec_check_fdt(void)
{
	/*
	 * We must have an FDT, but we cannot panic() yet since the console
	 * is not ready. So for now, just assert(). Boards which need an early
	 * FDT (prior to console ready) will need to make their own
	 * arrangements and do their own checks.
	 */
	assert(!fdtdec_prepare_fdt());
	return 0;
}
```

##### board_early_init_f

由于未配置 `CONFIG_BOARD_EARLY_INIT_F`，不会执行

##### timer_init

这个配置貌似跟 cpu 有关系，在每个不同的 cpu 有不同的 `timer_init`，另外在 `lib/time.c` 中也有一个弱空实现。

s5pv210 芯片的在 `arch/arm/cpu/armv7/s5p-common/time.c` 中。

使用 SOC 的 `Pwm Timer4` 作为定时器，Timer4 没有输出引脚，不耽误其 Pwm 功能。设置 Pwm Timer4 4 分频，设置占空比为 100000ns、周期为 100000ns,开启定时器 4。

接着将 `gd->arch.timer_reset_value` 复位。更新 `gd->arch.lastinc`，复位 `gd->arch.lastinc = readl(&timer->tcnto4);`，`gd->arch.tbl = 0;`

```c
// arch/arm/cpu/armv7/s5p-common/time.c
int timer_init(void)
{
	/* PWM Timer 4 */
	pwm_init(4, MUX_DIV_4, 0);
	pwm_config(4, 100000, 100000);
	pwm_enable(4);

	/* Use this as the current monotonic time in us */
	gd->arch.timer_reset_value = 0;

	/* Use this as the last timer value we saw */
	gd->arch.lastinc = timer_get_us_down();
	reset_timer_masked();
	return 0;
}
```

##### board_postclk_init

由于未配置 `CONFIG_BOARD_POSTCLK_INIT`，不会执行

##### get_clocks

由于未配置 `CONFIG_FSL_ESDHC`，不会执行

##### env_init

通过 SI 在 Symbols 中搜索发现有多个 `env_init`,猜测是不同的板子可能存到不同的存储设备上。

> 1. 环境变量可能位于很多地方，比如 EEPROM、FLASH 等地方，枚举 `enum env_location` 定义了一些常量用于标识环境变量的位置。这些常量有 ENVL_EEPROM、ENVL_EXT4 等
>
> 2. uboot 获取环境变量的位置时，有一个先后顺序，比如优先看环境变量是否在 EEPROM 中。这个优先顺序实现很简单：定义一个数组 `env_locations`，**数组里存的是环境变量各个的位置，存的顺序就是优先顺序，**然后只要从下标为 0 开始访问这个数组，即可按照数组指定的优先顺序来获取环境变量的位置。
>
> 3. 对于存在于不同位置的环境变量，u-boot 会使用 `U_BOOT_ENV_LOCATION` 定义一个相应的 `struct env_driver` 类型的 entry，多个 entry 在内存中连续分布，因此可以向遍历数组元素那样遍历这些 entry。而实现内存连续分布的关键在于\*\* 自定义段，
>
>    ```c
>    /*
>    	在链接脚本中将这些段按照段名的字母顺序排列:
>    	.u_boot_list : {
>      		KEEP(*(SORT(.u_boot_list*)));
>     	}
>    	因此可用u_boot_list_2_env_driver_1标记这些entry的起始地址
>    	而使用u_boot_list_2_env_driver_3标记相应的结束地址
>    */
>    .u_boot_list_2_env_driver_1
>    .u_boot_list_2_env_driver_2_eeprom
>    .u_boot_list_2_env_driver_2_ext4
>    .u_boot_list_2_env_driver_3
>    ```
>

smart210 这个板子在执行 `saveenv` 是直接存储环境变量到 `NAND` 的 0x60000 处。实际上通过编译脚本来说更有说服力，首先，看下 `common/Makefile`，要编译对应的文件，需要使能相关宏后才会编译。再次查看 `include/configs/smart210.h`，其定义了 `#define CONFIG_ENV_IS_IN_NAND 1`，所以 env_init 的路径为 `common/env_nand.c`

```c
# environment
obj-y += env_attr.o
obj-y += env_callback.o
obj-y += env_flags.o
obj-$(CONFIG_ENV_IS_IN_DATAFLASH) += env_dataflash.o
obj-$(CONFIG_ENV_IS_IN_EEPROM) += env_eeprom.o
extra-$(CONFIG_ENV_IS_EMBEDDED) += env_embedded.o
obj-$(CONFIG_ENV_IS_IN_EEPROM) += env_embedded.o
extra-$(CONFIG_ENV_IS_IN_FLASH) += env_embedded.o
obj-$(CONFIG_ENV_IS_IN_NVRAM) += env_embedded.o
obj-$(CONFIG_ENV_IS_IN_FLASH) += env_flash.o
obj-$(CONFIG_ENV_IS_IN_MMC) += env_mmc.o
obj-$(CONFIG_ENV_IS_IN_FAT) += env_fat.o
obj-$(CONFIG_ENV_IS_IN_NAND) += env_nand.o
obj-$(CONFIG_ENV_IS_IN_NVRAM) += env_nvram.o
obj-$(CONFIG_ENV_IS_IN_ONENAND) += env_onenand.o
obj-$(CONFIG_ENV_IS_IN_SPI_FLASH) += env_sf.o
obj-$(CONFIG_ENV_IS_IN_REMOTE) += env_remote.o
obj-$(CONFIG_ENV_IS_IN_UBI) += env_ubi.o
obj-$(CONFIG_ENV_IS_NOWHERE) += env_nowhere.o
```

所以我们只关心 `common/env_nand.c` 中的，别看代码这么长。实际上 `ENV_IS_EMBEDDED` 与 `CONFIG_NAND_ENV_DST` 我们都没有定义，所以真正的代码只有两句。

从注释就基本可以看出这个函数的作用，因为 `env_init` 要早于静态存储器的初始化，所以无法进行 `env` 的读写，这里将 `gd` 中的 `env` 相关变量进行配置，默认设置 `env` 为 `valid`。`default_environment` 是什么？**TODO: 待分析**

```c
gd->env_addr	= (ulong)&default_environment[0];
gd->env_valid	  = 1;
```

```c
// common/env_nand.c
/*
 * This is called before nand_init() so we can't read NAND to
 * validate env data.
 *
 * Mark it OK for now. env_relocate() in env_common.c will call our
 * relocate function which does the real validation.
 *
 * When using a NAND boot image (like sequoia_nand), the environment
 * can be embedded or attached to the U-Boot image in NAND flash.
 * This way the SPL loads not only the U-Boot image from NAND but
 * also the environment.
 */
int env_init(void)
{
#if defined(ENV_IS_EMBEDDED) || defined(CONFIG_NAND_ENV_DST)
	int crc1_ok = 0, crc2_ok = 0;
	env_t *tmp_env1;

#ifdef CONFIG_ENV_OFFSET_REDUND
	env_t *tmp_env2;

	tmp_env2 = (env_t *)((ulong)env_ptr + CONFIG_ENV_SIZE);
	crc2_ok = crc32(0, tmp_env2->data, ENV_SIZE) == tmp_env2->crc;
#endif
	tmp_env1 = env_ptr;
	crc1_ok = crc32(0, tmp_env1->data, ENV_SIZE) == tmp_env1->crc;

	if (!crc1_ok && !crc2_ok) {
		gd->env_addr	= 0;
		gd->env_valid	= 0;

		return 0;
	} else if (crc1_ok && !crc2_ok) {
		gd->env_valid = 1;
	}
#ifdef CONFIG_ENV_OFFSET_REDUND
	else if (!crc1_ok && crc2_ok) {
		gd->env_valid = 2;
	} else {
		/* both ok - check serial */
		if (tmp_env1->flags == 255 && tmp_env2->flags == 0)
			gd->env_valid = 2;
		else if (tmp_env2->flags == 255 && tmp_env1->flags == 0)
			gd->env_valid = 1;
		else if (tmp_env1->flags > tmp_env2->flags)
			gd->env_valid = 1;
		else if (tmp_env2->flags > tmp_env1->flags)
			gd->env_valid = 2;
		else /* flags are equal - almost impossible */
			gd->env_valid = 1;
	}

	if (gd->env_valid == 2)
		env_ptr = tmp_env2;
	else
#endif
	if (gd->env_valid == 1)
		env_ptr = tmp_env1;

	gd->env_addr = (ulong)env_ptr->data;

#else /* ENV_IS_EMBEDDED || CONFIG_NAND_ENV_DST */
	gd->env_addr	= (ulong)&default_environment[0];
	gd->env_valid	= 1;
#endif /* ENV_IS_EMBEDDED || CONFIG_NAND_ENV_DST */

	return 0;
}
```

##### init_baudrate

上面我们介绍了 `getenv_ulong`，从环境变量中获取参数，所以本函数的作用是从环境变量中取出 `baudrate` 对应的值给 `gd`，以 10 进制，如果没有，默认为 `115200`

```c
/* arch/arm/lib/board.c */
static int init_baudrate(void)
{
	gd->baudrate = getenv_ulong("baudrate", 10, CONFIG_BAUDRATE);
	return 0;
}
```

##### serial_init

获取当前的串口设备，并调用当前串口设备的 start 成员函数

```c
/* drivers/serial/serial.c */
int serial_init(void)
{
	return get_current()->start();
}
```

由于此时重定位还没做，GD_FLG_RELOC 标志未设置，因此条件满足，会进入第一个分支，因此 `dev = default_serial_console();` 后面调用 start。

```c
/* drivers/serial/serial.c */
static struct serial_device *get_current(void)
{
	struct serial_device *dev;

	if (!(gd->flags & GD_FLG_RELOC))
		dev = default_serial_console();
	else if (!serial_current)
		dev = default_serial_console();
	else
		dev = serial_current;

	/* We must have a console device */
	if (!dev) {
#ifdef CONFIG_SPL_BUILD
		puts("Cannot find console\n");
		hang();
#else
		panic("Cannot find console\n");
#endif
	}

	return dev;
}
```

后面在重定位完成后，且当前 `serial_current` 也不为空，会进入第三个分支，而第三个分支的 `dev = serial_current;`serial_current 是怎么确认是哪个串口的呢？

`board_init_r` 中会有一个串口初始化 `serial_initialize` 函数，这个函数中前面的都是初始化各种串口，比如我们板卡对应型号的为 `s5p_serial_initialize`，最后一个是设置要使用串口。

```c
// drivers/serial/serial.c
void serial_initialize(void)
{
         ...
	s5p_serial_initialize();
        ...
	serial_assign(default_serial_console()->name);
}
```

进去看下，四个注册函数，对应着我们板卡上的串口 0-3

```c
// drivers/serial/serial_s5p.c
void s5p_serial_initialize(void)
{
	serial_register(&s5p_serial0_device);
	serial_register(&s5p_serial1_device);
	serial_register(&s5p_serial2_device);
	serial_register(&s5p_serial3_device);
}
```

而这个注册函数，则是把当前注册的串口添加到链表的最前面。哦对，所有注册的设备节点都串在一个链表中，该链表的头部，也就是 `最近` 注册过的。

所以上面初始化函数实际上就是将所有串口设备串到同一个链表中 `serial_devices`。那么，如何知道使用哪个串口呢？

```c
// drivers/serial/serial.c
static struct serial_device *serial_devices;
static struct serial_device *serial_current;
...
void serial_register(struct serial_device *dev)
{
	dev->next = serial_devices;
	serial_devices = dev;
}
```

实际上，我们在其定义的结构中也许能看出一些端倪。其结构中有一个 `name` 成员，正是通过这个来确定是哪个串口的。那怎么确定的呢？名字又是从哪里来的呢？

```c
// include/serial.h
struct serial_device {
	/* enough bytes to match alignment of following func pointer */
	char	name[16];

	int		(*start)(void);
	int		(*stop)(void);
	void	(*setbrg)(void);
	int		(*getc)(void);
	int		(*tstc)(void);
	void	(*putc)(const char c);
	void	(*puts)(const char *s);
#if CONFIG_POST & CONFIG_SYS_POST_UART
	void	(*loop)(int);
#endif
	struct serial_device	*next;
};
```

别忘记，我们在 `serial_initialize` 还有另一个函数，`serial_assign(default_serial_console()->name);`，没错，这个就是决定你使用哪个串口的关键。首先我们看下这个函数，之后在去 `default_serial_console()`。代码很少，看着就很喜欢，其意思是什么呢？遍历上面我们初始化时注册的那些串口设备的链表，然后拿出名字来与传进来的名字比较，如果一致，`strcmp` 返回 `0`，条件不成立，继续往下走，`serial_current = s` 就把从链表中获取的要使用串口描述符地址取出来了然后赋值给 `serial_current`，所以后面通过 `get_current` 返回的地址为我们设定的串口描述符的地址。

```c
int serial_assign(const char *name)
{
	struct serial_device *s;

	for (s = serial_devices; s; s = s->next) {
		if (strcmp(s->name, name))
			continue;
		serial_current = s;
		return 0;
	}

	return -EINVAL;
}
```

接着我们看下 `default_serial_console()`，在 serial_s5p 中存在一个弱实现，`CONFIG_OF_CONTROL` 我们没定义，所以选择另一个分支，`config.enable=1`，然后我们在 `smart210.h中` 定义了 `CONFIG_SERIAL0` 为 1，所以此处 `return &s5p_serial0_device;`

```c
// drivers/serial/serial_s5p.c
__weak struct serial_device *default_serial_console(void)
{
#ifdef CONFIG_OF_CONTROL
	int index = 0;

	if ((!config.base_addr) && (fdtdec_decode_console(&index, &config))) {
		debug("Cannot decode default console node\n");
		return NULL;
	}

	switch (config.port_id) {
	case 0:
		return &s5p_serial0_device;
	case 1:
		return &s5p_serial1_device;
	case 2:
		return &s5p_serial2_device;
	case 3:
		return &s5p_serial3_device;
	default:
		debug("Unknown config.port_id: %d", config.port_id);
		break;
	}

	return NULL;
#else
	config.enabled = 1;
#if defined(CONFIG_SERIAL0)
	return &s5p_serial0_device;
#elif defined(CONFIG_SERIAL1)
	return &s5p_serial1_device;
#elif defined(CONFIG_SERIAL2)
	return &s5p_serial2_device;
#elif defined(CONFIG_SERIAL3)
	return &s5p_serial3_device;
#else
#error "CONFIG_SERIAL? missing."
#endif
#endif
}
```

在同文件中，`s5p_serial0_device` 是这样被定义的，也就是上面这三行，实际上定义了一堆函数，并把函数与结构体对应起来

```c
DECLARE_S5P_SERIAL_FUNCTIONS(0);
struct serial_device s5p_serial0_device =
	INIT_S5P_SERIAL_STRUCTURE(0, "s5pser0");
...
#define INIT_S5P_SERIAL_STRUCTURE(port, __name) {  \
	.name	= __name,						\
	.start	= s5p_serial##port##_init,			     \
	.stop	= NULL,							 \
	.setbrg	= s5p_serial##port##_setbrg,		  \
	.getc	= s5p_serial##port##_getc,			\
	.tstc	= s5p_serial##port##_tstc,			  \
	.putc	= s5p_serial##port##_putc,		       \
	.puts	= s5p_serial##port##_puts,		       \
}
#define DECLARE_S5P_SERIAL_FUNCTIONS(port) \
static int s5p_serial##port##_init(void) { return serial_init_dev(port); } \
static void s5p_serial##port##_setbrg(void) { serial_setbrg_dev(port); } \
static int s5p_serial##port##_getc(void) { return serial_getc_dev(port); } \
static int s5p_serial##port##_tstc(void) { return serial_tstc_dev(port); } \
static void s5p_serial##port##_putc(const char c) { serial_putc_dev(c, port); } \
static void s5p_serial##port##_puts(const char *s) { serial_puts_dev(s, port); }

------------------------------------------------------------------------------------------
static int s5p_serial0_init(void) { return serial_init_dev(port); }
static void s5p_serial0_setbrg(void) { serial_setbrg_dev(port); }
static int s5p_serial0_getc(void) { return serial_getc_dev(port); }
static int s5p_serial0_tstc(void) { return serial_tstc_dev(port); }
static void s5p_serial0_putc(const char c) { serial_putc_dev(c, port); }
static void s5p_serial0_puts(const char *s) { serial_puts_dev(s, port); }

struct serial_device s5p_serial0_device = {
        .name	= "s5pser0",			\
	.start	    = s5p_serial0_init		    \
	.stop	  = NULL,				 \
	.setbrg	   = s5p_serial0_setbrg, 	   \
        .getc	  = s5p_serial0_getc,              \
	.tstc	   = s5p_serial0_tstc,		   \
	.putc	 = s5p_serial0_putc,	      \
	.puts	 = s5p_serial0_puts,	      \
 }
```

##### console_init_f

将 `gd` 中 `have_console` 置为 1，然后打印 Pre-Console Buffer 中的数据。

```c
/* common/console.c */
int console_init_f(void)
{
	gd->have_console = 1;
#ifdef CONFIG_SILENT_CONSOLE
	if (getenv("silent") != NULL)
		gd->flags |= GD_FLG_SILENT;
#endif
	print_pre_console_buffer();
	return 0;
}
```

##### display_banner

上面我们已经配置好串口，现在可以输出信息到终端了。

```c
/* arch/arm/lib/board.c */
static int display_banner(void)
{
	printf("\n\n%s\n\n", version_string);
	debug("U-Boot code: %08lX -> %08lX  BSS: -> %08lX\n",(ulong)&_start,(ulong)&__bss_start, (ulong)&__bss_end);
#ifdef CONFIG_MODEM_SUPPORT
	debug("Modem Support enabled\n");
#endif
#ifdef CONFIG_USE_IRQ
	debug("IRQ Stack: %08lx\n", IRQ_STACK_START);
	debug("FIQ Stack: %08lx\n", FIQ_STACK_START);
#endif
	return (0);
}
```

此处就是打印，如果定义了宏 `DEBUG`，将输出更多信息。

```shell

U-Boot 2014.04-g8819fbf-dirty (Apr 12 2022 - 10:19:11) for SMART210

```

##### print_cpuinfo

打印 CPU 信息，所以该函数位置为 `arch/arm/cpu/armv7/s5p-common/cpu_info.c`

输出 CPU 名字、ID、频率信息。如

```shell
CPU:	S5PC110@1000MHz
```

```c
/* arch/arm/cpu/armv7/s5p-common/cpu_info.c */
int print_cpuinfo(void)
{
	char buf[32];

	printf("CPU:\t%s%X@%sMHz\n",
			s5p_get_cpu_name(), s5p_cpu_id,
			strmhz(buf, get_arm_clk()));

	return 0;
}
```

##### checkboard

配置了宏 `CONFIG_DISPLAY_BOARDINFO`,位于 `board/samsung/smart210/smart210.c`

```shell
Board:	SMART210
```

```c
/* board/samsung/smart210/smart210.c */
int checkboard(void) {
    printf("Board:\tSMART210\n");
    return 0;
}
```

##### init_func_i2c

似乎没有配置，暂不分析。

```c
/* board/samsung/smart210/smart210.c */
#if defined(CONFIG_HARD_I2C) || defined(CONFIG_SYS_I2C)
static int init_func_i2c(void)
{
	puts("I2C:   ");
#ifdef CONFIG_SYS_I2C
	i2c_init_all();
#else
	i2c_init(CONFIG_SYS_I2C_SPEED, CONFIG_SYS_I2C_SLAVE);
#endif
	puts("ready\n");
	return (0);
}
#endif
```

##### :negative_squared_cross_mark: dram_init

更新 `gd` 中的内存大小，其值为 ` get_ram_size((long *)PHYS_SDRAM_1, PHYS_SDRAM_1_SIZE)`。`get_ram_size` 又是什么呢？是用来检测内给定范围内的内存是否有效的一个小函数，在当前函数中检测范围为 `[PHYS_SDRAM_1, PHYS_SDRAM_1+PHYS_SDRAM_1_SIZE]`。如果没有问题，返回 `PHYS_SDRAM_1_SIZE`。`TODO:剩下的下次在分析`

```c
// board/samsung/smart210.c
int dram_init(void)
{
	gd->ram_size = get_ram_size((long *)PHYS_SDRAM_1, PHYS_SDRAM_1_SIZE);
	return 0;
}
```

```c
// common/memsize.c
long get_ram_size(long *base, long maxsize)
{
	volatile long *addr;
	long           save[32];
	long           cnt;
	long           val;
	long           size;
	int            i = 0;

	for (cnt = (maxsize / sizeof (long)) >> 1; cnt > 0; cnt >>= 1) {
		addr = base + cnt;	/* pointer arith! */
		sync ();
		save[i++] = *addr;
		sync ();
		*addr = ~cnt;
	}

	addr = base;
	sync ();
	save[i] = *addr;
	sync ();
	*addr = 0;

	sync ();
	if ((val = *addr) != 0) {
		/* Restore the original data before leaving the function.
		 */
		sync ();
		*addr = save[i];
		for (cnt = 1; cnt < maxsize / sizeof(long); cnt <<= 1) {
			addr  = base + cnt;
			sync ();
			*addr = save[--i];
		}
		return (0);
	}

	for (cnt = 1; cnt < maxsize / sizeof (long); cnt <<= 1) {
		addr = base + cnt;	/* pointer arith! */
		val = *addr;
		*addr = save[--i];
		if (val != ~cnt) {
			size = cnt * sizeof (long);
			/* Restore the original data before leaving the function.
			 */
			for (cnt <<= 1; cnt < maxsize / sizeof (long); cnt <<= 1) {
				addr  = base + cnt;
				*addr = save[--i];
			}
			return (size);
		}
	}

	return (maxsize);
}
```

##### 其它

上文我们将那一个初始化列表讲完了。下面接着对 board_init_f 分析。根据打印日志，删除没有用的宏开关，方便我们分析代码。

```shell
U-Boot 2014.04-g8819fbf-dirty (Apr 12 2022 - 10:16:10) for SMART210

U-Boot code: 20000000 -> 2003A2D0  BSS: -> 200706D0
CPU:	S5PC110@1000MHz
Board:	SMART210
monitor len: 000706D0
ramsize: 20000000
TLB table from 3fff0000 to 3fff4000
Top of RAM usable for U-Boot at: 3fff0000
Reserving 449k for U-Boot at: 3ff7f000
Reserving 1280k for malloc() at: 3fe3f000
Reserving 32 Bytes for Board Info at: 3fe3efe0
Reserving 160 Bytes for Global Data at: 3fe3ef40
New Stack Pointer is: 3fe3ef30
RAM Configuration:
Bank #0: 20000000 512 MiB
relocation Offset is: 1ff7f000
WARNING: Caches not enabled
```

接着打印 `gd->mon_len` 和 `gd->ram_size`，然后 `addr=CONFIG_SYS_SDRAM_BASE+get_effective_memsize();`，其中 `CONFIG_SYS_SDRAM_BASE` 为 SDRAM 起始地址，也就是 `0x20000000`，`get_effective_memsize` 在 `common/memsize.c` 有一个弱实现，其余与板级相关的地方没有发现该函数。

所以确认调用为 `common/memsize.c` 中 `get_effective_memsize()`，其实也就是判断定义内存大小是否超过了最大界限，超过则返回最大值，否则 `gd->ram_size`。而 `gd->ram_size` 在 [dram_init](#dram_init) 中得到的大小为 `PHYS_SDRAM_1_SIZE`，所以 addr 此时为 `CONFIG_SYS_SDRAM_BASE+PHYS_SDRAM_1_SIZE`。

然后更新 `gd->arch.tlb_size` 值为 `PGTABLE_SIZE`（`4096*4`，也就是 4k），然后下面一个 64kb 对齐（换句话说 `PGTABLE_SIZE` 最大可设置为 64kb），接着更新 `gd->arch.tlb_addr`。然后 4kb 对齐，`addr -= gd->mon_len`，继续 4kb 对齐。

最终内存分布是这样的。

总结一下，这段代码首先用来 `更新gd结构体` 中的内容，然后将 `gd` 结构体在拷贝到内存中对应位置。

此处需要注意下，当前代码还没有重定位，所以 `_start` 还是在起始位置，也就是 `0x20000000` 处，

`gd->relocaddr = addr`，`gd->reloc_off = addr - (ulong)&_start`，`gd->relocaddr` 存储的为重定位后的地址，所以 `gd->reloc_off` 存储的是重定位前后 `_start` 的绝对值。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204131359579.png)

```c
/* arch/arm/lib/board.c */
void board_init_f(ulong bootflag)
{
	bd_t *bd;
	init_fnc_t **init_fnc_ptr;
	gd_t *id;
	ulong addr, addr_sp;
#ifdef CONFIG_PRAM
	ulong reg;
#endif
	void *new_fdt = NULL;
	size_t fdt_size = 0;

	...
	debug("monitor len: %08lX\n", gd->mon_len);
	debug("ramsize: %08lX\n", gd->ram_size);
	addr = CONFIG_SYS_SDRAM_BASE + get_effective_memsize();	/* CONFIG_SYS_SDRAM_BASE+PHYS_SDRAM_1_SIZE */

#if !(defined(CONFIG_SYS_ICACHE_OFF) && defined(CONFIG_SYS_DCACHE_OFF))
	/* reserve TLB table */
	gd->arch.tlb_size = PGTABLE_SIZE;
	addr -= gd->arch.tlb_size;

	/* round down to next 64 kB limit */
	addr &= ~(0x10000 - 1);

	gd->arch.tlb_addr = addr;
	debug("TLB table from %08lx to %08lx\n", addr, addr + gd->arch.tlb_size);
#endif

	/* round down to next 4 kB limit */
	addr &= ~(4096 - 1);
	debug("Top of RAM usable for U-Boot at: %08lx\n", addr);

	/*
	 * reserve memory for U-Boot code, data & bss
	 * round down to next 4 kB limit
	 */
	addr -= gd->mon_len;
	addr &= ~(4096 - 1);

	debug("Reserving %ldk for U-Boot at: %08lx\n", gd->mon_len >> 10, addr);

#ifndef CONFIG_SPL_BUILD
	/*
	 * reserve memory for malloc() arena
	 */
	addr_sp = addr - TOTAL_MALLOC_LEN;
	debug("Reserving %dk for malloc() at: %08lx\n",
			TOTAL_MALLOC_LEN >> 10, addr_sp);
	/*
	 * (permanently) allocate a Board Info struct
	 * and a permanent copy of the "global" data
	 */
	addr_sp -= sizeof (bd_t);
	bd = (bd_t *) addr_sp;
	gd->bd = bd;
	debug("Reserving %zu Bytes for Board Info at: %08lx\n",
			sizeof (bd_t), addr_sp);

#ifdef CONFIG_MACH_TYPE
	gd->bd->bi_arch_number = CONFIG_MACH_TYPE; /* board id for Linux */
#endif

	addr_sp -= sizeof (gd_t);
	id = (gd_t *) addr_sp;
	debug("Reserving %zu Bytes for Global Data at: %08lx\n",
			sizeof (gd_t), addr_sp);

#ifndef CONFIG_ARM64
	/* setup stackpointer for exeptions */
	gd->irq_sp = addr_sp;
#ifdef CONFIG_USE_IRQ
	addr_sp -= (CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ);
	debug("Reserving %zu Bytes for IRQ stack at: %08lx\n",
		CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ, addr_sp);
#endif
	/* leave 3 words for abort-stack    */
	addr_sp -= 12;

	/* 8-byte alignment for ABI compliance */
	addr_sp &= ~0x07;
#else	/* CONFIG_ARM64 */
	/* 16-byte alignment for ABI compliance */
	addr_sp &= ~0x0f;
#endif	/* CONFIG_ARM64 */
#else
	addr_sp += 128;	/* leave 32 words for abort-stack   */
	gd->irq_sp = addr_sp;
#endif

	debug("New Stack Pointer is: %08lx\n", addr_sp);

	gd->bd->bi_baudrate = gd->baudrate;
	/* Ram ist board specific, so move it to board code ... */
	dram_init_banksize(); // 更新gd->bd->bi_dram[0] 的start(PHYS_SDRAM_1)和size(PHYS_SDRAM_1_SIZE)
	display_dram_config();	/* and display it */

	gd->relocaddr = addr;
	gd->start_addr_sp = addr_sp;
	gd->reloc_off = addr - (ulong)&_start;
	debug("relocation Offset is: %08lx\n", gd->reloc_off);
	if (new_fdt) {
		memcpy(new_fdt, gd->fdt_blob, fdt_size);
		gd->fdt_blob = new_fdt;
	}
	memcpy(id, (void *)gd, sizeof(gd_t));
}
```

#### \_main 继续，接上文

在非 SPL 下，接着就是更新 `sp` 和 `gd`，然后重定位代码。

```c
; arch/arm/lib/crt0.S
	...
#if ! defined(CONFIG_SPL_BUILD)
	ldr	sp, [r9, #GD_START_ADDR_SP]	/* sp = gd->start_addr_sp */
	bic	sp, sp, #7	/* 8-byte alignment for ABI compliance */
	ldr	r9, [r9, #GD_BD]		/* r9 = gd->bd */
	sub	r9, r9, #GD_SIZE		/* new GD is below bd */
	...
#endif
```

`GD_START_ADDR_SP` 在 `include/generated/generic-asm-offsets.h` 中定义，其值为 `start_addr_sp` 在 `global_data` 结构中的相对偏移，同样是编译后才生成的。

由上面的代码可知，刚刚为 `gd` 也就是 `global_data` 分配了大小为 `GD_SIZE` 的空间，现在 `r9` 和 `sp` 内的数据为地址 `CONFIG_SYS_INIT_SP_ADDR - GD_SIZE`（需要 8 字节对齐）中的数据。换句话说此时 `r9` 中为 `gd` 的地址，那么再加上 `GD_START_ADDR_SP`，也就是 `gd+GD_START_ADDR_SP`，将 `gd+GD_START_ADDR_SP` 地址中的值（`gd->start_addr_sp`）传递给 `sp`，接着 8 字节对齐。

然后更新 `r9` 的值为 `[r9 + GD_BD]`，也就是 `r9 = gd->bd`。

然后又分配一个大小为 `GD_SIZE` 的空间。

```c
; arch/arm/lib/crt0.S
	...
#if ! defined(CONFIG_SPL_BUILD)
	...
	adr	lr, here
	ldr	r0, [r9, #GD_RELOC_OFF]		/* r0 = gd->reloc_off */
	add	lr, lr, r0
	ldr	r0, [r9, #GD_RELOCADDR]		/* r0 = gd->relocaddr */
	b	relocate_code

here:
/* Set up final (full) environment */
	bl	c_runtime_cpu_setup	/* we still call old routine here */
	...

#endif
```

将 `here` 处的 `相对地址` 复制给 `lr`，将 `gd` 结构中的 `reloc_off` 值（其相对于 `gd` 偏移为 `GD_RELOC_OFF`）给 `r0`。

`lr` 与 `r0` 相加给 `lr`,即 `lr = lr + r0`，将 `gd` 结构中的 `relocaddr` 值（其相对于 `gd` 偏移为 `GD_RELOCADDR`）给 `r0`。

此时 `lr = here + gd->reloc_off`，`r0 = gd->relocaddr`，然后调用 `relocate_code`。

#### relocate_code

其定义位于 `include/common.h` 中，由于定义了宏 `CONFIG_ARM`，所以其声明为如下代码

```c
// include/common.h
void relocate_code(ulong);
```

其实现位于 `arch/arm/lib/relocate.S` 中

是一个比较麻烦的地方，并且需要函数传参。

看代码，进入到 `relocate_code` 后，此时 r0 寄存器中数据为 `gd->relocaddr`，也就是为重定位后的 u-boot 程序地址。

```c
; arch/arm/lib/relocate.S
ENTRY(relocate_code)
	ldr	r1, =__image_copy_start	/* r1 <- SRC &__image_copy_start */
	subs	r4, r0, r1		/* r4 <- relocation offset */
	beq	relocate_done		/* skip relocation */
	ldr	r2, =__image_copy_end	/* r2 <- SRC &__image_copy_end */

copy_loop:
	ldmia	r1!, {r10-r11}		/* copy from source address [r1]    */
	stmia	r0!, {r10-r11}		/* copy to   target address [r0]    */
	cmp	r1, r2			/* until source end address [r2]    */
	blo	copy_loop

	/*
	 * fix .rel.dyn relocations
	 */
	ldr	r2, =__rel_dyn_start	/* r2 <- SRC &__rel_dyn_start */
	ldr	r3, =__rel_dyn_end	/* r3 <- SRC &__rel_dyn_end */
fixloop:
	ldmia	r2!, {r0-r1}		/* (r0,r1) <- (SRC location,fixup) */
	and	r1, r1, #0xff
	cmp	r1, #23			/* relative fixup? */
	bne	fixnext

	/* relative fix: increase location by offset */
	add	r0, r0, r4
	ldr	r1, [r0]
	add	r1, r1, r4
	str	r1, [r0]
fixnext:
	cmp	r2, r3
	blo	fixloop

relocate_done:

#ifdef __XSCALE__
	/*
	 * On xscale, icache must be invalidated and write buffers drained,
	 * even with cache disabled - 4.2.7 of xscale core developer's manual
	 */
	mcr	p15, 0, r0, c7, c7, 0	/* invalidate icache */
	mcr	p15, 0, r0, c7, c10, 4	/* drain write buffer */
#endif

	/* ARMv4- don't know bx lr but the assembler fails to see that */

#ifdef __ARM_ARCH_4__
	mov        pc, lr
#else
	bx        lr
#endif

ENDPROC(relocate_code)
```

在进入第一句代码之前，我们先了解一下 `__image_copy_start` 和 `__image_copy_end`，在 `arch/arc/lib/sections.c` 定义，并通过 `__attribute__` 放到了对应段中。在编译之后，这些段有什么用呢？

```c
// arch/arc/lib/sections.c
char __bss_start[0] __attribute__((section(".__bss_start")));
char __bss_end[0] __attribute__((section(".__bss_end")));
char __image_copy_start[0] __attribute__((section(".__image_copy_start")));
char __image_copy_end[0] __attribute__((section(".__image_copy_end")));
char __rel_dyn_start[0] __attribute__((section(".__rel_dyn_start")));
char __rel_dyn_end[0] __attribute__((section(".__rel_dyn_end")));
char __text_start[0] __attribute__((section(".__text_start")));
char __text_end[0] __attribute__((section(".__text_end")));
char __init_end[0] __attribute__((section(".__init_end")));
```

查看 `u-boot.lds`，在 `arch/arm/cpu/u-boot.lds`，但这个并不是最终的，最终的链接脚本是在此基础上生成的。需要编译 u-boot 后会在根目录下才会生成 u-boot.lds。

> GNU 编译器生成的目标文件缺省为 elf 格式，elf 文件由若干段（section）组成，如不特殊指明，由 C 源程序生成的目标代码中包含如下段：
>
> - .text(正文段) 包含程序的指令代码；
> - .data(数据段) 包含固定的数据，如常量、字符串；
> - .bss(未初始化数据段) 包含未初始化的变量、数组等。
>
> C++ 源程序生成的目标代码中还包括
>
> - .fini(析构函数代码)
> - .init(构造函数代码) 等.
>   链接器的任务就是将多个目标文件的.text、.data 和.bss 等段链接在一起，而链接脚本文件是告诉链接器从什么地址开始放置这些段.简而言之，由于一个工程中有多个.c 文件，当它们生成.o 文件后如何安排它们在可执行文件中的顺序，这就是链接脚本的作用.
>
> 这里以 u-boot 的 lds 为例说明 uboot 的链接过程，首先看一下 GNU 官方网站上对.lds 文件形式的完整描述：
>
> ```c
> SECTIONS {
> ...
> secname start BLOCK(align) (NOLOAD) : AT ( ldadr )
> { contents } >region :phdr =fill
> ...
> }
> ```
>
> 其中，secname 和 contents 是必须的，前者用来命名这个段，后者用来确定代码中的什么部分放在这个段，以下是对这个描述中的一些关键字的解释。
>
> - secname：段名
> - contents：决定哪些内容放在本段，可以是整个目标文件（如 start.o），也可以是目标 - 文件中的某段（代码段、数据段等）（如 start.o (.text .rodata)）
> - start：是段的重定位地址，本段链接（运行）的地址，如果代码中有位置无关指令，程序运行时这个段必须放在这个地址上。start 可以用任意一种描述地址的符号来描述。
>   AT（ldadr）：定义本段存储（加载）的地址，如果不使用这个选项，则加载地址等于运行地址，通过这个选项可以控制各段分别保存于输出文件中不同的位置。

上面我们了解到 lds 文件的格式之后，就可以直接阅读下面的 lds 文件，`. = 0x00000000;` 定义当前位置为 `0x00000000`; 紧接着 `.text:{...}`，所以.text 的相对地址也是 0，而在.text 中 `*(.__image_copy_start)` 又在开始的位置，也就是说把我们前面定义的 `.__image_copy_start` 段在链接的时候挪到这个位置当前位置（0x0），紧接着 `arch/arm/cpu/armv7/start.o (.text*)` 将 `arch/arm/cpu/armv7/start.o` 中的 `.text*` 段全部挪到当前位置（当前位置为 0x0 + `.__image_copy_start` 大小），后面也是如此。

...

将 `.__image_copy_end` 挪到 `.image_copy_end` 段。所以从 `.__image_copy_start` 到 `.__image_copy_end` 包括了代码段、只读数据段、读写数据段、`.u_boot_list`(uboot 的命令)。

```c
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
OUTPUT_ARCH(arm)
ENTRY(_start)
SECTIONS
{
 . = 0x00000000;
 . = ALIGN(4);
 .text :
 {
  *(.__image_copy_start)
  arch/arm/cpu/armv7/start.o (.text*)
  *(.text*)
 }
 . = ALIGN(4);
 .rodata : { *(SORT_BY_ALIGNMENT(SORT_BY_NAME(.rodata*))) }
 . = ALIGN(4);
 .data : {
  *(.data*)
 }
 . = ALIGN(4);
 . = .;
 . = ALIGN(4);
 .u_boot_list : {
  KEEP(*(SORT(.u_boot_list*)));
 }
 . = ALIGN(4);
 .image_copy_end :
 {
  *(.__image_copy_end)
 }
 .rel_dyn_start :
 {
  *(.__rel_dyn_start)
 }
 .rel.dyn : {
  *(.rel*)
 }
 .rel_dyn_end :
 {
  *(.__rel_dyn_end)
 }
 .end :
 {
  *(.__end)
 }
 _image_binary_end = .;
 . = ALIGN(4096);
 .mmutable : {
  *(.mmutable)
 }
 .bss_start __rel_dyn_start (OVERLAY) : {
  KEEP(*(.__bss_start));
  __bss_base = .;
 }
 .bss __bss_base (OVERLAY) : {
  *(.bss*)
   . = ALIGN(4);
   __bss_limit = .;
 }
 .bss_end __bss_limit (OVERLAY) : {
  KEEP(*(.__bss_end));
 }
 .dynsym _image_binary_end : { *(.dynsym) }
 .dynbss : { *(.dynbss) }
 .dynstr : { *(.dynstr*) }
 .dynamic : { *(.dynamic*) }
 .plt : { *(.plt*) }
 .interp : { *(.interp*) }
 .gnu.hash : { *(.gnu.hash) }
 .gnu : { *(.gnu*) }
 .ARM.exidx : { *(.ARM.exidx*) }
 .gnu.linkonce.armexidx : { *(.gnu.linkonce.armexidx.*) }
}
```

接着我们回到代码 `ldr r1, =__image_copy_start` 则是将 `__image_copy_start` 的地址取出给到 r1，`r4=r0-r1`,r0 为重定位后的代码位置，r1 为当前 `__image_copy_start` 的位置，那么 r4 为代码重定位前后的差，也就是 `重定位的偏移`。`beq` 实际上是一个跳转指令 `b` 和一个条件变量 `eq` 组合起来的，其意思也就是如果相等则跳转到 `relocate_done`。什么意思呢？这个地方怎么可能会相等呢？不对，你在想想，当前代码的起始位置是 `__image_copy_start`，重定位后代码的位置是 r0 即 `gd->relocaddr`，当重定位后，当前代码的起始位置是哪呢？不就是 `gd->relocaddr`，所以此时 `r0-r1` 为 `0`，也即满足相等的条件，跳转到 `relocate_done`，不需要重定位（**此处有疑问？该代码上电后会执行两次吗？如果不会执行两次，那么相等这个条件也就不会满足，也就不会有跳转这个分支了。。。。**）。重定位完成后执行 xxxxxxx 操作后跳出。

```c
; arch/arm/lib/relocate.S
ENTRY(relocate_code)
	ldr	r1, =__image_copy_start	  /* r1 <- SRC &__image_copy_start */
	subs	r4, r0, r1		       /* r4 <- relocation offset */
	beq	relocate_done		   /* skip relocation */
	ldr	r2, =__image_copy_end    /* r2 <- SRC &__image_copy_end */
	...

relocate_done:
#ifdef __XSCALE__
	/*
	 * On xscale, icache must be invalidated and write buffers drained,
	 * even with cache disabled - 4.2.7 of xscale core developer's manual
	 */
	mcr	p15, 0, r0, c7, c7, 0	/* invalidate icache */
	mcr	p15, 0, r0, c7, c10, 4	/* drain write buffer */
#endif
	...
ENDPROC(relocate_code)
```

建议先去寄存器中查看 LDM、STM 命令以及模式的用法。此时 `r0=gd->relocaddr`，`r1=&__image_copy_start`，`r2=&__image_copy_end`

此处从 r1 地址开始，将数据拷贝 r10、r11，每次拷贝后 r1 地址加 4，结束后 r1 的值为刚开始的值 +8。

然后此处从 r0 地址开始，将 r10、r11 寄存器中的值拷贝到 r0 地址处，每次拷贝结束后 r0 地址加 4，所以，结束后 r0 的值为刚开始的值 +8。

比较 r1 和 r2 的值，当 r1<r2 时，跳转到 `copy_loop`，继续拷贝。直到 r1>=r2 时，拷贝结束。此时 `__image_copy_start` 至 `__image_copy_end` 地址的数据已经拷贝到 `gd->relocaddr` 处了。

```c
; arch/arm/lib/relocate.S
ENTRY(relocate_code)
	...
copy_loop:
	ldmia	r1!, {r10-r11}		/* copy from source address [r1]    */
	stmia	r0!, {r10-r11}		/* copy to   target address [r0]    */
	cmp	r1, r2			          /* until source end address [r2]    */
	blo	copy_loop
	...
ENDPROC(relocate_code)
```

接着 `r2=__rel_dyn_start`，`r3=__rel_dyn_end`，从 r2 处加载数据到 r0 和 r1 中，取出 r1 中数据的低 8 位，与 23 比较，如果不相等，跳转到 `fixnext`，比较 r2、r3，如果 r2<r3，说明没修正结束，接着修正；如果相等，`r0=r0+r4`，停，r4 是什么？大家还有印象不，前面我们在刚进入 `relocate_code`，r4 为重定位前后的代码的相对地址偏移。所以代码的意思就是 r0=r0+r4，然后以 r0 中存的值为地址取出 已拷贝到新地址的.text 段中的值；将该值加上新旧.text 段的偏移，在写回到 新的.text 段中。这样就完成了重定位。**此处需明白重定位，到底重定位的是什么？**

代码解释完了，但是这又是什么意思啊？完全不理解，从 `.rel.dyn` 取出的是什么值啊，接下来我们了解了解 `.rel.dyn` 段，之后在回来看这段代码，应该就一目了然了。

```c
; arch/arm/lib/relocate.S
ENTRY(relocate_code)
	...
	/*
	 * fix .rel.dyn relocations
	 */
	ldr	r2, =__rel_dyn_start	/* r2 <- SRC &__rel_dyn_start */
	ldr	r3, =__rel_dyn_end	/* r3 <- SRC &__rel_dyn_end */
fixloop:
	ldmia	r2!, {r0-r1}		/* (r0,r1) <- (SRC location,fixup) */
	and	r1, r1, #0xff
	cmp	r1, #23			/* relative fixup? */
	bne	fixnext

	/* relative fix: increase location by offset */
	add	r0, r0, r4
	ldr	r1, [r0]
	add	r1, r1, r4
	str	r1, [r0]
fixnext:
	cmp	r2, r3
	blo	fixloop

relocate_done:
	...
ENDPROC(relocate_code)
```

> `.rel.dyn` 存放 `.text` 段中需要重定位地址的集合
>
> 接下来我们来说明一个很重要的概念。
>
> u-boot 在启动过程中，会把自己拷贝到 RAM 的顶端去执行。这一拷贝带来的问题是执行地址的混乱。代码的执行地址通常都是在编译时有链接地址指定的，如何保证拷贝前后都可以执行呢？
>　　一个办法是使用拷贝到 RAM 后的地址作为编译时的链接地址，拷贝前所有函数与全局变量的调用都增加偏移量。（如 VxWorks 的 bootloader）尽量减少拷贝前需要执行的代码量。
>　　另一个办法是把 image 编译成与地址无关的程序，也就是 PIC - Position independent code。编译器无法保证代码的独立性，它需要与加载器配合起来。U-boot 自己加载自己，所以它自己就是加载器。PIC 依赖于下面两种技术：
> 1）使用相对地址
> 2）加载器可以自动更新涉及到绝对地址的指令
>　　对于 PowerPC 架构，u-boot 只是在编译时使用了 -fpic，这种方式会生成一个.got 段来存储绝对地址符号。对于 ARM 架构，则是在编译时使用 -mword-relocations，生成与位置无关代码，链接时使用 -pie 生成.rel.dyn 段，该段中的每个条目被称为一个 LABEL，用来存储绝对地址符号的地址。
>
> 为了理解地址表的概念我们分析一段代码。
>
> 借助两个工具 `readelf`、`objdump`。
>
> ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204131923808.png)
>
> 通过 readelf 的 `-r` 选项查看 `relocate段`，`readelf -r u-boot | less` 打开，看到类型是可重定位段，其中有了 `offset、info、type` 以及后面没有为空的项。m 每个需要修改地址的信息占用 8 个字节，第一个为可重定位的地址，第二个为 0x17，0x17 对应的是 Arm32 位。那么让我们看一下第一条可重定位地址处（0x20000020）放的是什么？![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204132024704.png)
>
> 通过命令 `objdump -S u-boot | less`，看到此处存的正是 start.S 中 `_undefined_instruction: .word undefined_instruction` 入口地址。
>
> 因为这个入口地址如果直接.text 段拷贝过去，将来执行跳转的还是旧的 uboot 里面的 undefined_instruction，而不是我们新 uboot 里面的 undefined_instruction，所以这个要修改。
>
> 即要修改所有位置有关码的地址。
>
> 如何修改？
>
> 很简单，**旧的地址的值是什么我们取出来加上 新地址和旧地址的偏移，然后存入新的地址就可以了**。
>
> 因为编译器已经帮我们剥离出来要修改的地址和它所属的类型，存放进了 rel.dyn 段，所以我们只要修改新地址的就可以。![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204131932905.png)
>
> ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204131936214.png)

未定义\_\_ARM_ARCH_4\_\_，所以执行 `bx lr`，而此时 `lr` 中的值为 `here + gd->reloc_off`，怎么解释？此时重定位已经完成，重定位后的地址差值为 `gd->reloc_off`，`here` 为 `here` 处的相对地址，所以 **二者相加后为重定位后的 `here` 处的地址**，然后跳回去。

```c
; arch/arm/lib/relocate.S
/* ARMv4- don't know bx lr but the assembler fails to see that */

#ifdef __ARM_ARCH_4__
	mov        pc, lr
#else
	bx        lr
#endif
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204151531408.png)

#### \_main 继续，接上文：here

回到 `here` 处开始跳转到 `c_runtime_cpu_setup`，`c_runtime_cpu_setup` 执行结束后返回。

##### :negative_squared_cross_mark: c_runtime_cpu_setup

`c_runtime_cpu_setup` 位于 `arch/arm/cpu/armv7/start.S`，

配置协处理器 CP15

加载程序起始地址到 r0，设置 `CP15 VBAR register`

**TODO：待补充**

```c
; arch/arm/cpu/armv7/start.S
ENTRY(c_runtime_cpu_setup)
/*
 * If I-cache is enabled invalidate it
 */
#ifndef CONFIG_SYS_ICACHE_OFF
	mcr	   p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB
#endif
/*
 * Move vector table
 */
	/* Set vector address in CP15 VBAR register */
	ldr     r0, =_start
	mcr     p15, 0, r0, c12, c0, 0  @Set VBAR

	bx	lr

ENDPROC(c_runtime_cpu_setup)
```

继续执行 `r0 = __bss_start`，`r1 = __bss_end`，`r2 = 0`

```c
; arch/arm/lib/crt0.S
here:

/* Set up final (full) environment */

	bl	c_runtime_cpu_setup	/* we still call old routine here */

	ldr	r0, =__bss_start	/* this is auto-relocated! */
	ldr	r1, =__bss_end		/* this is auto-relocated! */

	mov	r2, #0x00000000		/* prepare zero to clear BSS */
```

将 `[__bss_start,__bss_end]` 范围内的内存清零。

比较 `r0` 和 `r1` 的值，首先 `r0=r0-r1`,当满足条件 `r0<r1` 时，将 `r2` 中的值 0 复制给以将 `r0` 的值作为地址的内存中，当满足条件 `r0<r1` 时，`r0=r0+4`，当满足条件 `r0<r1` 时，跳转到 `clbss_l` 处。接着循环，直到二者相等时，不满足条件，继续执行 `bl coloured_LED_init`。

```c
; arch/arm/lib/crt0.S

clbss_l:cmp	r0, r1			/* while not at end of BSS */
	strlo	r2, [r0]		/* clear 32-bit BSS word */
	addlo	r0, r0, #4		/* move to next */
	blo	clbss_l
```

跳转到 coloured_LED_init，跳转到 red_led_on，没啥内容。主要关注一下语法即可。

```c
; arch/arm/lib/crt0.S

	bl coloured_LED_init
	bl red_led_on
------------------------------
// common/board_f.c
inline void __coloured_LED_init(void) {}
void coloured_LED_init(void)
        __attribute__((weak, alias("__coloured_LED_init")));
inline void __red_led_on(void) {}
void red_led_on(void) __attribute__((weak, alias("__red_led_on")));
```

`r0 = r9 = gd`，`r1 = [r9(gd) + GD_RELOCADDR]`，也就是说 `r0` 此时为 `gd`,`r1` 为相对于 `gd` 偏移 `GD_RELOCADDR` 的地址中的值。

将 `pc` 值更改为 `board_init_r`，也就是跳转到 `board_init_r` 执行，此处不需要返回。因为要跳转到 c 代码，而 c 代码入口有两个参数，有 APCS 可知前四个整形实参被存入到 a1-a4 中，也即 r0-r3 中。此时 r0 为 gd，r1 为 `gd->relocaddr`

```c
; arch/arm/lib/crt0.S
	/* call board_init_r(gd_t *id, ulong dest_addr) */
	mov     r0, r9                  /* gd_t */
	ldr	r1, [r9, #GD_RELOCADDR]		/* dest_addr */
	/* call board_init_r */
	ldr	pc, =board_init_r	/* this is auto-relocated! */

	/* we should not return here. */
```

到此 `_main` 就结束了。

#### board_init_r

~~**board_init_r** 函数位于 `common/board_r.c` 中，位置分析错误~~，可以看下 `common/Makefile`，通过命令查找 `grep -nr CONFIG_SYS_GENERIC_BOARD` 有关其定义，在我们的 `smart210.h` 中并没有定义这个宏，所以调用的 `board_init_r` 不在此处的 `board_r.c` 中。

```shell
# boards
obj-$(CONFIG_SYS_GENERIC_BOARD) += board_f.o
obj-$(CONFIG_SYS_GENERIC_BOARD) += board_r.o
```

所以 `board_init_r` 函数其实位于 `arch/arm/lib/board.c`，跟 `board_init_f` 函数位于同一个文件中。

代码如下，我滴个老天爷这也太多了吧。一点点来，先把没用到的宏开关去掉。

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	ulong malloc_start;
#if !defined(CONFIG_SYS_NO_FLASH)
	ulong flash_size;
#endif

	gd->flags |= GD_FLG_RELOC;	/* tell others: relocation done */
	bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_R, "board_init_r");

	monitor_flash_len = (ulong)&__rel_dyn_end - (ulong)_start;

	/* Enable caches */
	enable_caches();

	debug("monitor flash len: %08lX\n", monitor_flash_len);
	board_init();	/* Setup chipselects */

	serial_initialize();

	debug("Now running in RAM - U-Boot at: %08lx\n", dest_addr);

	/* The Malloc area is immediately below the monitor copy in DRAM */
	malloc_start = dest_addr - TOTAL_MALLOC_LEN;
	mem_malloc_init (malloc_start, TOTAL_MALLOC_LEN);

#ifdef CONFIG_ARCH_EARLY_INIT_R
	arch_early_init_r();
#endif
	power_init_board();

#if !defined(CONFIG_SYS_NO_FLASH)
	puts("Flash: ");

	flash_size = flash_init();
	if (flash_size > 0) {
# ifdef CONFIG_SYS_FLASH_CHECKSUM
		print_size(flash_size, "");
		/*
		 * Compute and print flash CRC if flashchecksum is set to 'y'
		 *
		 * NOTE: Maybe we should add some WATCHDOG_RESET()? XXX
		 */
		if (getenv_yesno("flashchecksum") == 1) {
			printf("  CRC: %08X", crc32(0,
				(const unsigned char *) CONFIG_SYS_FLASH_BASE,
				flash_size));
		}
		putc('\n');
# else	/* !CONFIG_SYS_FLASH_CHECKSUM */
		print_size(flash_size, "\n");
# endif /* CONFIG_SYS_FLASH_CHECKSUM */
	} else {
		puts(failed);
		hang();
	}
#endif

#if defined(CONFIG_CMD_NAND)
	puts("NAND:  ");
	nand_init();		/* go init the NAND */
#endif


#ifdef CONFIG_HAS_DATAFLASH
	AT91F_DataflashInit();
	dataflash_print_info();
#endif

	/* initialize environment */
	if (should_load_env())
		env_relocate();
	else
		set_default_env(NULL);

	stdio_init();	/* get the devices list going. */

	jumptable_init();

#if defined(CONFIG_API)
	/* Initialize API */
	api_init();
#endif

	console_init_r();	/* fully init console as a device */

#ifdef CONFIG_DISPLAY_BOARDINFO_LATE
	checkboard();
#endif

#if defined(CONFIG_ARCH_MISC_INIT)
	/* miscellaneous arch dependent initialisations */
	arch_misc_init();
#endif
#if defined(CONFIG_MISC_INIT_R)
	/* miscellaneous platform dependent initialisations */
	misc_init_r();
#endif

	 /* set up exceptions */
	interrupt_init();
	/* enable exceptions */
	enable_interrupts();

	/* Initialize from environment */
	load_addr = getenv_ulong("loadaddr", 16, load_addr);

#ifdef CONFIG_BOARD_LATE_INIT
	board_late_init();
#endif

#ifdef CONFIG_BITBANGMII
	bb_miiphy_init();
#endif
#if defined(CONFIG_CMD_NET)
	puts("Net:   ");
	eth_initialize(gd->bd);
#if defined(CONFIG_RESET_PHY_R)
	debug("Reset Ethernet PHY\n");
	reset_phy();
#endif
#endif

#ifdef CONFIG_POST
	post_run(NULL, POST_RAM | post_bootmode_get(0));
#endif

#if defined(CONFIG_PRAM) || defined(CONFIG_LOGBUFFER)
	/*
	 * Export available size of memory for Linux,
	 * taking into account the protected RAM at top of memory
	 */
	{
		ulong pram = 0;
		uchar memsz[32];

#ifdef CONFIG_PRAM
		pram = getenv_ulong("pram", 10, CONFIG_PRAM);
#endif
#ifdef CONFIG_LOGBUFFER
#ifndef CONFIG_ALT_LB_ADDR
		/* Also take the logbuffer into account (pram is in kB) */
		pram += (LOGBUFF_LEN + LOGBUFF_OVERHEAD) / 1024;
#endif
#endif
		sprintf((char *)memsz, "%ldk", (gd->ram_size / 1024) - pram);
		setenv("mem", (char *)memsz);
	}
#endif

	/* main_loop() can return to retry autoboot, if so just run it again. */
	for (;;) {
		main_loop();
	}

	/* NOTREACHED - no way out of command loop except booting */
}
```

分析这个函数，这也太麻烦了。。。。。

先看一部分，首先设置 gd 中的 flags 告诉其它，我已经重定位完了。然后标记当前段为 `board_init_r`，然后计算 flash 监控区的长度 从 `_start` 到 `__rel_syn_end` 包括了代码段、只读数据段、读写数据段、`.u_boot_list`(uboot 的命令) 和重定位地址表段。

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	ulong malloc_start;
#if !defined(CONFIG_SYS_NO_FLASH)
	ulong flash_size;
#endif

	gd->flags |= GD_FLG_RELOC;	/* tell others: relocation done */
	bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_R, "board_init_r");

	monitor_flash_len = (ulong)&__rel_dyn_end - (ulong)_start;
        ...
	/* NOTREACHED - no way out of command loop except booting */
}
```

##### enable_caches

看下 enable_caches 函数的实现。

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
        enable_caches();
        ...
}
```

在 `arch/arm/lib/cache.c` 为默认实现，其也就是打印一下 cache 未开启。在此处，需注意下，`__attribute__` 的高级用法，声明一个弱符号函数，如果外部没有实现 enable_caches，就去调用默认的。

```c
// arch/arm/lib/cache.c
void __enable_caches(void)
{
	puts("WARNING: Caches not enabled\n");
}
void enable_caches(void)
	__attribute__((weak, alias("__enable_caches")));

```

##### :negative_squared_cross_mark: board_init

接着我们看下 board_init，网卡的初始化

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
        board_init();
        ...
}
```

board_init 是在对应的板卡文件中实现的，实现 dm9000 网卡的初始化，配置板卡的类型代号以及板卡中 boot 的参数区，起始地址为 `PHYS_SDRAM_1 + 0x100`，也即 `0x20000100`

```c
// board/samsung/smart210/smart210.c
int board_init(void)
{
	//smc9115_pre_init();
	dm9000_pre_init();

	gd->bd->bi_arch_number = MACH_TYPE_SMDKV210;
	gd->bd->bi_boot_params = PHYS_SDRAM_1 + 0x100;

	return 0;
}
```

接着看一下网卡的预初始化。呃，定义两个参数，对两个参数赋值，然后在将两个参数写入到 SROM 中。**这个值怎么来的呢？**

smc_bw_conf 和 smc_bc_conf 的配置可查看 `S5PV210_UM_REV1.1.pdf` 中 `2.4.1.1 SROM Bus Width` 与 `2.4.1.2 SROM Bank Control Register`

```c
// board/samsung/smart210/smart210.c
static void dm9000_pre_init(void)
{
	u32 smc_bw_conf, smc_bc_conf;

	/* Ethernet needs bus width of 16 bits */
	smc_bw_conf = SMC_DATA16_WIDTH(CONFIG_ENV_SROM_BANK)
		| SMC_BYTE_ADDR_MODE(CONFIG_ENV_SROM_BANK);
	smc_bc_conf = SMC_BC_TACS(0) | SMC_BC_TCOS(1) | SMC_BC_TACC(2)
		| SMC_BC_TCOH(1) | SMC_BC_TAH(0) | SMC_BC_TACP(0) | SMC_BC_PMC(0);

	/* Select and configure the SROMC bank */
	s5p_config_sromc(CONFIG_ENV_SROM_BANK, smc_bw_conf, smc_bc_conf);
}
```

这段展开也太多了，先留着，后续仔细分析 **TODO**

首先我们了解一下板卡硬件是怎么连的，由于是百兆口，只有四根线

```shell
网线 			   <---> RJ45(HR911105A) 		 	<---> phy(DM9000A)  <---> 芯片(s5pv210)
4根线	      			   自带变压器转换信号也还是4根线
两组差分信号 			两组差分信号
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204151450950.png)

##### serial_initialize

上面讲了，看 [serial_init](#serial_init)

##### mem_malloc_init

接着是计算 malloc 内存的起始地址和初始化

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
        malloc_start = dest_addr - TOTAL_MALLOC_LEN;
	mem_malloc_init (malloc_start, TOTAL_MALLOC_LEN);
        ...
}
```

计算 `malloc_start` 的地址，由 [c_runtime_cpu_setup](#c_runtime_cpu_setup) 可知 `gd->relocaddr` 为 `board_init_r` 函数的第二个参数，也即 `dest_addr=gd->relocaddr`，

而 `TOTAL_MALLOC_LEN` 被定义为 `(CONFIG_SYS_MALLOC_LEN + CONFIG_ENV_SIZE)=（CONFIG_ENV_SIZE + (1<<20) + CONFIG_ENV_SIZE）= 128<<10 * 2 + 1<<20=256k+1024k=1280k`，所以通过 `dest_addr-TOTAL_MALLOC_LEN` 获得可分配的最低位置。

```c
#define	TOTAL_MALLOC_LEN	(CONFIG_SYS_MALLOC_LEN + CONFIG_ENV_SIZE)
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204151531408.png)

接着便是对这段范围内的内存初始化，设置 malloc 的开始、结束及 brk 的位置，并将内存内的数据清零。由于未定义 `CONFIG_NEEDS_MANUAL_RELOC` 宏，所以此处 `malloc_bin_reloc` 为空。

```c
// common/dlmalloc.c
void mem_malloc_init(ulong start, ulong size)
{
	mem_malloc_start = start;
	mem_malloc_end = start + size;
	mem_malloc_brk = start;

	memset((void *)mem_malloc_start, 0, size);

	malloc_bin_reloc();
}
```

##### power_init_board

电源初始化，由于未定义 `CONFIG_POWER` 宏，所以此处实现为空

```c
__weak int power_init_board(void)
{
	return 0;
}
```

##### nand_init

此处开始初始化 nand，跳转到 nand_init 看一下。

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
#if defined(CONFIG_CMD_NAND)
	puts("NAND:  ");
	nand_init();		/* go init the NAND */
#endif
        ...
}
```

由于我们没有定义 `CONFIG_SYS_NAND_SELF_INIT`，CONFIG_SYS_MAX_NAND_DEVICE 大小为 1，所以会进入下面的循环初始化中，`CONFIG_SYS_NAND_SELECT_DEVICE` 没有定义，所以此函数中进有 `nand_init_chip(0);`

```c
// drivers/mtd/nand/nand.c
void nand_init(void)
{
#ifdef CONFIG_SYS_NAND_SELF_INIT
	board_nand_init();
#else
	int i;
	for (i = 0; i < CONFIG_SYS_MAX_NAND_DEVICE; i++)
		nand_init_chip(i);
#endif
	printf("%lu MiB\n", total_nand_size / 1024);

#ifdef CONFIG_SYS_NAND_SELECT_DEVICE
	/*
	 * Select the chip in the board/cpu specific driver
	 */
	board_nand_select_device(nand_info[nand_curr_device].priv, nand_curr_device);
#endif
}
```

接着进入 `nand_init_chip`，了解 `mtd_info`、`nand_info` 结构体，mtd 指向全局变量 nand_info，这个变量就是 nand 设备的信息；再看初始化：mtd->private=nand; mtd 的私有数据就是一个指向 `struct nand_chip` 类型的结构体

> 从编程的角度来说，一个硬件驱动应该有两个面，一个面向上层，提供接口；一个面向底层，提供硬件操作。
>
> 从广义上来看：
>
> `struct mtd_info` 就是面向上层，提供数据接口
>
> `struct nand_chip` 面向 nand 设备，提供硬件接口

```c
// drivers/mtd/nand/nand.c
#ifndef CONFIG_SYS_NAND_SELF_INIT
static void nand_init_chip(int i)
{
	struct mtd_info *mtd = &nand_info[i];
	struct nand_chip *nand = &nand_chip[i];
	ulong base_addr = base_address[i];
	int maxchips = CONFIG_SYS_NAND_MAX_CHIPS;

	if (maxchips < 1)
		maxchips = 1;

	mtd->priv = nand;
	nand->IO_ADDR_R = nand->IO_ADDR_W = (void  __iomem *)base_addr;

	if (board_nand_init(nand))
		return;

	if (nand_scan(mtd, maxchips))
		return;

	nand_register(i);
}
#endif
```

基于这个思路，看一下 `struct mtd_info`，记住几个

> **mtd_info** 描述原始设备层的一个分区的结构, 描述一个设备或一个多分区设备中的一个分区
>
> - type MTD 设备类型，有 MTD_RAM、MTD_ROM、MTD_NORFLASH、MTD_NAND_FLASH 等
> - flags 读写及权限标志位，有 MTD_WRITEABLE、MTD_BIT_WRITEABLE、MTD_NO_ERASE、MTD_UP_LOCK
> - size MTD 设备的大小
> - erase 主要擦除块的大小，NandFlash 就是块的大小
> - writesize 最小可写字节数，NandFlash 对应着 " 页 "
> - oobsize 一个 blokc 中可用的 oob 的字节数
> - oobavail 一个 block 中可用 oob 字节数
> - 接着下面就是一些函数指针，用于上层调用
> - priv 私有数据指针

```c
struct mtd_info {
	u_char type;		/* 设备类型 */
	u_int32_t flags;
	uint64_t size;	 /* Total size of the MTD */

	/* "Major" erase size for the device. Naïve users may take this
	 * to be the only erase size available, or may use the more detailed
	 * information below if they desire
	 */
	u_int32_t erasesize;
	/* Minimal writable flash unit size. In case of NOR flash it is 1 (even
	 * though individual bits can be cleared), in case of NAND flash it is
	 * one NAND page (or half, or one-fourths of it), in case of ECC-ed NOR
	 * it is of ECC block size, etc. It is illegal to have writesize = 0.
	 * Any driver registering a struct mtd_info must ensure a writesize of
	 * 1 or larger.
	 */
	u_int32_t writesize;

	u_int32_t oobsize;   /* Amount of OOB data per block (e.g. 16) */
	u_int32_t oobavail;  /* Available OOB bytes per block */

	/*
	 * read ops return -EUCLEAN if max number of bitflips corrected on any
	 * one region comprising an ecc step equals or exceeds this value.
	 * Settable by driver, else defaults to ecc_strength.  User can override
	 * in sysfs.  N.B. The meaning of the -EUCLEAN return code has changed;
	 * see Documentation/ABI/testing/sysfs-class-mtd for more detail.
	 */
	unsigned int bitflip_threshold;

	/* Kernel-only stuff starts here. */
	const char *name;
	int index;

	/* ECC layout structure pointer - read only! */
	struct nand_ecclayout *ecclayout;

	/* max number of correctible bit errors per ecc step */
	unsigned int ecc_strength;

	/* Data for variable erase regions. If numeraseregions is zero,
	 * it means that the whole device has erasesize as given above.
	 */
	int numeraseregions;
	struct mtd_erase_region_info *eraseregions;

	/*
	 * Do not call via these pointers, use corresponding mtd_*()
	 * wrappers instead.
	 */
	int (*_erase) (struct mtd_info *mtd, struct erase_info *instr);
	int (*_point) (struct mtd_info *mtd, loff_t from, size_t len,
			size_t *retlen, void **virt, phys_addr_t *phys);
	void (*_unpoint) (struct mtd_info *mtd, loff_t from, size_t len);
	int (*_read) (struct mtd_info *mtd, loff_t from, size_t len,
		     size_t *retlen, u_char *buf);
	int (*_write) (struct mtd_info *mtd, loff_t to, size_t len,
		      size_t *retlen, const u_char *buf);

	/* In blackbox flight recorder like scenarios we want to make successful
	   writes in interrupt context. panic_write() is only intended to be
	   called when its known the kernel is about to panic and we need the
	   write to succeed. Since the kernel is not going to be running for much
	   longer, this function can break locks and delay to ensure the write
	   succeeds (but not sleep). */

	int (*_panic_write) (struct mtd_info *mtd, loff_t to, size_t len, size_t *retlen, const u_char *buf);

	int (*_read_oob) (struct mtd_info *mtd, loff_t from,
			 struct mtd_oob_ops *ops);
	int (*_write_oob) (struct mtd_info *mtd, loff_t to,
			 struct mtd_oob_ops *ops);
	int (*_get_fact_prot_info) (struct mtd_info *mtd, struct otp_info *buf,
				   size_t len);
	int (*_read_fact_prot_reg) (struct mtd_info *mtd, loff_t from,
				   size_t len, size_t *retlen, u_char *buf);
	int (*_get_user_prot_info) (struct mtd_info *mtd, struct otp_info *buf,
				   size_t len);
	int (*_read_user_prot_reg) (struct mtd_info *mtd, loff_t from,
				   size_t len, size_t *retlen, u_char *buf);
	int (*_write_user_prot_reg) (struct mtd_info *mtd, loff_t to, size_t len,
				    size_t *retlen, u_char *buf);
	int (*_lock_user_prot_reg) (struct mtd_info *mtd, loff_t from,
				   size_t len);
	void (*_sync) (struct mtd_info *mtd);
	int (*_lock) (struct mtd_info *mtd, loff_t ofs, uint64_t len);
	int (*_unlock) (struct mtd_info *mtd, loff_t ofs, uint64_t len);
	int (*_block_isbad) (struct mtd_info *mtd, loff_t ofs);
	int (*_block_markbad) (struct mtd_info *mtd, loff_t ofs);
	/*
	 * If the driver is something smart, like UBI, it may need to maintain
	 * its own reference counting. The below functions are only for driver.
	 */
	int (*_get_device) (struct mtd_info *mtd);
	void (*_put_device) (struct mtd_info *mtd);

	/* ECC status information */
	struct mtd_ecc_stats ecc_stats;
	/* Subpage shift (NAND) */
	int subpage_sft;

	void *priv;

	struct module *owner;
	int usecount;
};
```

结构知道了，其值在哪定义的呢，在同文件下，定义了一个结构体数组，其大小为 `CONFIG_SYS_MAX_NAND_DEVICE`

```c
nand_info_t nand_info[CONFIG_SYS_MAX_NAND_DEVICE];
----------------------------------------------------------------------------------------
 // include/configs/smart210.h
 #define CONFIG_SYS_MAX_NAND_DEVICE  1
```

接着看 `mtd->priv = nand` 也就是将 nand 的结构体指针给了 mtd 的私有变量，说明 mtd 的下层操作的设备为 nand。接着往下走，`board_nand_init(nand)` 初始化 nand，配置 nand 一些参数及操作（代码可见 `drivers/mtd/nand/s5pv210_nand.c` 中的 `board_nand_init`）。

`nand_scan_ident` 扫描设备，获取当前 mtd 中的私有变量 `priv` 中的数据结构（刚刚我们在 `nand_init_chip` 函数中赋值 priv 为 nand，实际是全局变量，并经 `board_nand_init` 初始化），获取 buswidth 并设置、获取 chip 数量和大小。

`nand_scan_tail` 用默认值填充所有未初始化的函数指针

```c
nand_scan(mtd, maxchips)
------------------------------------------
// drivers/mtd/nand/nand_base.c
int nand_scan(struct mtd_info *mtd, int maxchips)
{
	int ret;
	ret = nand_scan_ident(mtd, maxchips, NULL); // 获取当前mtd下
	if (!ret)
		ret = nand_scan_tail(mtd);
	return ret;
}
```

还有一个注册函数，其实就是配置设备名称，形如 `nand0`，然后计算总的 `total_nand_size` 大小，并将上面的 `&nand_info[devnum]` 添加到 `mtd_table` 中，over。这个地方就结束了，其实想要更深层次的了解最好去看下 [参考 22](#参考)。

总结一下，其实就是定义了 2 个结构体数组，分别为 `nand_info`、`nand_chip`，分别初始化，然后赋值，`&nand_info[0]->prvi=&nand_chip[0]`，并在 `nand_chip` 中实现对应的操作函数。之后呢，在将 `&nand_info[0]` 放到 `mtd_table` 中，后续在使用时应该就是上层直接使用 `mtd_table` 来完成相应的操作。

```c
int nand_register(int devnum)
{
	struct mtd_info *mtd;

	if (devnum >= CONFIG_SYS_MAX_NAND_DEVICE)
		return -EINVAL;

	mtd = &nand_info[devnum];

	sprintf(dev_name[devnum], "nand%d", devnum);
	mtd->name = dev_name[devnum];

#ifdef CONFIG_MTD_DEVICE
	/*
	 * Add MTD device so that we can reference it later
	 * via the mtdcore infrastructure (e.g. ubi).
	 */
	add_mtd_device(mtd);
#endif

	total_nand_size += mtd->size / 1024;

	if (nand_curr_device == -1)
		nand_curr_device = devnum;

	return 0;
}
```

##### should_load_env

发现 `CONFIG_OF_CONTROL`、`CONFIG_DELAY_ENVIRONMENT` 均没有定义，返回 1。

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	/* initialize environment */
	if (should_load_env())
		env_relocate();
	else
		set_default_env(NULL);
        ...
}
```

也就是 `env_relocate()`，由于 `CONFIG_NEEDS_MANUAL_RELOC`、`CONFIG_ENV_IS_NOWHERE` 没有定义

```c
void env_relocate(void)
{
	if (gd->env_valid == 0) {
#if defined(CONFIG_ENV_IS_NOWHERE) || defined(CONFIG_SPL_BUILD)
		/* Environment not changable */
		set_default_env(NULL);
#else
		bootstage_error(BOOTSTAGE_ID_NET_CHECKSUM);
		set_default_env("!bad CRC");
#endif
	} else {
		env_relocate_spec();
	}
}
```

所以代码可精简为如下, 由于在 [env_init](#env_init) 中配置为 1，所以此处走 `else` 分支。

```c
void env_relocate(void)
{
	if (gd->env_valid == 0) {
		bootstage_error(BOOTSTAGE_ID_NET_CHECKSUM);
		set_default_env("!bad CRC");
	} else {
		env_relocate_spec();
	}
}
```

由于上述我们使用的为 nand，所以此处 env_relocate_spec 位置在 `common/env_nand.c` 中，先看下这个又臭又长的宏定义。

```c
// common/env_nand.c
void env_relocate_spec(void)
{
	int ret;
	ALLOC_CACHE_ALIGN_BUFFER(char, buf, CONFIG_ENV_SIZE);
	ret = readenv(CONFIG_ENV_OFFSET, (u_char *)buf);
	if (ret) {
		set_default_env("!readenv() failed");
		return;
	}
	env_import(buf, 1);
}
```

看下下面的分析，其实也就是定义了一个数组，然后返回一个数组指针，只不过其中保证了内存对齐，

```c
// common/env_nand.c
ALLOC_CACHE_ALIGN_BUFFER(char, buf, CONFIG_ENV_SIZE);
//-------------------------------------------------------------------------------------------
// include/common.h
#define ALLOC_CACHE_ALIGN_BUFFER(type, name, size)	ALLOC_ALIGN_BUFFER(type, name, size, ARCH_DMA_MINALIGN)
...
#define ALLOC_ALIGN_BUFFER(type, name, size, align)	       ALLOC_ALIGN_BUFFER_PAD(type, name, size, align, 1)
...
#define ALLOC_ALIGN_BUFFER_PAD(type, name, size, align, pad)		\
	char __##name[ROUND(PAD_SIZE((size) * sizeof(type), pad), align)  \
		      + (align - 1)];					\
									\
	type *name = (type *) ALIGN((uintptr_t)__##name, align)
//---------------------------------------------------------------------------------------------
#define ALLOC_CACHE_ALIGN_BUFFER(type, name, size)  \
	char __##name[ROUND(PAD_SIZE((size) * sizeof(type), 1), ARCH_DMA_MINALIGN)  \
		      + (ARCH_DMA_MINALIGN - 1)];					\
	type *name = (type *) ALIGN((uintptr_t)__##name, ARCH_DMA_MINALIGN)
//------------------------------------------------------------------------------------------------
ALLOC_CACHE_ALIGN_BUFFER(char, buf, CONFIG_ENV_SIZE);
char __buf[CONFIG_ENV_SIZE];
char *buf = (char *)ALIGN((uintptr_t)__buf,ARCH_DMA_MINALIGN);
```

接着看 readenv 函数，在实际中打印

```shell
readenv
end of readenv,amount_loaded=131072 CONFIG_ENV_SIZE:131072
```

`nand_info[0].erasesize` 又是在哪赋值的呢？

在 `nand_init_chip` 中，没有与 `mtd->erasesize` 相关的，而 `board_nand_init` 仅与 `nand_chip` 有关，`nand_register` 中也没有与之有关的。

所以一路找下去，最终是在 `nand_get_flash_type` 中的 `nand_decode_id`（或者 `nand_decode_ext_id`）获取类型中给 `mtd->erasesize` 赋值的，具体值的大小与 nand 的厂商有关。

```c
--> nand_init
          |
	 --> nand_init_chip
	 		|
	 		--> nand_scan
	 			|
	 			--> nand_scan_ident
	 				|
	 				--> nand_get_flash_type
	 					|
	 					--> nand_decode_ext_id
					        --> nand_decode_id
```

从 `blocksize` 或者 `CONFIG_ENV_SIZE` 中获取最小值，接着一个循环 条件呢？就是总共加载的大小与配置的环境大小相比且当前的偏移小于配置的环境的大小。

先判断当前块是否是坏的，如果是，则跳过，`offset += blocksize`

否则，在里面接着以跳过坏块的方式读，并将读取的环境变量写入 `buf` 中，二者相等返回 0。

```c
int readenv(size_t offset, u_char *buf)
{
	size_t end = offset + CONFIG_ENV_RANGE;
	size_t amount_loaded = 0;
	size_t blocksize, len;
	u_char *char_ptr;

	debug("readenv\r\n");
	blocksize = nand_info[0].erasesize;
	if (!blocksize)
		return 1;

	len = min(blocksize, CONFIG_ENV_SIZE);

	while (amount_loaded < CONFIG_ENV_SIZE && offset < end) {
		if (nand_block_isbad(&nand_info[0], offset)) {
			offset += blocksize;
		} else {
			char_ptr = &buf[amount_loaded];
			if (nand_read_skip_bad(&nand_info[0], offset,
					       &len, NULL,
					       nand_info[0].size, char_ptr))
				return 1;

			offset += blocksize;
			amount_loaded += len;
		}
	}
	debug("end of readenv,amount_loaded=%d CONFIG_ENV_SIZE:%d\r\n",amount_loaded,CONFIG_ENV_SIZE);
	if (amount_loaded != CONFIG_ENV_SIZE)
		return 1;
	return 0;
}
```

返回到 `env_relocate_spec` 中，执行 `env_import(buf,1)` 这个函数是个难点，也是个重点，涉及到对哈希表的操作。

这个函数有两个操作，一个是 crc 校验；另一个是创建哈希表，并将环境变量中的数据提取出来插入表中。

```c
int env_import(const char *buf, int check)
{
	env_t *ep = (env_t *)buf;
	debug("env_import\r\n");
	if (check) {
		uint32_t crc;
		memcpy(&crc, &ep->crc, sizeof(crc));
		if (crc32(0, ep->data, ENV_SIZE) != crc) {
			debug("env_import --> crc32\r\n");
			set_default_env("!bad CRC");
			return 0;
		}
	}
	if (himport_r(&env_htab, (char *)ep->data, ENV_SIZE, '\0', 0,0, NULL)) {
		debug("env_import --> himport_r\r\n");
		gd->flags |= GD_FLG_ENV_READY;
		return 1;
	}

	error("Cannot import environment: errno = %d\n", errno);
	set_default_env("!import failed");
	return 0;
}
```

主要是分析下创建哈希表和哈希表的插入操作。**TODO：明天挑一段长的时间分析吧！**

```c
// common/env_common.c
himport_r(&env_htab, (char *)ep->data, ENV_SIZE, '\0', 0,0, NULL)
// lib/hashtable.c
int himport_r(struct hsearch_data *htab,const char *env, size_t size, const char sep, int flag,int nvars, char * const vars[])
{
	char *data, *sp, *dp, *name, *value;
	char *localvars[nvars];
	int i;

	/* Test for correct arguments.  */
	if (htab == NULL) {
		__set_errno(EINVAL);
		return 0;
	}

	/* we allocate new space to make sure we can write to the array */
	if ((data = malloc(size)) == NULL) {
		debug("himport_r: can't malloc %zu bytes\n", size);
		__set_errno(ENOMEM);
		return 0;
	}
	memcpy(data, env, size);
	dp = data;

	/* make a local copy of the list of variables */
	if (nvars)
		memcpy(localvars, vars, sizeof(vars[0]) * nvars);

	if ((flag & H_NOCLEAR) == 0) {
		/* Destroy old hash table if one exists */
		debug("Destroy Hash Table: %p table = %p\n", htab, htab->table);
		if (htab->table) hdestroy_r(htab);
	}

	if (!htab->table) {
		int nent = CONFIG_ENV_MIN_ENTRIES + size / 8;
		if (nent > CONFIG_ENV_MAX_ENTRIES) nent = CONFIG_ENV_MAX_ENTRIES;
		debug("Create Hash Table: N=%d\n", nent);
		if (hcreate_r(nent, htab) == 0) {
			free(data);
			return 0;
		}
	}

	/* Parse environment; allow for '\0' and 'sep' as separators */
	do {
		ENTRY e, *rv;
		/* skip leading white space */
		while (isblank(*dp))
			++dp;
		/* skip comment lines */
		if (*dp == '#') {
			while (*dp && (*dp != sep))
				++dp;
			++dp;
			continue;
		}

		/* parse name */
		for (name = dp; *dp != '=' && *dp && *dp != sep; ++dp)
			;

		/* deal with "name" and "name=" entries (delete var) */
		if (*dp == '\0' || *(dp + 1) == '\0' ||
		    *dp == sep || *(dp + 1) == sep) {
			if (*dp == '=')
				*dp++ = '\0';
			*dp++ = '\0';	/* terminate name */

			debug("DELETE CANDIDATE: \"%s\"\n", name);
			if (!drop_var_from_set(name, nvars, localvars))
				continue;

			if (hdelete_r(name, htab, flag) == 0)
				debug("DELETE ERROR ##############################\n");

			continue;
		}
		*dp++ = '\0';	/* terminate name */

		/* parse value; deal with escapes */
		for (value = sp = dp; *dp && (*dp != sep); ++dp) {
			if ((*dp == '\\') && *(dp + 1))
				++dp;
			*sp++ = *dp;
		}
		*sp++ = '\0';	/* terminate value */
		++dp;

		if (*name == 0) {
			debug("INSERT: unable to use an empty key\n");
			__set_errno(EINVAL);
			return 0;
		}

		/* Skip variables which are not supposed to be processed */
		if (!drop_var_from_set(name, nvars, localvars))
			continue;

		/* enter into hash table */
		e.key = name;
		e.data = value;

		hsearch_r(e, ENTER, &rv, htab, flag);
		if (rv == NULL)
			printf("himport_r: can't insert \"%s=%s\" into hash table\n",name, value);
		debug("INSERT: table %p, filled %d/%d rv %p ==> name=\"%s\" value=\"%s\"\n",htab, htab->filled, htab->size,rv, name, value);
	} while ((dp < data + size) && *dp);	/* size check needed for text */
						/* without '\0' termination */
	debug("INSERT: free(data = %p)\n", data);
	free(data);

	for (i = 0; i < nvars; i++) {
		if (localvars[i] == NULL)
			continue;
		if (hdelete_r(localvars[i], htab, flag) == 0)
			printf("WARNING: '%s' neither in running nor in imported env!\n", localvars[i]);
		else
			printf("WARNING: '%s' not in imported env, deleting it!\n", localvars[i]);
	}

	debug("INSERT: done\n");
	return 1;		/* everything OK */
}
```

先把打印的日志摆这，对着分析还容易点。。。。

```shell
Destroy Hash Table: 3ffb4f0c table = 00000000
Create Hash Table: N=512
INSERT: table 3ffb4f0c, filled 1/521 rv 3fe62868 ==> name="baudrate" value="115200"
INSERT: table 3ffb4f0c, filled 2/521 rv 3fe643fc ==> name="bootargs" value="root=/dev/nfs rw nfsroot=10.0.0.97:/home/acoollib/workspace/git/smart210-SDK/rootfs ip=10.0.0.98:10.0.0.97:10.0.0.1:255.0.0.0::eth0:off init=/linuxrc console=ttySAC0，115200"
INSERT: table 3ffb4f0c, filled 3/521 rv 3fe62c8c ==> name="bootcmd" value="nfs 20000000 10.0.0.97:/home/acoollib/workspace/git/smart210-SDK/rootfs/uImage;nfs 21000000 10.0.0.97:/home/acoollib/workspace/git/smart210-SDK/rootfs/s5pv210-smart210.dtb;bootm 20000000 - 21000000"
INSERT: table 3ffb4f0c, filled 4/521 rv 3fe634e8 ==> name="bootdelay" value="1"
INSERT: table 3ffb4f0c, filled 5/521 rv 3fe62bb0 ==> name="ethact" value="dm9000"
INSERT: table 3ffb4f0c, filled 6/521 rv 3fe625fc ==> name="ethaddr" value="1a:2a:3a:4a:5a:6a"
INSERT: table 3ffb4f0c, filled 7/521 rv 3fe64320 ==> name="ipaddr" value="10.0.0.98"
INSERT: table 3ffb4f0c, filled 8/521 rv 3fe6232c ==> name="machid" value="0xffffffff"
INSERT: table 3ffb4f0c, filled 9/521 rv 3fe63e48 ==> name="qqqqq" value="1111"
INSERT: table 3ffb4f0c, filled 10/521 rv 3fe62a84 ==> name="serverip" value="10.0.0.20"
INSERT: table 3ffb4f0c, filled 11/521 rv 3fe62fac ==> name="stderr" value="serial"
INSERT: table 3ffb4f0c, filled 12/521 rv 3fe627c8 ==> name="stdin" value="serial"
INSERT: table 3ffb4f0c, filled 13/521 rv 3fe638f8 ==> name="stdout" value="serial"
INSERT: free(data = 3fe41bd0)
INSERT: done
```

##### :negative_squared_cross_mark: stdio_init

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	stdio_init();	/* get the devices list going. */
    ...
}
```

因为一些宏在 smart210.h 中未定义，所以 此处留下会执行的代码。

```c
// common/stdio.c
int stdio_init (void)
{
	/* Initialize the list */
	INIT_LIST_HEAD(&(devs.list));
	drv_system_init ();
	serial_stdio_init ();
	return (0);
}
```

首先初始化列表，双向列表的头初始化时 其下一个和前一个节点均指向自身。此处尽管 `INIT_LIST_HEAD` 为 static 函数，但是其在 common.h 实现，在 c 中展开相当于仅在当前 c 中使用。再看下 `devs.list` 是什么东西？

```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

此处又引出一个结构体 `stdio_dev`，其中包括标志位、支持的扩展、设备名称、设备操作函数、私有扩展和一个双向列表。

```c
// common/stdio.c
static struct stdio_dev devs;
```

```c
// include/stdio_dev.h
/* Device information */
struct stdio_dev {
	int		flags;			 /* Device flags: input/output/system	*/
	int		ext;			/* Supported extensions			*/
        char	name[16];		/* Device name				*/

/* GENERAL functions */
	int (*start) (void);		 /* To start the device			*/
	int (*stop) (void);		       /* To stop the device			*/

/* OUTPUT functions */
	void (*putc) (const char c);	/* To put a char			*/
	void (*puts) (const char *s);	/* To put a string (accelerator)	*/

/* INPUT functions */
	int (*tstc) (void);		/* To test if a char is ready...	*/
	int (*getc) (void);		/* To get that char			*/

/* Other functions */
	void *priv;			/* Private extensions			*/
	struct list_head list;
}
```

接着看下 **drv_system_init**，其中定义了 `struct stdio_dev dev`，然后初始化。最后注册这个结构。直接看下是如何注册的？

```c
// common/stdio.c
static void drv_system_init (void)
{
	struct stdio_dev dev;
	memset (&dev, 0, sizeof (dev));
	strcpy (dev.name, "serial");
	dev.flags = DEV_FLAGS_OUTPUT | DEV_FLAGS_INPUT | DEV_FLAGS_SYSTEM;
	dev.putc = serial_putc;
	dev.puts = serial_puts;
	dev.getc = serial_getc;
	dev.tstc = serial_tstc;
	stdio_register (&dev);
}
```

在注册函数中，首先克隆一个当前的结构（先分配空间，在拷贝数据），然后把克隆出的结构体在挂到 `devs.list` 上（`TODO：此处待理解双向列表添加后在分析`）。

```c
// common/stdio.c
int stdio_register (struct stdio_dev * dev)
{
	struct stdio_dev *_dev;

	_dev = stdio_clone(dev);
	if(!_dev)
		return -1;
	list_add_tail(&(_dev->list), &(devs.list));
	return 0;
}
```

再看下 `serial_stdio_init`，`serial_devices` 不知各位看官姥爷还有没有印象，反正笔者是没有印象了。。。。好了，话说回来，在 [serial_init](#serial_init) 中我们分析了串口的初始化，其节点都挂到 `serial_devices` 列表中，`serial_current` 中保存着当前串口的指针。

这段代码的意思是 遍历 `serial_devices` 中的所有节点，将其添加到 `devs` 中去。

```c
void serial_stdio_init(void)
{
	struct stdio_dev dev;
	struct serial_device *s = serial_devices;
	while (s) {
		memset(&dev, 0, sizeof(dev));
		strcpy(dev.name, s->name);
		dev.flags = DEV_FLAGS_OUTPUT | DEV_FLAGS_INPUT;
		dev.start = s->start;
		dev.stop = s->stop;
		dev.putc = s->putc;
		dev.puts = s->puts;
		dev.getc = s->getc;
		dev.tstc = s->tstc;
		stdio_register(&dev);
		s = s->next;	// 移到下一个节点
	}
}
```

##### jumptable_init

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	jumptable_init();
    ...
}
```

在 `common/exports.c` 中实现

```c
// common/exports.c
void jumptable_init(void)
{
	gd->jt = malloc(XF_MAX * sizeof(void *));
#include <_exports.h>
}
```

手动模拟编译器展开（滑稽）看一下，此处只需要记住，宏直接展开。注意下在 `exports.c` 中包含 `exports.h`，而 `exports.h` 定义了一个 enum，那个 enum 展开后对应的恰是 `_exports.h` 中的所有函数的索引（`0` 开始，`XF_MAX` 结束，所以前面分配内存时可直接 `XF_MAX`）。所以在 `exports.c` 就可以直接使用索引访问数组并给每个数组变量赋值。至于这些个函数就不一一分析了。

```c
// common/exports.c
#define EXPORT_FUNC(sym) gd->jt[XF_##sym] = (void *)sym;
void jumptable_init(void)
{
	gd->jt = malloc(XF_MAX * sizeof(void *));
        EXPORT_FUNC(get_version)
        EXPORT_FUNC(getc)
        EXPORT_FUNC(tstc)
        EXPORT_FUNC(putc)
        EXPORT_FUNC(puts)
        EXPORT_FUNC(printf)
        EXPORT_FUNC(install_hdlr)
        EXPORT_FUNC(free_hdlr)
        ...
        EXPORT_FUNC(spi_xfer)
}
//--------------------------------------------------------------------------------------
void jumptable_init(void)
{
	gd->jt = malloc(XF_MAX * sizeof(void *));
    gd->jt[XF_getc] = (void *)get_c;
	gd->jt[XF_get_version] = (void *)get_version;
	gd->jt[XF_tstc] = (void *)tstc;
	gd->jt[XF_putc] = (void *)putc;
	gd->jt[XF_puts] = (void *)puts;
	gd->jt[XF_printf] = (void *)printf;
	gd->jt[XF_install_hdlr] = (void *)install_hdlr;
	gd->jt[XF_free_hdlr] = (void *)free_hdlr;
	...
    gd->jt[XF_spi_xfer] = (void *)spi_xfer;
}
```

由于头文件一般都在函数之前，所以此处定义 `EXPORT_FUNC(x)` 不会出错，待使用结束后 `undef`，所以后面使用时也没有问题。

```c
enum {
#define EXPORT_FUNC(x) XF_ ## x ,
#include <_exports.h>
#undef EXPORT_FUNC

	XF_MAX
};
```

##### console_init_r

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	console_init_r();	/* fully init console as a device */
    ...
}
```

看下 `CONFIG_SYS_CONSOLE_IS_IN_ENV` 宏是否定义，没有定义，那么代码就是下面这一段。

`list_for_each(pos, list)` 是一个迭代器，好吧，也就是一个 `for` 循环，但是注意里面有一个 `prefetch`，此处需分析一下。`TODO：待分析`

然后 `list_entry` 也需要分析一下。`TODO：待分析`

这个循环的目的，根据标志位判断输入输出节点，并将其赋值给 `inputdev`、`outputdev`，如果这两者都有的话，结束这个循环。

一个 `console_setfile` 将 dev 设置到 `stdio_devices[i]`，同时也设给 `console_devices[i][0]`。

```c
// common/console.c
int console_init_r(void)
{
	struct stdio_dev *inputdev = NULL, *outputdev = NULL;
	int i;
	struct list_head *list = stdio_get_list();
	struct list_head *pos;
	struct stdio_dev *dev;

	/* Scan devices looking for input and output devices */
	list_for_each(pos, list) {
		dev = list_entry(pos, struct stdio_dev, list);

		if ((dev->flags & DEV_FLAGS_INPUT) && (inputdev == NULL)) {
			inputdev = dev;
		}
		if ((dev->flags & DEV_FLAGS_OUTPUT) && (outputdev == NULL)) {
			outputdev = dev;
		}
		if(inputdev && outputdev)
			break;
	}

	/* Initializes output console first */
	if (outputdev != NULL) {
		console_setfile(stdout, outputdev);
		console_setfile(stderr, outputdev);
#ifdef CONFIG_CONSOLE_MUX
		console_devices[stdout][0] = outputdev;
		console_devices[stderr][0] = outputdev;
#endif
	}

	/* Initializes input console */
	if (inputdev != NULL) {
		console_setfile(stdin, inputdev);
#ifdef CONFIG_CONSOLE_MUX
		console_devices[stdin][0] = inputdev;
#endif
	}

#ifndef CONFIG_SYS_CONSOLE_INFO_QUIET
	stdio_print_current_devices();
#endif /* CONFIG_SYS_CONSOLE_INFO_QUIET */

	/* Setting environment variables */
	for (i = 0; i < 3; i++) {
		setenv(stdio_names[i], stdio_devices[i]->name);
	}

	gd->flags |= GD_FLG_DEVINIT;	/* device initialization completed */
	return 0;
}
```

然后调用 `stdio_print_current_devices` 打印，debug 输出如下。并设置环境变量 `stderr`、`stdin`、`stdout` 为 `serial`。

```c
In:    serial
Out:   serial
Err:   serial
```

最后一个设置标志位 `GD_FLG_DEVINIT` 表示 dev 初始化完成。

##### checkboard

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	checkboard();	/* fully init console as a device */
    ...
}
// board/samsung/smart210/smart210.c
int checkboard(void)
{
	printf("Board:\tSMART210\n");
	return 0;
}
```

##### :negative_squared_cross_mark: interrupt_init

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	/* set up exceptions */
	interrupt_init();
    ...
}
```

没有定义宏 `CONFIG_USE_IRQ`，走这个分支，这个代码也只是设置 `IRQ_STACK_START_IN` 中断栈起始地址为 `gd->irq_sp + 8`？不应该是 `-8` 吗？

```c
// arch/arm/lib/interrupts.c
int interrupt_init (void)
{
	IRQ_STACK_START_IN = gd->irq_sp + 8;
	return 0;
}
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204151531408.png)

##### enable_interrupts

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	/* enable exceptions */
	enable_interrupts();
    ...
}
```

空实现

```c
// arch/arm/lib/interrupts.c
void enable_interrupts (void)
{
	return;
}
```

##### eth_initialize

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
	...
	puts("Net:   ");
	eth_initialize(gd->bd);
    ...
}
```

类似串口，这个地方又有个数据结构 `struct eth_device`，定义了网络设备的基本操作。其中从面向对象的角度来说，在驱动层面只需要把对应的 init、send、recv、halt、weite_hwaddr 这几种方法实现，就可以正常收发数据了，至于数据收回来要怎么办，那是上层决定的，我们并不需要关心。

```c
// include/net.h
struct eth_device {
	char name[16];
	unsigned char enetaddr[6];
	int iobase;
	int state;

	int  (*init) (struct eth_device *, bd_t *);
	int  (*send) (struct eth_device *, void *packet, int length);
	int  (*recv) (struct eth_device *);
	void (*halt) (struct eth_device *);
#ifdef CONFIG_MCAST_TFTP
	int (*mcast) (struct eth_device *, const u8 *enetaddr, u8 set);
#endif
	int  (*write_hwaddr) (struct eth_device *);
	struct eth_device *next;
	int index;
	void *priv;
};
```

接着看 `eth_initialize` 函数，上来首先将 `eth_devices`（网卡设备链表）、`eth_current`（当前网卡设备）置空，然后标记当前阶段为 `BOOTSTAGE_ID_NET_ETH_START`，从环境变量查找 `"bootfile"`，如果有，则拷贝至 `BootFile` 中。

接着往下走，实际上是一个初始化函数，只不过分了几种情况，如果 `board-specific` 存在的话，就调用它；如果没有，调用 `CPU-specific` 那个。

本文中走的是第一个分支，调用的 `board_eth_init(bis)` 为 `board/samsung/smart210/smart210.c` 中的，我们分析一下。

```c
// net/eth.c
int eth_initialize(bd_t *bis)
{
	int num_devices = 0;
	eth_devices = NULL;
	eth_current = NULL;

	bootstage_mark(BOOTSTAGE_ID_NET_ETH_START);

	eth_env_init(bis);

	/*
	 * If board-specific initialization exists, call it.
	 * If not, call a CPU-specific one
	 */
	if (board_eth_init != __def_eth_init) {
		if (board_eth_init(bis) < 0)
			printf("Board Net Initialization Failed\n");
	} else if (cpu_eth_init != __def_eth_init) {
		if (cpu_eth_init(bis) < 0)
			printf("CPU Net Initialization Failed\n");
	} else
		printf("Net Initialization Skipped\n");
	...
}
```

先是调用的 `board_eth_init`，接着调用 `dm9000_initialize`，在这个函数里，就是将 `dm9000_info.netdev` 结构给初始化，并且从 EEPROM 中获取 MAC 地址，初始化函数，然后注册（在注册中，如果 `eth_devices` 网卡设备链表为空，则使 `eth_devices`、`eth_devices` 指向当前设备 dev，并且如果当前网卡存在则设置环境变量中 `ethact` 的名称为当前网卡名，也就是说如果手动更改环境变量中 `ethact` 的 value，即使保存后，下次重启后 `ethact` 为前面设置的 `dm9000`。接着就是将网卡状态置位，将 `next` 指针指向自己，索引自加）。

```c
// board/samsung/smart210/smart210.c
int board_eth_init(bd_t *bis)
{
	int rc = 0;
	rc = dm9000_initialize(bis);
	return rc;
}
//----------------------------------------------------------------------------------
// drivers/net/dm9000x.c
int dm9000_initialize(bd_t *bis)
{
	struct eth_device *dev = &(dm9000_info.netdev);

	/* Load MAC address from EEPROM */
	dm9000_get_enetaddr(dev);

	dev->init = dm9000_init;
	dev->halt = dm9000_halt;
	dev->send = dm9000_send;
	dev->recv = dm9000_rx;
	sprintf(dev->name, "dm9000");

	eth_register(dev);

	return 0;
}
```

接着我们返回到 `eth_initialize`，看一下剩下的部分。从 `eth_devices` 中取出，从环境变量中取出 `ethprime` 的 value（应该为空，因为环境变量中没有此项），标记一下当前阶段。接着就是如果有多个网卡，则在显示的时候中间用 `,` 分开，比如：`Net: dm9000,dm9001`。然后 `eth_write_hwaddr` 写网卡的 MAC 地址到 dev 中，接着更新下一个，但是我们刚刚设置 dev->next 指向自己，所以条件不满足，退出循环，返回网卡的个数。

```c
// net/eth.c
int eth_initialize(bd_t *bis)
{
	...
	if (!eth_devices) {
		puts("No ethernet found.\n");
		bootstage_error(BOOTSTAGE_ID_NET_ETH_START);
	} else {
		struct eth_device *dev = eth_devices;
		char *ethprime = getenv("ethprime");

		bootstage_mark(BOOTSTAGE_ID_NET_ETH_INIT);
		do {
			if (dev->index)
				puts(", ");
			printf("%s", dev->name);

			if (ethprime && strcmp(dev->name, ethprime) == 0) {
				eth_current = dev;
				puts(" [PRIME]");
			}

			if (strchr(dev->name, ' '))
				puts("\nWarning: eth device name has a space!" "\n");

			if (eth_write_hwaddr(dev, "eth", dev->index))
				puts("\nWarning: failed to set MAC address\n");

			dev = dev->next;
			num_devices++;
		} while (dev != eth_devices);

		eth_current_changed();
		putc('\n');
	}
	return num_devices;
}
```

##### main_loop

终于到最后一个函数了，

```c
// arch/arm/lib/board.c
void board_init_r(gd_t *id, ulong dest_addr)
{
    ...
	/* main_loop() can return to retry autoboot, if so just run it again. */
	for (;;) {
		main_loop();
	}
	/* NOTREACHED - no way out of command loop except booting */
}
```

已定义宏 `CONFIG_SYS_HUSH_PARSER`、`CONFIG_BOOTDELAY` 精简后代码如下，首先标记当前阶段 `BOOTSTAGE_ID_MAIN_LOOP`，接着看一下这几个函数。

```c
// common/main.c
void main_loop(void)
{
	bootstage_mark_name(BOOTSTAGE_ID_MAIN_LOOP, "main_loop");
	u_boot_hush_start();
	process_boot_delay();
	/*
	 * Main Loop for Monitor Command Processing
	 */
	parse_file_outer();
	/* This point is never reached */
	for (;;);
}
```

###### u_boot_hush_start

`u_boot_hush_start` 在 `common/hush.c` 中，给 `top_vars` 分配空间，然后初始化

```c
// common/hush.c
int u_boot_hush_start(void)
{
	if (top_vars == NULL) {
		top_vars = malloc(sizeof(struct variables));
		top_vars->name = "HUSH_VERSION";
		top_vars->value = "0.01";
		top_vars->next = NULL;
		top_vars->flg_export = 0;
		top_vars->flg_read_only = 1;
	}
	return 0;
}
```

###### process_boot_delay

`process_boot_delay` 在 `common/main.c` 中，从环境变量中获取 `bootdelay` 对应的 value，获取 `bootcmd` 对应的 value，然后进入 `abortboot(bootdelay)` 函数，

```c
// common/main.c
static void process_boot_delay(void)
{
	char *s;
	int bootdelay;
	s = getenv ("bootdelay");
	bootdelay = s ? (int)simple_strtol(s, NULL, 10) : CONFIG_BOOTDELAY;
	debug ("### main_loop entered: bootdelay=%d\n", bootdelay);
	s = getenv ("bootcmd");
	debug ("### main_loop: bootcmd=\"%s\"\n", s ? s : "<UNDEFINED>");
	if (bootdelay != -1 && s && !abortboot(bootdelay)) {
		run_command_list(s, -1, 0);
	}
}
```

`abortboot` 处理延迟逻辑

```c
// common/main.c
static int abortboot(int bootdelay)
{
	return abortboot_normal(bootdelay);
}
```

如果获取到字符，就停止计时，否则一直在此空转，`bootdelay` 秒后，函数返回。

```c
// common/main.c
static int abortboot_normal(int bootdelay)
{
	int abort = 0;
	unsigned long ts;

	if (bootdelay >= 0)
		printf("Hit any key to stop autoboot: %2d ", bootdelay);

	/*
	 * Check if key already pressed
	 * Don't check if bootdelay < 0
	 */
	if (bootdelay >= 0) {
		if (tstc()) {	/* we got a key press	*/
			(void) getc();  /* consume input	*/
			puts ("\b\b\b 0");
			abort = 1;	/* don't auto boot	*/
		}
	}

	while ((bootdelay > 0) && (!abort)) {
		--bootdelay;
		/* delay 1000 ms */
		ts = get_timer(0);
		do {
			if (tstc()) {	/* we got a key press	*/
				abort  = 1;	/* don't auto boot	*/
				bootdelay = 0;	/* no more delay	*/
				(void) getc();  /* consume input	*/
				break;
			}
			udelay(10000);
		} while (!abort && get_timer(ts) < 1000);
		printf("\b\b\b%2d ", bootdelay);
	}
	putc('\n');
	return abort;
}
```

然后运行 `run_command_list(s, -1, 0);`，此时 s 中数据为 `bootcmd="nfs 20000000 10.0.0.97:/home/acoollib/workspace/git/smart210-SDK/rootfs/uImage;nfs 21000000 10.0.0.97:/home/acoollib/workspace/git/smart210-SDK/rootfs/s5pv210-smart210.dtb;bootm 20000000 - 21000000"`

###### run_command_list

```c
int run_command_list(const char *cmd, int len, int flag)
{
	int need_buff = 1;
	char *buff = (char *)cmd;	/* cast away const */
	int rcode = 0;

	if (len == -1) {
		len = strlen(cmd);
		/* hush will never change our string */
		need_buff = 0;
	}
	rcode = parse_string_outer(buff, FLAG_PARSE_SEMICOLON);// 1<<1
	return rcode;
}
```

`run_command_list` 只是对 `hush shell` 中的函数 `parse_string_outer` 进行了一层封装。`parse_string_outer` 函数调用了 `hush shell` 的命令解释器 `parse_stream_outer` 函数来解释 bootcmd 的命令。

###### parse_string_outer

我们看下 `parse_stream_outer` 这个执行过程实在太复杂了。从这开始整个执行流程是这样的，最后就跳转到调用的地方了。

```c
--> parse_string_outer
    |
    --> parse_stream_outer
    	|
    	--> parse_stream
    	--> run_list
    		|
    		--> run_list_real
    			|
    			--> run_pipe_real
    				|
    				--> cmd_process
    					|
    					--> cmd_call
```

~~这个过程需要理解一系列的代码以及各种神奇操作，让我们先省略这一段继续往后吧。~~

最后调用 `parse_string_outer`，在源码中，我们定义了 `__U_BOOT__` 宏，所以 `parse_string_outer` 源码如下

首先进入代码后先判空，然后搜索 `\n`，如果以 `\n` 结尾，~~会走 if 分支，否则走 else 分支~~，其区别就是是否需要分配空间，拷贝一下，添加 `\n` 到尾部。

然后转换一下格式，在 `setup_string_in_str` 中将其转为结构体 `struct in_str`，然后进入 `parse_stream_outer`，~~此处 hush 解析器不在分析。~~还是得分析。

```c
// common/hush.c
int parse_string_outer(const char *s, int flag)
{
	struct in_str input;
	char *p = NULL;
	int rcode;
	if ( !s || !*s)
		return 1;
	if (!(p = strchr(s, '\n')) || *++p) {
		p = xmalloc(strlen(s) + 2);
		strcpy(p, s);
		strcat(p, "\n");
		setup_string_in_str(&input, p);
		rcode = parse_stream_outer(&input, flag);
		free(p);
		return rcode;
	} else {
		setup_string_in_str(&input, s);
		return parse_stream_outer(&input, flag);
	}
}
```

###### parse_stream_outer

先进入 `parse_stream` 函数对数据进行处理，识别到执行命令后，最后进入 `run_list` 函数，`parse_stream` 和 `run_list` 中间的数据是怎么衔接的呢？借助 `ctx`，一会看一下 `ctx`（`p_context`）的结构。

```c
// common/hush.c
static int parse_stream_outer(struct in_str *inp, int flag)
{
	struct p_context ctx;
	o_string temp=NULL_O_STRING;
	int rcode;
	int code = 0;
	do {
		ctx.type = flag;
		initialize_context(&ctx);
		update_ifs_map();
		if (!(flag & FLAG_PARSE_SEMICOLON) || (flag & FLAG_REPARSING)) mapset((uchar *)";$&|", 0);
		inp->promptmode=1;
		rcode = parse_stream(&temp, &ctx, inp, '\n');
		if (rcode == 1) flag_repeat = 0;
		if (rcode != 1 && ctx.old_flag != 0) {
			syntax();
			flag_repeat = 0;
		}
		if (rcode != 1 && ctx.old_flag == 0) {
			done_word(&temp, &ctx);
			done_pipe(&ctx,PIPE_SEQ);
			code = run_list(ctx.list_head);
			if (code == -2) {	/* exit */
				b_free(&temp);
				code = 0;
				/* XXX hackish way to not allow exit from main loop */
				if (inp->peek == file_peek) {
					printf("exit not allowed from main input shell.\n");
					continue;
				}
				break;
			}
			if (code == -1)
			    flag_repeat = 0;
		} else {
			if (ctx.old_flag != 0) {
				free(ctx.stack);
				b_reset(&temp);
			}
			if (inp->__promptme == 0) printf("<INTERRUPT>\n");
			inp->__promptme = 1;
			temp.nonnull = 0;
			temp.quote = 0;
			inp->p = NULL;
			free_pipe_list(ctx.list_head,0);
		}
		b_free(&temp);
	} while (rcode != -1 && !(flag & FLAG_EXIT_FROM_LOOP));   /* loop on syntax errors, return on EOF */
	return (code != 0) ? 1 : 0;
}
```

###### :negative_squared_cross_mark: p_context

```c
/* This holds pointers to the various results of parsing */
struct p_context {
	struct child_prog *child;
	struct pipe *list_head;
	struct pipe *pipe;
#ifndef __U_BOOT__
	struct redir_struct *pending_redirect;
#endif
	reserved_style w;
	int old_flag;				/* for figuring out valid reserved words */
	struct p_context *stack;
	int type;			/* define type of parser : ";$" common or special symbol */
	/* How about quoting status? */
};
```

###### run_list

由于定义了 `__U_BOOT__`，所以此处会直接运行 `rcode = run_list_real(pi);`，进入到 `run_list_real` 函数。

```c
static int run_list(struct pipe *pi)
{
	int rcode=0;
#ifndef __U_BOOT__
	if (fake_mode==0) {
#endif
		rcode = run_list_real(pi);
		debug("rcode:%d\n",rcode);
#ifndef __U_BOOT__
	}
#endif
	/* free_pipe_list has the side effect of clearing memory
	 * In the long run that function can be merged with run_list_real,
	 * but doing that now would hobble the debugging effort. */
	free_pipe_list(pi,0);
	return rcode;
}
```

###### :negative_squared_cross_mark: run_list_real

**TODO：待分析**

只会运行一次，且不会返回。

跳转到 `rcode = run_pipe_real(pi);`

###### :negative_squared_cross_mark: run_pipe_real

先看一下 `struct pipe` 是什么东西，在 u-boot 中有 `__U_BOOT__` 宏，去掉了相关无用的代码。

此处看一下 busybox 中的 [hush_doc.txt](https://git.busybox.net/busybox/tree/shell/hush_doc.txt) 中的示例就明白了。

```c
struct pipe {
	int num_progs;				/* 在当前任务流中的命令总数 */
	struct child_prog *progs;		/* 在管道中的命令数组 */
	struct pipe *next;			     /* 下一个命令的指针 */
	pipe_style followup;		    /* 表示这一截管道的类型 PIPE_BG, PIPE_SEQ, PIPE_OR, PIPE_AND；比如a && b，则此处应为PIPE_AND */
	reserved_style r_mode;		  /* 表示控制流 比如if,for, while, until这种 */
};
```

解析管道中的命令，处理条件语句，比如 IF、THEN、ELSE 等。

###### cmd_process

此处函数在重启时会执行三次，因为在 bootcmd 启动命令中有三段。

`nfs 20000000 10.0.0.97:/home/glj0/worksapce/os/smart210/rootfs/uImage;`

`nfs 21000000 10.0.0.97:/home/glj0/worksapce/os/smart210/rootfs/s5pv210-smart210.dtb;`

`bootm 20000000 - 21000000`

这个函数会在之前定义的 boot_list 中找命令对应的函数，比如 nfs、bootm 等，找到后将其填充到 `cmd_tbl_t *cmdtp` 中，然后跳入 `cmd_call` 中

```c
enum command_ret_t cmd_process(int flag, int argc, char * const argv[],
			       int *repeatable, ulong *ticks)
{
	enum command_ret_t rc = CMD_RET_SUCCESS;
	cmd_tbl_t *cmdtp;
	debug("flag:%d argc:%d\r\n",flag,argc);
	int i=0;
	for(;i<argc;i++){
		printf("argv[%d]:%s\r\n",i,argv[i]);
	}
	/* Look up command in command table */
	cmdtp = find_cmd(argv[0]);
	if (cmdtp == NULL) {
		printf("Unknown command '%s' - try 'help'\n", argv[0]);
		return 1;
	}

	/* found - check max args */
	if (argc > cmdtp->maxargs)
		rc = CMD_RET_USAGE;

#if defined(CONFIG_CMD_BOOTD)
	/* avoid "bootd" recursion */
	else if (cmdtp->cmd == do_bootd) {
		if (flag & CMD_FLAG_BOOTD) {
			puts("'bootd' recursion detected\n");
			rc = CMD_RET_FAILURE;
		} else {
			flag |= CMD_FLAG_BOOTD;
		}
	}
#endif

	/* If OK so far, then do the command */
	if (!rc) {
		if (ticks)
			*ticks = get_timer(0);
		debug("cmd_call cmdtp->name:%s\r\n",cmdtp->name);
		rc = cmd_call(cmdtp, flag, argc, argv);
		if (ticks)
			*ticks = get_timer(*ticks);
		*repeatable &= cmdtp->repeatable;
	}
	if (rc == CMD_RET_USAGE)
		rc = cmd_usage(cmdtp);
	return rc;
}
```

###### cmd_call

```c
static int cmd_call(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	int result;
	debug("cmdtp:%p flag:%d argc:%d\r\n",cmdtp,flag,argc);
	debug("cmdtp->name:%s maxargs:%d cmd:%p usage:%s \r\n",cmdtp->name,cmdtp->maxargs,cmdtp->cmd,cmdtp->usage);
	result = (cmdtp->cmd)(cmdtp, flag, argc, argv);
	if (result)
		debug("Command failed, result=%d", result);
	return result;
}
```

首先看下是怎么调用的，最终在 cmd_call 中完成调用，最关键的一句 `result = (cmdtp->cmd)(cmdtp, flag, argc, argv);` 此时并不能发现什么猫腻，无法就是 `cmdtp->cmd` 是一个函数地址，后面这一堆是参数，那么，这个地址指向的是什么呢？肯定是一个 `函数`。由于我们 DEBUG 模式下加了许多打印，所以可以轻松得到这个地址值为 `3ff8b590`，前面我们有个重定位还有印象吗？没有印象去查看一下 [relocate_code](#relocate_code)，`代码的地址：编译后函数的地址+重定位偏移量` 接着查日志中，发现 relocate 偏移量为 `1ff7b000`，那么这个函数偏移前地址也就是 `3ff8b590-1ff7b000=20010590`，看一下 u-boot.map 文件下该地址对应的是什么，对应的是 `do_nfs`。![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204211947823.png)

查一下该符号 `grep -nr do_nfs`，发现存在该函数。那么我们就可以理解了，在 `cmd_call` 之前通过一系列的骚操作，得到 `某指令` 对应的 `do_某指令` 函数地址，然后跳转到这个函数地址就可以进行后面的操作了。![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204211950205.png)

在 do_nfs 附近有一个 `U_BOOT_CMD ...`

```c
static int do_nfs(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	return netboot_common(NFS, cmdtp, argc, argv);
}

U_BOOT_CMD(
	nfs,	3,	1,	do_nfs,
	"boot image via network using NFS protocol",
	"[loadAddress] [[hostIPaddr:]bootfilename]"
);
```

该宏在 `include/command.h` 定义

```c
// include/command.h
/**
    　各个参数的意义如下：
       _name：命令名，非字符串，但在U_BOOT_CMD中用“#”符号转化为字符串
       _maxargs：命令的最大参数个数
       _rep：是否自动重复（按Enter键是否会重复执行）
       _cmd：该命令对应的响应函数指针
       _usage：简短的使用说明（字符串）
       _help：较详细的使用说明（字符串）
       */
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)	U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, NULL)

...
#define U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, _comp) \
	ll_entry_declare(cmd_tbl_t, _name, cmd) =			\
		U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,_usage, _help, _comp);
...
#define U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,_usage, _help, _comp)			\
		{ #_name, _maxargs, _rep, _cmd, _usage,	_CMD_HELP(_help) _CMD_COMPLETE(_comp) }
--------------------------------------------------------------------------------------------------------------------
// 手动替换 U_BOOT_CMD_COMPLETE 以及 U_BOOT_CMD_MKENT_COMPLETE
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)	\
	ll_entry_declare(cmd_tbl_t, _name, cmd) = { #_name, _maxargs, _rep, _cmd, _usage,_help ,NULL }
--------------------------------------------------------------------------------------------------------------------
// include/linker_lists.h
// ll_entry_declare 在 include/linker_lists.h 定义
#define ll_entry_declare(_type, _name, _list)				\
	_type _u_boot_list_2_##_list##_2_##_name __aligned(4) __attribute__((unused,section(".u_boot_list_2_"#_list"_2_"#_name)))
--------------------------------------------------------------------------------------------------------------------
// 示例 -- 手动解释这段宏
 U_BOOT_CMD(
	nfs,	3,	1,	do_nfs,
	"boot image via network using NFS protocol",
	"[loadAddress] [[hostIPaddr:]bootfilename]"
);
// 替换后也就是
cmd_tbl_t  _u_boot_list_2_cmd_2_nfs __aligned(4) __attribute__((unused,section(".u_boot_list_2_cmd_2_nfs"))) = \
	{nfs,3,1,do_nfs,"boot image via network using NFS protocol","[loadAddress] [[hostIPaddr:]bootfilename]"}
```

`cmdtp->cmd` 我们知道是什么了，`cmdtp` 呢？看下地址：`3ffb8f64`（通过日志查看）在减去重定位偏移 `1ff7b000`，也就是 `2003DF64`，通过在 `u-boot.map` 中查看对应地址 ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204220931044.png)

发现其也就是我们上面使用 `U_BOOT_CMD` 定义的这一串，而这一串的格式也恰恰与 `struct cmd_tbl_s` 一致。![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204220934517.png)

```c
struct cmd_tbl_s {
	char	*name;		/* 指令名称 */
	int		maxargs;	 /* 命令的最大参数个数*/
	int		repeatable;	    /* 是否自动重复		*/
					    /* Implementation function	*/
	int		(*cmd)(struct cmd_tbl_s *, int, int, char * const []);
	char		*usage;		/* 简短的使用说明 */
#ifdef	CONFIG_SYS_LONGHELP
	char		*help;		/* 较详细的使用说明 */
#endif
#ifdef CONFIG_AUTO_COMPLETE
	/* do auto completion on the arguments */
	int		(*complete)(int argc, char * const argv[], char last_char, int maxv, char *cmdv[]);
#endif
};
```

###### do_nfs

前面我们知道了在 `cmd_call` 中会调用 `do_nfs` 函数，接下来就分析一下该函数。

```shell
 do_nfs
	|
	--> netboot_common
		|
		--> NetLoop
			|
			--> eth_halt
				|
				-->  dm9000_halt
				|
		     	   eth_set_current
              		     eth_init
				|
				--> dm9000_init
              				|
              				 --> NfsStart
              			      		|
              			      		-->  Nfs_Send
              			      	--> eth_rx
						|
						--> dm9000_rx
```

根据日志，我们接着分析。先分析这一段 ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204221402853.png)

通过上面调用关系，直接看 `netboot_common`，在这个函数中，上面首先是各种初始化，根据参数的不同设定对应的 `load_addr`，这部分我们先不看，在函数的中间有一个 `NetLoop` 函数，其中参数为协议类型，此处也就是 nfs

```c
// common/cmd_net.c
static int netboot_common(enum proto_t proto, cmd_tbl_t *cmdtp, int argc,
		char * const argv[])
{
	...
	if ((size = NetLoop(proto)) < 0) {
		bootstage_error(BOOTSTAGE_ID_NET_NETLOOP_OK);
		return 1;
	}
	...
}
```

由于 `eth_is_on_demand_init` 返回值为 1，所以会进入 if 分支下，然后 `eth_halt` 也就是调用 `eth_current->halt(eth_current)`

接着 `eth_set_current`：在环境变量中找到 ethact 的名字，在 `eth_current` 链表中查找该设备，找到就退出

```c
// net/net.c
int NetLoop(enum proto_t protocol)
{
	bd_t *bd = gd->bd;
	int ret = -1;

	NetRestarted = 0;
	NetDevExists = 0;
	NetTryCount = 1;
	debug_cond(DEBUG_INT_STATE, "--- NetLoop Entry\n");

	bootstage_mark_name(BOOTSTAGE_ID_ETH_START, "eth_start");
	net_init();
	if (eth_is_on_demand_init() || protocol != NETCONS) {
		eth_halt();
		eth_set_current();
		if (eth_init(bd) < 0) {
			eth_halt();
			return -1;
		}
	} else
		eth_init_state_only(bd);
    ...
}
```

在 `eth_init` 中，先是打印 `Trying ...`，然后调用 `eth_current->init(eth_current, bis)`，这个 init 函数调到什么地方去了呢？

```c
int eth_init(bd_t *bis)
{
   	 ...
	do {
		debug("Trying %s\n", eth_current->name);

		if (eth_current->init(eth_current, bis) >= 0) {
			eth_current->state = ETH_STATE_ACTIVE;

			return 0;
		}
		debug("FAIL\n");
		eth_try_another(0);
	} while (old_current != eth_current);
	return -1;
}
```

不知各位看官还有印象没，我们在前面 board_init_r 中的 [eth_initialize](#eth_initialize) 初始化的时候，会调用到

`int dm9000_initialize(bd_t *bis)`，在此函数中，绑定了针对 dm9000 芯片的操作，然后注册到了 `eth_devices`，所以我们刚刚调用 `eth_current->init(eth_current, bis)` 就是直接执行 `dm9000_init(struct eth_device *dev, bd_t *bd)`，所以日志中也就会这些输出了。

```c
static int dm9000_init(struct eth_device *dev, bd_t *bd)
{
	int i, oft, lnk;
	u8 io_mode;
	struct board_info *db = &dm9000_info;

	DM9000_DBG("%s\n", __func__);

	/* RESET device */
	dm9000_reset();

	if (dm9000_probe() < 0)
		return -1;

	/* Auto-detect 8/16/32 bit mode, ISR Bit 6+7 indicate bus width */
	io_mode = DM9000_ior(DM9000_ISR) >> 6;

	switch (io_mode) {
	case 0x0:  /* 16-bit mode */
		printf("DM9000: running in 16 bit mode\n");
		db->outblk    = dm9000_outblk_16bit;
		db->inblk     = dm9000_inblk_16bit;
		db->rx_status = dm9000_rx_status_16bit;
		break;
	case 0x01:  /* 32-bit mode */
		printf("DM9000: running in 32 bit mode\n");
		db->outblk    = dm9000_outblk_32bit;
		db->inblk     = dm9000_inblk_32bit;
		db->rx_status = dm9000_rx_status_32bit;
		break;
	case 0x02: /* 8 bit mode */
		printf("DM9000: running in 8 bit mode\n");
		db->outblk    = dm9000_outblk_8bit;
		db->inblk     = dm9000_inblk_8bit;
		db->rx_status = dm9000_rx_status_8bit;
		break;
	default:
		/* Assume 8 bit mode, will probably not work anyway */
		printf("DM9000: Undefined IO-mode:0x%x\n", io_mode);
		db->outblk    = dm9000_outblk_8bit;
		db->inblk     = dm9000_inblk_8bit;
		db->rx_status = dm9000_rx_status_8bit;
		break;
	}

	/* Program operating register, only internal phy supported */
	DM9000_iow(DM9000_NCR, 0x0);
	/* TX Polling clear */
	DM9000_iow(DM9000_TCR, 0);
	/* Less 3Kb, 200us */
	DM9000_iow(DM9000_BPTR, BPTR_BPHW(3) | BPTR_JPT_600US);
	/* Flow Control : High/Low Water */
	DM9000_iow(DM9000_FCTR, FCTR_HWOT(3) | FCTR_LWOT(8));
	/* SH FIXME: This looks strange! Flow Control */
	DM9000_iow(DM9000_FCR, 0x0);
	/* Special Mode */
	DM9000_iow(DM9000_SMCR, 0);
	/* clear TX status */
	DM9000_iow(DM9000_NSR, NSR_WAKEST | NSR_TX2END | NSR_TX1END);
	/* Clear interrupt status */
	DM9000_iow(DM9000_ISR, ISR_ROOS | ISR_ROS | ISR_PTS | ISR_PRS);

	printf("MAC: %pM\n", dev->enetaddr);
	if (!is_valid_ether_addr(dev->enetaddr)) {
#ifdef CONFIG_RANDOM_MACADDR
		printf("Bad MAC address (uninitialized EEPROM?), randomizing\n");
		eth_random_enetaddr(dev->enetaddr);
		printf("MAC: %pM\n", dev->enetaddr);
#else
		printf("WARNING: Bad MAC address (uninitialized EEPROM?)\n");
#endif
	}

	/* fill device MAC address registers */
	for (i = 0, oft = DM9000_PAR; i < 6; i++, oft++)
		DM9000_iow(oft, dev->enetaddr[i]);
	for (i = 0, oft = 0x16; i < 8; i++, oft++)
		DM9000_iow(oft, 0xff);

	/* read back mac, just to be sure */
	for (i = 0, oft = 0x10; i < 6; i++, oft++)
		DM9000_DBG("%02x:", DM9000_ior(oft));
	DM9000_DBG("\n");

	/* Activate DM9000 */
	/* RX enable */
	DM9000_iow(DM9000_RCR, RCR_DIS_LONG | RCR_DIS_CRC | RCR_RXEN);
	/* Enable TX/RX interrupt mask */
	DM9000_iow(DM9000_IMR, IMR_PAR);

	i = 0;
	while (!(dm9000_phy_read(1) & 0x20)) {	/* autonegation complete bit */
		udelay(1000);
		i++;
		if (i == 10000) {
			printf("could not establish link\n");
			return 0;
		}
	}

	/* see what we've got */
	lnk = dm9000_phy_read(17) >> 12;
	printf("operating at ");
	switch (lnk) {
	case 1:
		printf("10M half duplex ");
		break;
	case 2:
		printf("10M full duplex ");
		break;
	case 4:
		printf("100M half duplex ");
		break;
	case 8:
		printf("100M full duplex ");
		break;
	default:
		printf("unknown: %d ", lnk);
		break;
	}
	printf("mode\n");
	return 0;
}
```

接着看下 `NfsStart`，首先是获取一些关键参数并打印。设置 nfs 的一些关键参数，比如要传输的地址，超时等等，设置回调函数 `NfsHandler`，然后 `NfsSend` 发送请求。

```c
// net/nfs.c
void NfsStart(void)
{
	debug("%s\n", __func__);
	nfs_download_state = NETLOOP_FAIL;

	NfsServerIP = NetServerIP;
	nfs_path = (char *)nfs_path_buff;

	if (nfs_path == NULL) {
		net_set_state(NETLOOP_FAIL);
		puts("*** ERROR: Fail allocate memory\n");
		return;
	}

	if (BootFile[0] == '\0') {
		sprintf(default_filename, "/nfsroot/%02X%02X%02X%02X.img",NetOurIP & 0xFF,(NetOurIP >>  8) & 0xFF,(NetOurIP >> 16) & 0xFF,(NetOurIP >> 24) & 0xFF);
		strcpy(nfs_path, default_filename);
		printf("*** Warning: no boot file name; using '%s'\n",nfs_path);
	} else {
		char *p = BootFile;
		p = strchr(p, ':');
		if (p != NULL) {
			NfsServerIP = string_to_ip(BootFile);
			++p;
			strcpy(nfs_path, p);
		} else {
			strcpy(nfs_path, BootFile);
		}
	}

	nfs_filename = basename(nfs_path);
	nfs_path     = dirname(nfs_path);

	printf("Using %s device\n", eth_get_name());
	printf("File transfer via NFS from server %pI4" "; our IP address is %pI4", &NfsServerIP, &NetOurIP);

	/* Check if we need to send across this subnet */
	if (NetOurGatewayIP && NetOurSubnetMask) {
		IPaddr_t OurNet	    = NetOurIP	  & NetOurSubnetMask;
		IPaddr_t ServerNet  = NetServerIP & NetOurSubnetMask;

		if (OurNet != ServerNet)
			printf("; sending through gateway %pI4",&NetOurGatewayIP);
	}
	printf("\nFilename '%s/%s'.", nfs_path, nfs_filename);

	if (NetBootFileSize) {
		printf(" Size is 0x%x Bytes = ", NetBootFileSize<<9);
		print_size(NetBootFileSize<<9, "");
	}
	printf("\nLoad address: 0x%lx\n" "Loading: *\b", load_addr);

	NetSetTimeout(nfs_timeout, NfsTimeout);
	net_set_udp_handler(NfsHandler);

	NfsTimeoutCount = 0;
	NfsState = STATE_PRCLOOKUP_PROG_MOUNT_REQ;

	/*NfsOurPort = 4096 + (get_ticks() % 3072);*/
	/*FIX ME !!!*/
	NfsOurPort = 1000;

	/* zero out server ether in case the server ip has changed */
	memset(NetServerEther, 0, 6);
	NfsSend();
}
```

接着就是 `eth_rx`，由于我们前面已经初始化过 `dev->recv = dm9000_rx;` 所以这个函数中返回也就是调用我们前面初始化的网卡，DM9000 的 `dm9000_rx` 函数。

也就是下面那一块，在这个里面完成数据的接收。如要查看接收的数据，可以打开 `CONFIG_DM9000_DEBUG` 开关。

```c
// net/eth.c
int eth_rx(void)
{
	if (!eth_current)
		return -1;

	return eth_current->recv(eth_current);
}

// drivers/net/dm9000x.c
static int dm9000_rx(struct eth_device *netdev)
{
	u8 rxbyte, *rdptr = (u8 *) NetRxPackets[0];
	u16 RxStatus, RxLen = 0;
	struct board_info *db = &dm9000_info;
	/* Check packet ready or not, we must check
	   the ISR status first for DM9000A */
	if (!(DM9000_ior(DM9000_ISR) & 0x01)) /* Rx-ISR bit must be set. */
		return 0;

	DM9000_iow(DM9000_ISR, 0x01); /* clear PR status latched in bit 0 */

	/* There is _at least_ 1 package in the fifo, read them all */
	for (;;) {
		DM9000_ior(DM9000_MRCMDX);	/* Dummy read */

		/* Get most updated data,
		   only look at bits 0:1, See application notes DM9000 */
		rxbyte = DM9000_inb(DM9000_DATA) & 0x03;

		/* Status check: this byte must be 0 or 1 */
		if (rxbyte > DM9000_PKT_RDY) {
			DM9000_iow(DM9000_RCR, 0x00);	/* Stop Device */
			DM9000_iow(DM9000_ISR, 0x80);	/* Stop INT request */
			printf("DM9000 error: status check fail: 0x%x\n",rxbyte);
			return 0;
		}

		if (rxbyte != DM9000_PKT_RDY)
			return 0; /* No packet received, ignore */

		DM9000_DBG("receiving packet\n");

		/* A packet ready now  & Get status/length */
		(db->rx_status)(&RxStatus, &RxLen);

		DM9000_DBG("rx status: 0x%04x rx len: %d\n", RxStatus, RxLen);

		/* Move data from DM9000 */
		/* Read received packet from RX SRAM */
		(db->inblk)(rdptr, RxLen);

		if ((RxStatus & 0xbf00) || (RxLen < 0x40)
			|| (RxLen > DM9000_PKT_MAX)) {
			if (RxStatus & 0x100) {
				printf("rx fifo error\n");
			}
			if (RxStatus & 0x200) {
				printf("rx crc error\n");
			}
			if (RxStatus & 0x8000) {
				printf("rx length error\n");
			}
			if (RxLen > DM9000_PKT_MAX) {
				printf("rx length too big\n");
				dm9000_reset();
			}
		} else {
			DM9000_DMP_PACKET(__func__ , rdptr, RxLen);

			DM9000_DBG("passing packet to upper layer\n");
			NetReceive(NetRxPackets[0], RxLen);
		}
	}
	return 0;
}
```

我们接着看下，接收完成呢？又该去干什么了？别忘了，接收只是其中一步，我们在 `bootcmd` 中一共有三部分，传送内核到内存 20000000 处，传送设备树到 21000000 处，使用 bootm 启动。现在就当已经接收完成，接着看看后面做了什么。清除接收函数，如果接收的数据大小大于 0，设置 `filesize` 为接收的数字大小，`fileaddr` 为接收的地址。然后跳转到 done 处，done 处又是清除 udp、icmp 句柄，然后函数就退出了。

```c
// net/net.c
int NetLoop(enum proto_t protocol)
{
			...
			net_cleanup_loop();
			if (NetBootFileXferSize > 0) {
				printf("Bytes transferred = %ld (%lx hex)\n",
					NetBootFileXferSize,
					NetBootFileXferSize);
				setenv_hex("filesize", NetBootFileXferSize);
				setenv_hex("fileaddr", load_addr);
			}
			...
			goto done;
done:
#ifdef CONFIG_USB_KEYBOARD
	net_busy_flag = 0;
#endif
#ifdef CONFIG_CMD_TFTPPUT
	/* Clear out the handlers */
	net_set_udp_handler(NULL);
	net_set_icmp_handler(NULL);
#endif
	return ret;
}
```

返回到 `netboot_common` 中去继续执行，设置参数，刷新缓存，查看是否要自动开始，也就是环境变量中的 `autostart` 为 `yes` 时，然后跳转到 `do_bootm` 中，此处就不再分析了。

下面接着分析 do_bootm。

###### do_bootm

首先了解一下 uImage 和 zImage 的区别。

> 编译 kernel 之后，会生成 Image 或者压缩过的 zImage。但是这两种镜像的格式并没有办法提供给 uboot 的足够的信息来进行 load、jump 或者验证操作等等。因此，uboot 提供了 mkimage 工具，来将 kernel 制作为 uboot 可以识别的格式，将生成的文件称之为 uImage。
> uboot 支持两种类型的 uImage，如下
>
> - Legacy-uImage
>   在 kernel 镜像的基础上，加上 64Byte 的信息提供给 uboot 使用。
>
> - FIT-uImage
>   以类似 FDT 的方式，将 kernel、fdt、ramdisk 等等镜像打包到一个 image file 中，并且加上一些需要的信息（属性）。uboot 只要获得了这个 image file，就可以得到 kernel、fdt、ramdisk 等等镜像的具体信息和内容。
>
> Legacy-uImage 实现较为简单，并且长度较小。但是实际上使用较为麻烦，需要在启动 kernel 的命令中额外添加 fdt、ramdisk 的加载信息。
> 而 FIT-uImage 实现较为复杂，但是使用起来较为简单，兼容性较好,（可以兼容多种配置）。但是需要的额外信息也较长。
>
> uImage 相较于 zImage 其在头部添加了 64bytes `image_header` 用来表示 Legacy-uImage 的头部
>
> ```c
> typedef struct image_header {
>     __be32      ih_magic;   /* Image Header Magic Number    */   // 幻数头，用来校验是否是一个Legacy-uImage
>     __be32      ih_hcrc;    /* Image Header CRC Checksum    */ // 头部的CRC校验值
>     __be32      ih_time;    /* Image Creation Timestamp */ // 镜像创建的时间戳
>     __be32      ih_size;    /* Image Data Size      */ // 镜像数据长度
>     __be32      ih_load;    /* Data  Load  Address      */ // 加载地址
>     __be32      ih_ep;      /* Entry Point Address      */ // 入口地址
>     __be32      ih_dcrc;    /* Image Data CRC Checksum  */ // 镜像的CRC校验
>     uint8_t     ih_os;      /* Operating System     */ // 操作系统类型
>     uint8_t     ih_arch;    /* CPU architecture     */ // 体系
>     uint8_t     ih_type;    /* Image Type           */ // 镜像类型
>     uint8_t     ih_comp;    /* Compression Type     */ // 压缩类型
>     uint8_t     ih_name[IH_NMLEN];  /* Image Name       */ // 镜像名
> } image_header_t;
> #define IH_NMLEN        32  /* Image Name Length        */
> ```
>
> 通过比较编译出的 uImage 和 zImage，其大小符合上述我们说的，并且可以通过这 64 字节得到
>
> - ih_magic 0x27051956 幻术头
>
> - ih_size 003fe868 4188264 zImage 大小
>
> - ih_load 0x20008000 数据加载地址
>
> - ih_ep 0x20008000 入口地址
>
> - etc
>
>   ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204241433964.png)

bootm 格式如下

`bootm Legacy-uImage加载地址 ramdisk加载地址 dtb加载地址`

```c
// 假设Legacy-uImage的加载地址是0x20008000，ramdisk的加载地址是0x21000000，fdt的加载地址是0x22000000

// 1.只加载kernel的情况下
bootm 0x20008000

// 2. 加载kernel和ramdisk
bootm 0x20008000 0x21000000

// 3.加载kernel和fdt
bootm 0x20008000 - 0x22000000

// 4.加载kernel、ramdisk、fdt
bootm 0x20008000 0x21000000 0x22000000
```

看一下 do_bootm 的代码

```c
int do_bootm(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	debug("argc:%d\r\n",argc);
	int idx=0;
	for(;idx<argc;idx++){
		printf("argv[%d]:%s\r\n",idx,argv[idx]);
	}
	//  在这里，bootm的第一个参数会被去掉。
        //  也就是当"bootm 0x20000000 - 0x21000000 "时，
        //  argc = 3，argv[0] = "0x20000000"，argv[1]= "-"，argv[2]="0x21000000"
        //  当"bootm 0x20000000"时，argc=1，argv[0]="0x20000000"
	argc--; argv++;
	if (argc > 0) {
		char *endp;
		simple_strtoul(argv[0], &endp, 16);
		if ((*endp != 0) && (*endp != ':') && (*endp != '#')) // 判断是否有子命令，这里我们不管。
			return do_bootm_subcommand(cmdtp, flag, argc, argv);
	}

	return do_bootm_states(cmdtp, flag, argc, argv, BOOTM_STATE_START |
		BOOTM_STATE_FINDOS | BOOTM_STATE_FINDOTHER |
		BOOTM_STATE_LOADOS |
		BOOTM_STATE_OS_PREP | BOOTM_STATE_OS_FAKE_GO |
		BOOTM_STATE_OS_GO, &images, 1);
}
```

通过日志可得，其传入的参数也就是我们在 bootcmd 中配置的 bootm 及其参数，也就是加载在 0x20000000 处的 kernel 和 0x21000000 处的设备树。

```shell
[File:common/cmd_bootm.c, Line:797, Function:do_bootm] argc:4
argv[0]:bootm
argv[1]:20000000
argv[2]:-
argv[3]:21000000
```

最后，跳转到 `do_bootm_states` 中，此时标志有

- BOOTM_STATE_START
- BOOTM_STATE_FINDOS
- BOOTM_STATE_FINDOTHER
- BOOTM_STATE_LOADOS
- BOOTM_STATE_OS_PREP
- BOOTM_STATE_OS_FAKE_GO
- BOOTM_STATE_OS_GO

还有一个全局参数 `images` 的地址，而 `images` 的格式是 `struct bootm_headers`

```c
do_bootm
    |
    --> do_bootm_states
```

`struct bootm_headers`

```c
typedef struct bootm_headers {
	/*
	 * Legacy os image header, if it is a multi component image
	 * then boot_get_ramdisk() and get_fdt() will attempt to get
	 * data from second and third component accordingly.
	 */
	image_header_t	*legacy_hdr_os;		/* image header pointer */ //Legacy-uImage的镜像头
	image_header_t	legacy_hdr_os_copy;	/* header copy */		//Legacy-uImage的镜像头备份
	ulong		legacy_hdr_valid;		// Legacy-uImage的镜像头是否存在的标记

#if defined(CONFIG_FIT)
	const char	*fit_uname_cfg;	/* configuration node unit name */     // 配置节点名

	void		*fit_hdr_os;	/* os FIT image header */		// FIT-uImage中kernel镜像头
	const char	*fit_uname_os;	/* os subimage node unit name */   // FIT-uImage中kernel的节点名
	int		fit_noffset_os;	/* os subimage node offset */	             //  FIT-uImage中kernel的节点偏移

	void		*fit_hdr_rd;	/* init ramdisk FIT image header */ // FIT-uImage中ramdisk的镜像头
	const char	*fit_uname_rd;	/* init ramdisk subimage node unit name */ // FIT-uImage中ramdisk的节点名
	int		fit_noffset_rd;	/* init ramdisk subimage node offset */		       // FIT-uImage中ramdisk的节点偏移

	void		*fit_hdr_fdt;	/* FDT blob FIT image header */		      // FIT-uImage中FDT的镜像头
	const char	*fit_uname_fdt;	/* FDT blob subimage node unit name */	// FIT-uImage中FDT的节点名
	int		fit_noffset_fdt;/* FDT blob subimage node offset */			  // FIT-uImage中ramdisk的节点偏移
#endif

#ifndef USE_HOSTCC
	image_info_t	os;		/* os image info */				// 操作系统信息的结构体
	ulong		ep;		/* entry point of OS */				// 操作系统的入口地址

	ulong		rd_start, rd_end;/* ramdisk start/end */		// ramdisk在内存上的起始地址和结束地址

	char		*ft_addr;	/* flat dev tree address */			// fdt在内存上的地址
	ulong		ft_len;		/* length of flat device tree */	     // fdt在内存上的长度

	ulong		initrd_start;
	ulong		initrd_end;
	ulong		cmdline_start;
	ulong		cmdline_end;
	bd_t		*kbd;
#endif

	int		verify;		/* getenv("verify")[0] != 'n' */ 	  // 是否需要验证
	int		state;								// 状态标识，用于标识对应的bootm需要做什么操作，具体看下面宏

#ifdef CONFIG_LMB
	struct lmb	lmb;		/* for memory mgmt */	   // 内存管理
#endif
} bootm_headers_t;

#define	BOOTM_STATE_START	(0x00000001) 		//开始执行bootm的准备动作
#define	BOOTM_STATE_FINDOS	(0x00000002) 		// 查找操作系统镜像
#define	BOOTM_STATE_FINDOTHER	(0x00000004)       // 查找操作系统镜像外的其他镜像，比如FDT/ramdisk等

#define	BOOTM_STATE_LOADOS	(0x00000008) 		// 加载操作系统
#define	BOOTM_STATE_RAMDISK	(0x00000010)		// 操作ramdisk
#define	BOOTM_STATE_FDT		(0x00000020) 		  // 操作FDT
#define	BOOTM_STATE_OS_CMDLINE	(0x00000040)	// 操作commandline
#define	BOOTM_STATE_OS_BD_T	(0x00000080)
#define	BOOTM_STATE_OS_PREP	(0x00000100)		// 跳转到操作系统前的准备动作
#define	BOOTM_STATE_OS_FAKE_GO	(0x00000200)	// 伪跳转，一般都能直接跳转到kernel中去
#define	BOOTM_STATE_OS_GO	(0x00000400) 		 // 跳转到kernel中去

extern bootm_headers_t images;
```

`do_bootm_states`

```c
static int do_bootm_states(cmd_tbl_t *cmdtp, int flag, int argc,
		char * const argv[], int states, bootm_headers_t *images,
		int boot_progress)
{
	boot_os_fn *boot_fn;
	ulong iflag = 0;
	int ret = 0, need_boot_fn;
	// 更新images->state中的状态标志
	images->state |= states;

	/*
	 * Work through the states and see how far we get. We stop on
	 * any error.
	 */
         // 判断当前状态是否有BOOTM_STATE_START，也就是是否需要执行前的准备动作，如果需要则调用bootm_start
	if (states & BOOTM_STATE_START)
		ret = bootm_start(cmdtp, flag, argc, argv);
	// 判断当前状态是否有BOOTM_STATE_FINDOS，也就是是否需要执行查找OS，如果需要则调用 bootm_find_os，注意此步是紧接着上一步的
	if (!ret && (states & BOOTM_STATE_FINDOS))
		ret = bootm_find_os(cmdtp, flag, argc, argv);
	// 判断当前状态是否有BOOTM_STATE_FINDOTHER，也就是是否需要执行查找其它FDT/ramdisk等，如果需要则调用bootm_find_other，注意此步是紧接着上一步的
	if (!ret && (states & BOOTM_STATE_FINDOTHER)) {
		ret = bootm_find_other(cmdtp, flag, argc, argv);
		argc = 0;	/* consume the args */
	}

        // 注意：之前都是查找os，查找fdt等填充images中的结构，下面则是利用images中的结构数据。
	/* Load the OS */
         // 判断当前状态是否有BOOTM_STATE_LOADOS，也就是是否需要执行加载OS，如果需要则调用bootm_load_os，注意此步是紧接着上一步的
	if (!ret && (states & BOOTM_STATE_LOADOS)) {
		ulong load_end;

		iflag = bootm_disable_interrupts();
		ret = bootm_load_os(images, &load_end, 0);
		if (ret == 0)
			lmb_reserve(&images->lmb, images->os.load,
				    (load_end - images->os.load));
		else if (ret && ret != BOOTM_ERR_OVERLAP)
			goto err;
		else if (ret == BOOTM_ERR_OVERLAP)
			ret = 0;
#if defined(CONFIG_SILENT_CONSOLE) && !defined(CONFIG_SILENT_U_BOOT_ONLY)
		if (images->os.os == IH_OS_LINUX)
			fixup_silent_linux();
#endif
	}

	/* Relocate the ramdisk */ // 是否需要重定向ramdinsk，do_bootm流程的话是不需要的
#ifdef CONFIG_SYS_BOOT_RAMDISK_HIGH
	if (!ret && (states & BOOTM_STATE_RAMDISK)) {
		ulong rd_len = images->rd_end - images->rd_start;

		ret = boot_ramdisk_high(&images->lmb, images->rd_start,
			rd_len, &images->initrd_start, &images->initrd_end);
		if (!ret) {
			setenv_hex("initrd_start", images->initrd_start);
			setenv_hex("initrd_end", images->initrd_end);
		}
	}
#endif
#if defined(CONFIG_OF_LIBFDT) && defined(CONFIG_LMB)
         // 是否需要重定向fdt，do_bootm流程的话是不需要的
	if (!ret && (states & BOOTM_STATE_FDT)) {
		boot_fdt_add_mem_rsv_regions(&images->lmb, images->ft_addr);
		ret = boot_relocate_fdt(&images->lmb, &images->ft_addr,
					&images->ft_len);
	}
#endif

	/* From now on, we need the OS boot function */
	if (ret)
		return ret;
        // 获取对应操作系统的启动函数，存放到boot_fn中
	boot_fn = boot_os[images->os.os];
	need_boot_fn = states & (BOOTM_STATE_OS_CMDLINE |
			BOOTM_STATE_OS_BD_T | BOOTM_STATE_OS_PREP |
			BOOTM_STATE_OS_FAKE_GO | BOOTM_STATE_OS_GO);
	if (boot_fn == NULL && need_boot_fn) {
		if (iflag)
			enable_interrupts();
		printf("ERROR: booting os '%s' (%d) is not supported\n",
		       genimg_get_os_name(images->os.os), images->os.os);
		bootstage_error(BOOTSTAGE_ID_CHECK_BOOT_OS);
		return 1;
	}

	/* Call various other states that are not generally used */
	if (!ret && (states & BOOTM_STATE_OS_CMDLINE))
		ret = boot_fn(BOOTM_STATE_OS_CMDLINE, argc, argv, images);
	if (!ret && (states & BOOTM_STATE_OS_BD_T))
		ret = boot_fn(BOOTM_STATE_OS_BD_T, argc, argv, images);
	if (!ret && (states & BOOTM_STATE_OS_PREP))// 判断当前状态是否有BOOTM_STATE_OS_PREP，也就是跳转前的最后准备动作，如果需要则调用boot_fn
		ret = boot_fn(BOOTM_STATE_OS_PREP, argc, argv, images);

#ifdef CONFIG_TRACE
	 // 判断当前状态是否有BOOTM_STATE_OS_FAKE_GO，伪跳转到操作系统。
	if (!ret && (states & BOOTM_STATE_OS_FAKE_GO)) {
		char *cmd_list = getenv("fakegocmd");

		ret = boot_selected_os(argc, argv, BOOTM_STATE_OS_FAKE_GO,
				images, boot_fn);
		if (!ret && cmd_list)
			ret = run_command_list(cmd_list, -1, flag);
	}
#endif

	/* Check for unsupported subcommand. */
	if (ret) {
		puts("subcommand not supported\n");
		return ret;
	}

	/* Now run the OS! We hope this doesn't return */ // 判断当前状态是否有BOOTM_STATE_OS_GO，也就是现在直接跳转到SO，并且不需要返回
	if (!ret && (states & BOOTM_STATE_OS_GO))
		ret = boot_selected_os(argc, argv, BOOTM_STATE_OS_GO,
				images, boot_fn);

	/* Deal with any fallout */
err:
	if (iflag)
		enable_interrupts();

	if (ret == BOOTM_ERR_UNIMPLEMENTED)
		bootstage_error(BOOTSTAGE_ID_DECOMP_UNIMPL);
	else if (ret == BOOTM_ERR_RESET)
		do_reset(cmdtp, flag, argc, argv);

	return ret;
}
```

**BOOTM_STATE_START**

bootm_start，简单看一下，其实也就是初始化 `bootm_headers_t images`，也就是从环境变量中获取 `verify` 对应的 value，然后返回 0 或 1，赋值给 `images.verify`。配置 images 中的内存保留区域，标记 images 中的当前状态。

**实现 verify 和 lmb**

> LMB 是指 logical memory blocks，主要是用于表示内存的保留区域，主要有 fdt 的区域，ramdisk 的区域等等。

```c
// common/cmd_bootm.c
static int bootm_start(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
	memset((void *)&images, 0, sizeof(images));
	images.verify = getenv_yesno("verify");

	boot_start_lmb(&images);

	bootstage_mark_name(BOOTSTAGE_ID_BOOTM_START, "bootm_start");
	images.state = BOOTM_STATE_START;

	return 0;
}
// ----------------------------------------------------------------------------------------------------------
// common/cmd_bootm.c
static void boot_start_lmb(bootm_headers_t *images)
{
	ulong		mem_start;
	phys_size_t	mem_size;
	// 先初始化images的lmb内存管理，注意cnt为1
	lmb_init(&images->lmb);
	// 然后从环境变量中获取bootm_low，我们的板子上没有但是定义了CONFIG_SYS_SDRAM_BASE，所以此处返回 CONFIG_SYS_SDRAM_BASE
	mem_start = getenv_bootm_low();
	mem_size = getenv_bootm_size(); // 从环境变量中获取 bootm_size，但是我们的板子上又没有，所以此函数中tmp为0，然后CONFIG_ARM有定义，所以此处的值为 gd->bd->bi_dram[0].size - tmp，也就是 gd->bd->bi_dram[0].size，也就是 PHYS_SDRAM_1_SIZE
	// 此处也就是将 mem_start, mem_size，配置到images->lmb中，在此处会用到cnt=1
	lmb_add(&images->lmb, (phys_addr_t)mem_start, mem_size);
	// 这两个函数的作用暂时不理解
	arch_lmb_reserve(&images->lmb);
	board_lmb_reserve(&images->lmb);
}
```

**BOOTM_STATE_FINDOS**

bootm_find_os，主要是验证内核，从 `bootm` 命令及 `image_header` 中获取内核信息并更新到 `images`。

实现 os 和 ep。

```c
static int bootm_find_os(cmd_tbl_t *cmdtp, int flag, int argc,
			 char * const argv[])
{
	const void *os_hdr;

	/* get kernel image header, start address and length */
	// 验证内核，获取kernel起始地址和长度
	os_hdr = boot_get_kernel(cmdtp, flag, argc, argv,
			&images, &images.os.image_start, &images.os.image_len);
	if (images.os.image_len == 0) {
		puts("ERROR: can't get kernel image!\n");
		return 1;
	}

	/* get image parameters */
	// 更新images中的参数
	switch (genimg_get_format(os_hdr)) {
	case IMAGE_FORMAT_LEGACY:
		images.os.type = image_get_type(os_hdr);
		images.os.comp = image_get_comp(os_hdr);
		images.os.os = image_get_os(os_hdr);

		images.os.end = image_get_image_end(os_hdr);
		images.os.load = image_get_load(os_hdr);
		break;
#if defined(CONFIG_FIT)
	case IMAGE_FORMAT_FIT:
		if (fit_image_get_type(images.fit_hdr_os,
					images.fit_noffset_os, &images.os.type)) {
			puts("Can't get image type!\n");
			bootstage_error(BOOTSTAGE_ID_FIT_TYPE);
			return 1;
		}

		if (fit_image_get_comp(images.fit_hdr_os,
					images.fit_noffset_os, &images.os.comp)) {
			puts("Can't get image compression!\n");
			bootstage_error(BOOTSTAGE_ID_FIT_COMPRESSION);
			return 1;
		}

		if (fit_image_get_os(images.fit_hdr_os,
					images.fit_noffset_os, &images.os.os)) {
			puts("Can't get image OS!\n");
			bootstage_error(BOOTSTAGE_ID_FIT_OS);
			return 1;
		}

		images.os.end = fit_get_end(images.fit_hdr_os);

		if (fit_image_get_load(images.fit_hdr_os, images.fit_noffset_os,
					&images.os.load)) {
			puts("Can't get image load address!\n");
			bootstage_error(BOOTSTAGE_ID_FIT_LOADADDR);
			return 1;
		}
		break;
#endif
	default:
		puts("ERROR: unknown image format type!\n");
		return 1;
	}

	/* find kernel entry point */
	// 获取内核的entry point
	if (images.legacy_hdr_valid) {
		images.ep = image_get_ep(&images.legacy_hdr_os_copy);
#if defined(CONFIG_FIT)
	} else if (images.fit_uname_os) {
		int ret;

		ret = fit_image_get_entry(images.fit_hdr_os,
					  images.fit_noffset_os, &images.ep);
		if (ret) {
			puts("Can't get entry point property!\n");
			return 1;
		}
#endif
	} else {
		puts("Could not find kernel entry point!\n");
		return 1;
	}

	if (images.os.type == IH_TYPE_KERNEL_NOLOAD) {
		images.os.load = images.os.image_start;
		images.ep += images.os.load;
	}

	images.os.start = (ulong)os_hdr;

	return 0;
}
```

**BOOTM_STATE_FINDOTHER**

bootm_find_other

实现 rd_start，rd_end，ft_addr 和 initrd_end。

```c
static int bootm_find_other(cmd_tbl_t *cmdtp, int flag, int argc,
			    char * const argv[])
{
	if (((images.os.type == IH_TYPE_KERNEL) ||
	     (images.os.type == IH_TYPE_KERNEL_NOLOAD) ||
	     (images.os.type == IH_TYPE_MULTI)) &&
	    (images.os.os == IH_OS_LINUX ||
		 images.os.os == IH_OS_VXWORKS)) {
		if (bootm_find_ramdisk(flag, argc, argv))
			return 1;

#if defined(CONFIG_OF_LIBFDT)
		if (bootm_find_fdt(flag, argc, argv))
			return 1;
#endif
	}

	return 0;
}
```

先看 `bootm_find_ramdisk`，由于我们在 bootm 中传递的 ramfs 值为 `-`，也就是忽略 ramfs 的意思，所以会直接跳过，详见下面的日志。

```shell
[File:common/image.c, Line:814, Function:boot_get_ramdisk] ## Skipping init Ramdisk
[File:common/image.c, Line:944, Function:boot_get_ramdisk] ## No init Ramdisk
[File:common/image.c, Line:950, Function:boot_get_ramdisk]    ramdisk start = 0x00000000, ramdisk end = 0x00000000
```

接着看一下 `bootm_find_fdt`，无非就是验证 bootm 中的第三个参数以及是否可用，并确认设备树的类型，根据日志对应着代码看就可，此处不在展开说。

另外，此处会把

```c
[File:common/image-fdt.c, Line:271, Function:boot_get_fdt] *  fdt: cmdline image address = 0x21000000
[File:common/image-fdt.c, Line:289, Function:boot_get_fdt] ## Checking for 'FDT'/'FDT Image' at 21000000
[File:common/image-fdt.c, Line:370, Function:boot_get_fdt] *  fdt: raw FDT blob
## Flattened Device Tree blob at 21000000
   Booting using the fdt blob at 0x21000000
```

**BOOTM_STATE_LOADOS**

bootm_load_os

在 [do_bootm](#do_bootm) 中可知 uImage header 的定义，所以我们可以得出镜像压缩类型值为 0，此处也即走 `IH_COMP_NONE` 分支，所以会有如下打印

```shell
   Loading Kernel Image ... OK
[File:common/cmd_bootm.c, Line:481, Function:bootm_load_os]    kernel loaded at 0x20008000, end = 0x20406868
[File:common/cmd_bootm.c, Line:486, Function:bootm_load_os] images.os.start = 0x20000000, images.os.end = 0x203fe8a8
[File:common/cmd_bootm.c, Line:488, Function:bootm_load_os] images.os.load = 0x20008000, load_end = 0x20406868
```

精简一下代码，去掉不走的分支，如下，此处功能就是通过 `images` 获得镜像信息，看是否需要解压镜像，验证类型等。

```c
// common/cmd_bootm.c
static int bootm_load_os(bootm_headers_t *images, unsigned long *load_end,
		int boot_progress)
{
	image_info_t os = images->os;
	uint8_t comp = os.comp;
	ulong load = os.load;
	ulong blob_start = os.start;
	ulong blob_end = os.end;
	ulong image_start = os.image_start;
	ulong image_len = os.image_len;
	__maybe_unused uint unc_len = CONFIG_SYS_BOOTM_LEN;
	int no_overlap = 0;
	void *load_buf, *image_buf;
#if defined(CONFIG_LZMA) || defined(CONFIG_LZO)
	int ret;
#endif /* defined(CONFIG_LZMA) || defined(CONFIG_LZO) */

	const char *type_name = genimg_get_type_name(os.type);

	load_buf = map_sysmem(load, unc_len);
	image_buf = map_sysmem(image_start, image_len);
	switch (comp) {
	case IH_COMP_NONE:
		if (load == blob_start || load == image_start) {
			printf("   XIP %s ... ", type_name);
			no_overlap = 1;
		} else {
			printf("   Loading %s ... ", type_name);
			memmove_wd(load_buf, image_buf, image_len, CHUNKSZ);
		}
		*load_end = load + image_len;
		break;
#ifdef CONFIG_GZIP
	case IH_COMP_GZIP:
		...
#endif /* CONFIG_GZIP */
#ifdef CONFIG_BZIP2
	case IH_COMP_BZIP2:
		...
#endif /* CONFIG_BZIP2 */
#ifdef CONFIG_LZMA
	case IH_COMP_LZMA:
		...
#endif /* CONFIG_LZMA */
#ifdef CONFIG_LZO
	case IH_COMP_LZO:
		...
#endif /* CONFIG_LZO */
	default:
		printf("Unimplemented compression type %d\n", comp);
		return BOOTM_ERR_UNIMPLEMENTED;
	}

	flush_cache(load, (*load_end - load) * sizeof(ulong));

	puts("OK\n");
	debug("   kernel loaded at 0x%08lx, end = 0x%08lx\n", load, *load_end);
	bootstage_mark(BOOTSTAGE_ID_KERNEL_LOADED);

	if (!no_overlap && (load < blob_end) && (*load_end > blob_start)) {
		debug("images.os.start = 0x%lX, images.os.end = 0x%lx\n",blob_start, blob_end);
		debug("images.os.load = 0x%lx, load_end = 0x%lx\n", load,*load_end);

		/* Check what type of image this is. */
		if (images->legacy_hdr_valid) {
			if (image_get_type(&images->legacy_hdr_os_copy)== IH_TYPE_MULTI)
				puts("WARNING: legacy format multi component image overwritten\n");
			return BOOTM_ERR_OVERLAP;
		} else {
			puts("ERROR: new format image overwritten - must RESET the board to recover\n");
			bootstage_error(BOOTSTAGE_ID_OVERWRITTEN);
			return BOOTM_ERR_RESET;
		}
	}

	return 0;
}
```

回到 do_bootm_states 函数，调用 `lmb_reserve` 对剩余的内存进行管理。

`BOOTM_STATE_RAMDISK`

接着调用 `boot_ramdisk_high`，然而其中的 rd_data 为 0，所以整个函数退出。

```shell
[File:common/image.c, Line:1003, Function:boot_ramdisk_high] ## initrd_high = 0xffffffff, copy_to_ram = 1
[File:common/image.c, Line:1047, Function:boot_ramdisk_high]    ramdisk load start = 0x00000000, ramdisk load end = 0x00000000
```

`BOOTM_STATE_FDT`

验证 FDT，将代码从 `21000000` 重定位到 `3fe30000` 处。

```shell
[File:common/image-fdt.c, Line:175, Function:boot_relocate_fdt] ## device tree at 21000000 ... 210063a5 (len=37798 [0x93A6])
   Loading Device Tree to 3fe30000, end 3fe393a5 ... OK
```

接着配置 `boot_fn`，由于是 `images->os.os` 为 1，也就是 linux，所以 `boot_fn=do_bootm_linux`

由于上面 states 中含有这几个标志，

- BOOTM_STATE_START
- BOOTM_STATE_FINDOS
- BOOTM_STATE_FINDOTHER
- BOOTM_STATE_LOADOS
- BOOTM_STATE_OS_PREP
- BOOTM_STATE_OS_FAKE_GO
- BOOTM_STATE_OS_GO

当前要配置的状态有

- BOOTM_STATE_OS_CMDLINE
- BOOTM_STATE_OS_BD_T
- BOOTM_STATE_OS_PREP
- BOOTM_STATE_OS_FAKE_GO
- BOOTM_STATE_OS_GO

`need_boot_fn` 状态为二者共有的，也即 BOOTM_STATE_OS_PREP、BOOTM_STATE_OS_FAKE_GO、BOOTM_STATE_OS_GO

所以，接着会调用 `do_bootm_linux(BOOTM_STATE_OS_PREP, argc, argv, images);`

也就是会调用 `boot_jump_linux`

看下 `boot_jump_linux`，我们是 32 位的 ARM，所以删掉 64 位的代码。直接看剩下的，获取设备 id，并从环境变量中获取 `machid` 对应的 value，打印。

然后定义一个函数指针，有三个参数，其类型分别为 `int, int, uint`，

> 内核启动之前要求 r0 为 0，r1 为 machid，r2 为 atags 或设备树地址。

所以对应的为 r0，r1，r2。

然后 `kernel_entry(0, machid, r2)` 也就是直接执行内核 entry point 处的代码，换句话说也就是跳转到了 kernel 中。

```c
static void boot_jump_linux(bootm_headers_t *images, int flag)
{
#ifdef CONFIG_ARM64
	...
#else
	unsigned long machid = gd->bd->bi_arch_number;
	char *s;
	void (*kernel_entry)(int zero, int arch, uint params);
	unsigned long r2;
	int fake = (flag & BOOTM_STATE_OS_FAKE_GO);

	kernel_entry = (void (*)(int, int, uint))images->ep;

	s = getenv("machid");
	if (s) {
		strict_strtoul(s, 16, &machid);
		printf("Using machid 0x%lx from environment\n", machid);
	}

	debug("## Transferring control to Linux (at address %08lx)" "...\n", (ulong) kernel_entry);
	bootstage_mark(BOOTSTAGE_ID_RUN_OS);
	announce_and_cleanup(fake);

	if (IMAGE_ENABLE_OF_LIBFDT && images->ft_len)
		r2 = (unsigned long)images->ft_addr;
	else
		r2 = gd->bd->bi_boot_params;

	if (!fake)
		kernel_entry(0, machid, r2);
#endif
}
```

## 参考

1. [wowo 的 u-boot 启动流程分析（1）平台相关部分\_](http://www.wowotech.net/u-boot/boot_flow_1.html)
2. [ooonebook 大佬的 project-X 系列](https://blog.csdn.net/ooonebook/category_6471981.html)
3. [ooonebook 大佬的 u-boot 系列](https://blog.csdn.net/ooonebook/category_6484145.html)
4. [S5PV210_iROM_ApplicationNote_Preliminary_20091126.pdf](https://github.com/limingth/ARM-Resources/blob/master/tiny210/Datasheet/S5PV210_iROM_ApplicationNote_Preliminary_20091126.pdf)
5. [S5PV210_UM_REV1.1.pdf](https://github.com/limingth/ARM-Resources/blob/master/tiny210/Datasheet/S5PV210_UM_REV1.1.pdf)
6. https://re-eject.gbadev.org/files/GasARMRef.pdf
7. [第三十二章 U-Boot 启动流程详解 -摘自【正点原子】I.MX6U 嵌入式 Linux 驱动开发指南 V1.0](https://blog.csdn.net/weixin_55796564/article/details/119949722)
8. [嵌入式 Linux 学习：u-boot 源码分析（1）--AM335X 系列的 2014.10 版](https://blog.csdn.net/u012176730/article/details/54670569)
9. [ARM 汇编指令集汇总](https://blog.csdn.net/qq_40531974/article/details/83897559)
10. [ARM 指令 CMP 详解](https://blog.csdn.net/zhouqt/article/details/78172332)
11. [[ARM 汇编指令 ADR 与 LDR 使用 ](https://www.cnblogs.com/zackary/p/9343253.html)](https://www.cnblogs.com/zackary/p/9343253.html)
12. [谈谈 U-boot 启动流程](https://0uyangsheng.github.io/2018/04/25/Deep-into-Rk3399-uboot/)
13. [u-boot-2019.10 源码分析——init_sequence_f 中的函数](https://blog.csdn.net/gzxb1995/article/details/103458589)
14. [uboot 环境变量实现分析](https://blog.csdn.net/skyflying2012/article/details/39005705)
15. [LD 脚本](https://sourceware.org/binutils/docs/ld/Scripts.html)
16. [u-boot.lds 链接文件详解](https://www.jianshu.com/p/ec39403db315)
17. [Arm 汇编指令](http://blog.chinaunix.net/uid-23193900-id-3251565.html)
18. [ARM 汇编之解惑条件标志，条件码，条件执行](https://www.jianshu.com/p/0c6192da2fd0)
19. [从零开始之 uboot、移植 uboot2017.01 @奔跑的小刺猬](https://blog.csdn.net/qq_16777851/article/details/81749077)
20. [协处理器 CP15 介绍—MCR/MRC 指令](https://www.cnblogs.com/lifexy/p/7203786.html)
21. [网卡 DM9000 裸机驱动详解](https://zhuanlan.zhihu.com/p/374941834)
22. [【详解】如何编写 Linux 下 Nand Flash 驱动](https://www.crifan.com/files/doc/docbook/linux_nand_driver/release/html/linux_nand_driver.html)
23. [u-boot 中的 shell](https://wushifu-notes.readthedocs.io/zh/latest/u-boot%20%E4%B8%AD%E7%9A%84%20shell/)
24. [Uboot 命令 U_BOOT_CMD 分析](https://blog.csdn.net/qq_33894122/article/details/86129765)
25. [u-boot 的网络](https://oska874.github.io/%E6%BA%90%E7%A0%81/uboot%E7%9A%84%E7%BD%91%E7%BB%9C.html)
26. [busybox 中 hush_doc 的说明](https://git.busybox.net/busybox/tree/shell/hush_doc.txt)
27. [ARM64 汇编——寄存器和指令](https://www.jianshu.com/p/2f4a5f74ac7a)
