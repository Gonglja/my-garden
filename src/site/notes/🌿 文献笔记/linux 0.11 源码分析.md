---
{"dg-publish":true,"permalink":"/🌿 文献笔记/linux 0.11 源码分析/","tags":["Linux","Linux/kernel","源码分析"]}
---


# Linux 0.11 环境搭建与阅读

## 背景

更好的学习 linux 内核思想，做一个有想法的人。

## 名词约定

- CPL：Current privilege level，当前权限级别
- DPL：Descriptor privilege level，
- RPL：Requested privilege level，请求权限级别

## 环境搭建

在 windows 下使用虚拟机 vmware，虚拟机使用 ubuntu 22.04 LTS，模拟器使用 qemu。

搭建虚拟机 vmware 过程省略，具体的可查看 [archlinux 的搭建](https://gonglja.github.io/posts/d78cdbc6/)

### 搭建 ubuntu 中 qemu

> 由于使用命令 sudo apt install qemu 安装版本过旧并可能存在问题，所以本文使用编译方式安装。

ubuntu 宿主机环境如下：

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202205110934467.png)

### 安装环境必备包

```shell
# 1. 安装环境必备包
sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev
sudo apt install ninja-build

# 2. 下载qemu源码
git clone https://mirrors.tuna.tsinghua.edu.cn/git/qemu.git

# 3. 配置qemu
cd qemu
./configure
# 如果配置没有报错，则可以直接编译并安装了。
# 大约 6376.73s 编译完成，可使用命令 time make -j$(nproc) 查看时间，以下为我本次编译所需时间
# make -j12  6376.73s user 729.92s system 1160% cpu 10:12.17 total
make -j$(nproc)

# 直接./configure 可能会报错，./configure --with-git-submodules=validate 重新配置后编译安装即可。
sudo make install -j$(nproc)

# 4.验证 通过命令可以看出 qemu 的版本号，代表安装完成。
u@u-virtual-machine /home/u/workspace/tools
⚡ qemu-system-x86_64 --version
QEMU emulator version 7.0.50
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

配置好后，打开发现只输出一句话，VNC Server runnning on 127.0.0.1:5900

这是因为没有支持 `SDL`(Simple DirectMedia Layer)

所以需要重新配置后在安装，

```shell
# 安装SDL支持依赖
sudo apt install libsdl1.2-dev libsdl2-dev

# 继续配置，一会搜索一下看下SDL是否为YES，为YES则已经开启了，剩下的直接编译安装就可。
./configure --with-git-submodules=validate

# 编译和安装
make -j$(nproc) && sudo make install -j$(nproc)

# 再次验证，输入下面命令后会直接弹出qemu虚拟机
qemu-system-x86_64
```

### 配置 gdb 环境

```shell
 sudo apt install gdb
```

习惯使用 pwndbg 了，添加此插件。

## 代码分析

首先下载源码，并编译，没有问题那么就开始分析源码了。

### 下载源码

地址为 [https://github.com/yuan-xy/Linux-0.11](https://github.com/yuan-xy/Linux-0.11)

不过也可以 fork 到自己的仓库中，方便后面更改后的上传。

```shell
git clone https://github.com/Gonglja/Linux-0.11.git && cd Linux-0.11
make clean -j$(nproc) && make -j$(nproc)
# 直接可以编过，不会缺少什么库。
```

代码汇编部分采用 AT&T 语法，需要熟悉 AT&T

### 源码分析

#### 寄存器

TODO

#### 进入内核前的苦力活

##### 系统上电后跳转到至 bios

当系统启动或者重置时，处理器在已知位置执行代码。在笔记本（PC）中，此位置位于 BIOS（Base input/output System，基本输入/输出系统），该系统存储在主板的 bios 芯片中。

当系统启动或重置时，处理器会默认跳转到 bios 中执行代码。为什么？

首先我们要了解 CPU 上电后运行在实模式下，在实模式下 CPU 的寻址地址为:$CS*4 + IP$，而上电后 $CS$ 默认值为 `0xffff`,$IP$ 默认值为 `0x0000`。所以上电后 CPU 会到 $CS*16+IP=0xffff << 4 + 0x0 = 0xffff0$ 处执行第一条指令。

但在 20 位地址下，最大访问空间也就是 `1Mb`（`0xfffff`），此处仅有 16 字节，空间太小以至于无法存储 bios 的代码，所以一般此处都为一个跳转指令。跳转到真正的 bios 处执行代码。

##### bios 完成后跳转 0x7c00 处

在 bios 中，首先完成 *POST*（Power On Self Test，上电自检），bios 对计算机各部件进行初始化，如果有错误则报警提示。下一步在外部存储设备中寻找操作系统，找到第一个可启动设备。将第一个可启动存储设备（第一个扇区最后两字节为 `0x55`、`0xaa`）的第一个扇区（首 512 字节）原封不动的复制到 `0x7c00`，然后跳转到 `0x7c00` 处，去执行相应的代码。

![Pasted image 20221130104420.png](/img/user/Resources/Images/Pasted%20image%2020221130104420.png)

###### 第一扇区

先看下最外层的 Makefile，关注几个点

```c
LDFLAGS += -Ttext 0 -e startup_32
```

当链接器链接的时候，其参数携带 `-Ttext 0`，指定代码段的运行地址从 `0` 开始

```c
all:	Image

Image: boot/bootsect boot/setup tools/system
	@cp -f tools/system system.tmp
	@$(STRIP) system.tmp
	@$(OBJCOPY) -O binary -R .note -R .comment system.tmp tools/kernel
	@tools/build.sh boot/bootsect boot/setup tools/kernel Image $(ROOT_DEV)
	@rm system.tmp
	@rm -f tools/kernel
	@sync

```

另一个就是 `all`，`all` 依赖 Image，`Image` 依赖 `boot/bootsect boot/setup tools/system` 等，编译结束后又经过 `tools/build.sh` 脚本构建镜像，通过脚本可知第一个 512 字节存的是 bootsect，后面接着 setup 和 system，具体可查看表格（根据 build.sh 脚本得出）

```c
# Write bootsect (512 bytes, one sector) to stdout
[ ! -f "$bootsect" ] && echo "there is no bootsect binary file there" && exit -1
dd if=$bootsect bs=512 count=1 of=$IMAGE 2>&1 >/dev/null

# Write setup(4 * 512bytes, four sectors) to stdout
[ ! -f "$setup" ] && echo "there is no setup binary file there" && exit -1
dd if=$setup seek=1 bs=512 count=4 of=$IMAGE 2>&1 >/dev/null

# Write system(< SYS_SIZE) to stdout
[ ! -f "$system" ] && echo "there is no system binary file there" && exit -1
system_size=`wc -c $system |cut -d" " -f1`
[ $system_size -gt $SYS_SIZE ] && echo "the system binary is too big" && exit -1
dd if=$system seek=5 bs=512 count=$((2888-1-4)) of=$IMAGE 2>&1 >/dev/null
```

| 模块                                  | 偏移量（段/Byte）| 大小（段/Byte）| 备注       |
| ------------------------------------- | ------------------- | ------------------------------------------------------- | ---------- |
| bootsect                              | 0/0                 | 1/512                                                   | 启动阶段   |
| setup                                 | 1/512               | 4/2048                                                  | 配置阶段   |
| system                                | 5/2560              | (2888-1-4)\*512=1476096，最大长度，实际为 system 的大小 | 内核等     |
| DEFAULT_MINOR_ROOT DEFAULT_MAJOR_ROOT | 0 段中的第 508 字节 | 2                                                       | 版本号信息 |

也由此可知，上电后从 bios 跳转过来后，直接执行的是 `bootsect` 中的内容。

###### 跳转到 0x7c00

```shell
➜  Linux-0.11 git:(master) tree boot
boot
├── bootsect.s
├── head.s
├── Makefile
└── setup.s

0 directories, 4 files
```

先看下 `Makefile`，通过编译脚本可得 链接器参数为 `-Ttext 0`，另外三个 `bootsect`、`setup`、`head` 模块也是分别编译。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204021631675.png)

**当 cpu 上电后，bios 会将第一个==可启动硬盘==的前 512 字节数据，拷贝到 0x7c00 处。**

> [!tips] 什么叫可启动设备
> 只要第一个扇区的 512 字节的最后两字节分别为 0x55、0xaa，那么其就是一个可启动设备。

前 512 字节是什么呢？`bootsect` 模块，换句话说，当 `cpu` 上电后，`bios` 将 `bootsect` 拷贝到 `0x7c00` 处，其大小为 `512` Byte。

之后就进入到 `0x7c00` 处，开始执行 `bootsect` 模块的代码。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204021638456.png)

##### 拷贝第一个 512 字节到 0x90000 处

`.global` 声明部分标号对全局可见，

- \_start 程序开始的地方
- begtext text 段开始的位置
- begdata data 段开始的位置
- begbss bss 段开始的位置
- endtext text 段结束的位置
- enddata data 段结束的位置
- endbss bss 段结束的位置

`.equ` 表达式赋值操作符

| 标号 | 值 | 含义 |
| -------- | -------------- | ----------------------- |
| SYSSIZE | 0x3000 (196kb) | system 大小 |
| SETUPLEN | 4 | setup 段中长度 |
| BOOTSEG | 0x07c0 | boot 段源地址 |
| INITSEG | 0x9000 | boot 段即将移动到的位置 |
| SETUPSEG | 0x9020 | setup 段开始的位置 |
| SYSSEG | 0x1000 | system 加载的位置 0x10000 |
| ENDSEG | SYSSEG+SYSSIZE | 停止加载的位置 |

`ljmp $BOOTSEG, $_start` 长跳转，跳转至 $0x7c00 = BOOTSEG<<16 + \_start$ 处

```c
 _start:
     mov $BOOTSEG, %ax    
     mov %ax, %ds      #将ds段寄存器设置为0x7C0
     mov $INITSEG, %ax
     mov %ax, %es      #将es段寄存器设置为0x9000
     mov $256, %cx     #设置移动计数值256字
     sub %si, %si      #源地址 ds:si = 0x07C0:0x0000
     sub %di, %di      #目标地址 es:si = 0x9000:0x0000
          rep          #重复执行并递减cx的值
     movsw             #从内存[si]处移动cx个字到[di]处
```

由于不能直接给 ds 赋值，借助 ax，将 ds 寄存器值设置为 `0x7C0`；同理，将 es 段寄存器设置为 `0x900`。

接着设置 cx 寄存器值为 `256`，将 si、di 寄存器清零。

> **movsw**：数据传送指令，从源地址向目的地址传送数据
> 在 16 位模式下，源地址 `DS:SI`，目的地址 `ES:DI`
> 在 32 位模式下，源地址 `DS:ESI`，目的地址 `ES:EDI`
> movsb、movsw、movsd 区别，b 字节、w 字、d 双字，也即传递一个字节、一个字、一个双字。

所以，这段代码的作用就是将 `ds:si`(`0x7c0:0x0` 即 `0x7c00`) 处开始，大小为 `256字`，即 `512字节` 的数据拷贝到 `es:di`(`0x9000:0x0` 即 `0x90000`) 处。

也就是 cpu 上电后，bios 将第一个可启动设备的第一个 `512字节` 先拷贝到 `0x7c00` 处，接着跳转到 `0x7c00` 处执行，然后 0x7c00 中的部分代码又将该部分（`512字节`）拷贝到 `0x90000` 处

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091106175.png)

`ljmp $INITSEG, $go` 长跳转，直接跳转至 `0x9000:go` 处。

##### 跳转至 0x9c000 处对内存进行分配

接着，一个远跳 跳转至 go 处，继续执行。

> [!tips]
> 短跳：可跳至距当前位置 128 字节内以内的范围（CS 不变，(E)IP 变化）
> 近跳：可跳转至当前段内的任意位置（CS 不变，(E)IP 变化）
> 远跳：可跳转至任意位置（CS 变，(E)IP 变）

接着执行 `go` 处的代码。由于是长跳转，所以 cs 的值为 `$INITSEG`，也就是 `0x9000`。

所以此处代码也就是将 `ds`、`es`、`ss` 设置为移动后代码所在的段处，并且将堆栈段 设置为 `0x9000:0` - `0x9000:0xff00`，即 `0x90000`- `0x9ff00`

```c
 go:  mov   %cs, %ax #将ds，es，ss都设置成移动后代码所在的段处(0x9000)
     mov %ax, %ds
     mov %ax, %es
     # put stack at 0x9ff00.
     mov %ax, %ss
     mov $0xFF00, %sp    # arbitrary value >>512
```

接着阅读下面代码，都是 mov 操作，将 ax 的值给 ds、es 和 ss 寄存器，而 ax 等于多少呢？由上一条远跳指令可知，cs 寄存器值被更改为 0x9000，所以 ds、es、ss 值均为 0x9000。

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204022109389.png)

ss 为栈段寄存器，后面要配合栈基址寄存器 sp 来表示此时的栈顶地址。而此时 sp 寄存器被赋为 0xFF00,所以目前栈顶地址就是 ss:ip 所指向的地址 0x9FF00。

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204022115634.png)

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091424605.png)

##### 将硬盘剩余部分也放到内存中

```c
 load_setup:
     mov $0x0000, %dx    # drive 0, head 0
     mov $0x0002, %cx    # sector 2, track 0
     mov $0x0200, %bx    # address = 512, in INITSEG
     .equ    AX, 0x0200+SETUPLEN
     mov     $AX, %ax    # service 2, nr of sectors
     int $0x13           # read it
```

看一下 `int $13`，BIOS int 13h 中断也叫直接磁盘服务（Direct Disk Service）其对应。

> 此处 int 为中断，`int 0x13`，发起 `0x13` 号中断。
> 当中断发生后，BIOS 会根据中断编号去找对应的中断函数入口地址并跳转过去执行，相当于此处执行了一个函数。

- ax=0x0204 其功能描述为读扇区，扇区数为 4
- bx=0x0200 其功能配置 es 寄存器组成缓冲区地址，es:bx 缓冲区地址
- cx=0x0002 分为 ch 和 cl，ch 柱面，cl 扇区。即 0 柱面 2 扇区。
- dx=0x0000 分为 dh 和 dl，dh 磁头，dl 驱动器 00H~7FH 软盘；80H~FFH 硬盘

也就是从 `软盘驱动器0` 的 `0柱面2扇区` 开始，拷贝 `4扇区` 到 `es:bx`，也就是 `0x9000:0x0200` 即 `0x90200`

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091342057.png)

拷贝的是什么东西呢？硬盘中 1-5 扇区 共 4 扇区的代码。

```c
load_setup:
	...
	jnc	ok_load_setup		# ok - continue
	mov	$0x0000, %dx
	mov	$0x0000, %ax		# reset the diskette
	int	$0x13
	jmp	load_setup
```

接着 因为 `AX>=0`，跳转到 `ok_load_setup` 执行。

```c
ok_load_setup:
	...
	mov	$SYSSEG, %ax
	mov	%ax, %es		# segment of 0x010000
	call    read_it
	...
	ljmp	$SETUPSEG, $0
```

这部分只看主要代码，其将剩下的从第 6 个扇区后面的 x 个扇区，加载到内存 `0x10000` 处，简单来说，就是将 system 代码挪了个地。

接着一个长跳转 `SETUPSEG`，将 CS 设置为 `0x9020`，EIP 设置为 `0x0` 也就是跳转到 `0x9020<<16 | 0` 即 `0x90200`

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091422374.png)

硬盘中数据是怎么分区的呢

通过 `Makefile` 和 `tools/build.sh` 配合完成，其中

- `bootsect.s` 编译成 bootsect，放在第 `1` 扇区
- `setup.s` 编译成 `setup`，放在 `2~5` 扇区
- 将剩下的 `head.s` 和其他代码编译成 `system`，放在随后的 240 个扇区？（不一定是 240 扇区）
  ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204022233895.png)

**总结一下：

cpu 上电后 bios 将第一个可启动分区的前 512 字节 (bootsect) 拷贝到 0x7c00 处，并跳转过去执行，接着 bootsect 又把自己搬到了 0x90000 处。

然后跳转过去，将 2 扇区 -5 扇区（setup）共 4 扇区拷贝到 0x90200 处。接着将 6 扇区以后（system）拷贝到 0x10000 处。最后跳转到 `0x90200` 处执行**。

> 在分析的过程中，我们借助 `gdb`，`target remote :1234`，
> b \*0x7c00 在 0x7c00 处加个断点
> b \*0x90200 在 0x90200 处加个断点
> 当跳转到 0x90200 处，通过命令 `x/512b 0x90200`,`x/256h 0x90200` 处值，发现就是我们拷贝过去的第一个扇区（bootsect）
> ![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206061437146.png)

##### setup 获取系统数据、修改内存布局

setup.s 负责从 BIOS 中获取系统数据，并将这些数据放到系统内存的合适地方。这段代码询问 bios 有关内存/磁盘/其它参数，并将这些参数存到一个“安全的”地方：0x90000-0x901FF。

```c
.equ SETUPSEG, 0x9020 # this is the current segment
...
ljmp $SETUPSEG, $_start
_start:
 mov %cs,%ax
 mov %ax,%ds
 mov %ax,%es
```

跳转到相对于 `0x90200` 处偏移 `_start` 的位置，也就是当前 `_start` 代码的位置，更新当前 ds、es 寄存器值为 `0x9020`。

接着往下看代码，都是形似 `mov %ax,$xxa;mov %bx,$xxb;mov %cx,$xxc;mov %dx,$xxd;int xxe;` 都是通过 bios 中断获取信息，然后将其存在内存中。

存在哪呢？实际上是保存在 ds 寄存器值为 都为 cs 寄存器的值，偏移为 0 处（`cs:0 = cs<<16+0`）。

```c
ljmp $SETUPSEG, $_start
_start:
 mov %cs,%ax
 mov %ax,%ds
 ...
 int $0x10 # save it in known place, con_init fetches
 mov %dx, %ds:0 # it from 0x90000.
```

最终会通过 bios 获取到这些数据，将之存到起始为 0x90000 的位置。

| 内存地址 | 长度 (字节) | 名称          |
| -------- | ---------- | ------------- |
| 0x90000  | 2          | 光标位置      |
| 0x90002  | 2          | 扩展内存数    |
| 0x90004  | 2          | 显示页面      |
| 0x90006  | 1          | 显示模式      |
| 0x90007  | 1          | 字符列数      |
| 0x90008  | 2          | 未知          |
| 0x9000A  | 1          | 显示内存      |
| 0x9000B  | 1          | 显示状态      |
| 0x9000C  | 2          | 显卡特性参数  |
| 0x9000E  | 1          | 屏幕行数      |
| 0x9000F  | 1          | 屏幕列数      |
| 0x90080  | 16         | 硬盘 1 参数表 |
| 0x90090  | 16         | 硬盘 2 参数表 |
| 0x901FC  | 2          | 根设备号      |

将以上信息存储到 0x90000 处后，将关闭中断。因为后面我们要自己实现中断，并且将 bios 的中断向量表破坏掉，所以这个时候是不允许中断进来的。

```c
  ...
  cli
  ...
```

看下面的部分，是不很熟悉 `movsw`，是一个数据传送指令，将一段数据从源地址传送到目的地址，详见 [指令](#%E6%8C%87%E4%BB%A4)。

```c
	mov	$0x0000, %ax
	cld			# 'direction'=0, movs moves forward
do_move:
	mov	%ax, %es	# destination segment
	add	$0x1000, %ax
	cmp	$0x9000, %ax
	jz	end_move
	mov	%ax, %ds	# source segment
	sub	%di, %di
	sub	%si, %si
	mov 	$0x8000, %cx
	rep
	movsw
	jmp	do_move
```

这段代码，也就是将 system 模块移动到新的位置，新位置起始为 0。

与以下 c 代码相同。

```c
char add[0x90000];
int j=0;
for(int i=0x10000; i<0x90000;i++){
	add[j++] = add[i];
}
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091459373.png)

重新规划后内存布局如下，

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091501381.png)

#### 实模式切换到保护模式 (分段机制)

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204041906123.png)

实模式与保护模式的第一个区别：物理地址计算方式不同

实模式下：$物理地址 = 段寄存器中的地址<<16 + 偏移地址$。

保护模式下：段寄存器中 存的是段选择子，段选择子去全局描述符中寻找段选择符，从中取出段基地址 再加上偏移地址才是物理地址。

> “计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”。
> "Any problem in computer science can be solved by anther layer of indirection."

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206061651461.png)

那么是如何从实模式（16 位）切换到保护模式（32 位）的呢？

[LGDT/LIDT](https://www.felixcloutier.com/x86/lgdt:lidt) 根据操作数的值大小确定 LGDT/LIDT 寄存器的结构

> 在 16 位操作数下：IDTR(Limit) <- SRC[0:15],IDTR(Base) <- SRC[16:47] & 00FFFFFFH;
>
> 在 32 位操作数下：IDTR(Limit) <- SRC[0:15],IDTR(Base) <- SRC[16:47];
>
> 在 64 位操作数下：IDTR(Limit) <- SRC[0:15],IDTR(Base) <- SRC[16:79];
>
> 那么 IDTR 寄存器是多么大的呢？

```c
lidt	[idt_48]		; load idt with 0,0
lgdt	[gdt_48]		; load gdt with whatever appropriate
```

`lidt [idt_48]` 汇编指令将 `idt_48` 处的 **48** 字节加载至 ldtr 寄存器中

`lgdt [gdt_48]` 汇编指令将 `gdt_48` 处的 **48** 字节加载至 lgtr 寄存器中

```c
idt_48:
	dw	0			; idt limit=0
	dw	0,0			; idt base=0L

gdt_48:
    dw	0x800		; gdt limit=2048, 256 GDT entries
    dw	512+gdt,0x9	; gdt base = 0X9xxxx  0x9 << 32 + (512 +gdt(gdt为在此文件中的偏移))

```

gdtr 寄存器结构

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204040902532.png)

gdt 处便为全局描述符在内存中的位置了，可以看出，一共有三段，第一段为 dummy，第二段为代码段描述符（可读可执行），第三段为数据段描述符（可读可写）

```c
gdt:
	dw	0,0,0,0		; dummy

	dw	0x07FF		; 8Mb - limit=2047 (2048*4096=8Mb)
	dw	0x0000		; base address=0
	dw	0x9A00		; code read/exec
	dw	0x00C0		; granularity=4096, 386

DATA_DESCRIPTOR:
	dw	0x07FF		; 8Mb - limit=2047 (2048*4096=8Mb)
	dw	0x0000		; base address=0
	dw	0x9200		; data read/write
	dw	0x00C0		; granularity=4096, 386
```

其中一个描述符的格式如下，根据段描述符结构我们可以看出三个段基址均为 0

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204040913252.png)

内存中的分布如下：

- 栈顶地址 0x9ff00
- 硬盘的第 2-5 个扇区在 0x90200 处，其中 gdt 和 idt 放在 0x90200 开始的某一个地方，地址（0x90200 + gdt/idt 偏移）存储在 gdtr/idtr 寄存器中
- 0x90000 存放的是从 bios 中读取的临时存放的变量
- 0x0~0x80000 存放的是操作系统的全部代码

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091544587.png)

后面 **却换到保护模式后，段寄存器（cs,ds,ss）中存储的是段选择子，段选择子去全局描述符中寻找段描述符，从中取出段基址**

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204040934036.png)

```c
    mov	al,0xD1		; command write
    out	0x64,al

    mov	al,0xDF		; A20 on
    out	0x60,al
```

这段代码的意思是打开 A20 地址线，那么问题来了，什么是 A20 地址线？为什么要开？

A20 地址线是为了突破 20 位地址线的限制，变成 32 位可用，所以即使地址线有 32 位了，但是你如果不手动开启，还是会限制 20 位可用。现在的 CPU 位数都 32 位、64 位，为了兼容以前的 20 位地址总线，便有了此选项。

接着往下走，这一堆代码是 **对可变成中断控制器 8259 芯片的编程**。

```c
	mov	al,0x11		; initialization sequence
	out	0x20,al		; send it to 8259A-1
	dw	0x00eb,0x00eb		; jmp $+2, jmp $+2
	out	0xA0,al		; and to 8259A-2
	dw	0x00eb,0x00eb
	mov	al,0x20		; start of hardware int's (0x20)
	out	0x21,al
	dw	0x00eb,0x00eb
	mov	al,0x28		; start of hardware int's 2 (0x28)
	out	0xA1,al
	dw	0x00eb,0x00eb
	mov	al,0x04		; 8259-1 is master
	out	0x21,al
	dw	0x00eb,0x00eb
	mov	al,0x02		; 8259-2 is slave
	out	0xA1,al
	dw	0x00eb,0x00eb
	mov	al,0x01		; 8086 mode for both
	out	0x21,al
	dw	0x00eb,0x00eb
	out	0xA1,al
	dw	0x00eb,0x00eb
	mov	al,0xFF		; mask off all interrupts for now
	out	0x21,al
	dw	0x00eb,0x00eb
	out	0xA1,al
```

在对 8259 芯片重新编程后，PIC 请求号和中断号的对应关系如下：

| PIC 请求号 | 中断号 |     用途     |
| :--------: | :----: | :----------: |
|    IRQ0    |  0x20  |   时钟中断   |
|    IRQ1    |  0x21  |   键盘中断   |
|    IRQ2    |  0x22  |  接连从芯片  |
|    IRQ3    |  0x23  |    串口 2    |
|    IRQ4    |  0x24  |    串口 1    |
|    IRQ5    |  0x25  |    并口 2    |
|    IRQ6    |  0x26  |  软盘驱动器  |
|    IRQ7    |  0x27  |    并口 1    |
|    IRQ8    |  0x28  |  实时钟中断  |
|    IRQ9    |  0x29  |     保留     |
|   IRQ10    |  0x2a  |     保留     |
|   IRQ11    |  0x2b  |     保留     |
|   IRQ12    |  0x2c  |   鼠标中断   |
|   IRQ13    |  0x2d  | 数学协处理器 |
|   IRQ14    |  0x2e  |   硬盘中断   |
|   IRQ15    |  0x2f  |     保留     |

```c
	mov	ax,0x0001	; protected mode (PE) bit
	lmsw	ax		; This is it;
	jmp	8:0			; jmp offset 0 of segment 8 (cs)
```

前两行，将 cr0 这个寄存器的位 0 置 1，模式就从是模式切换到保护模式了。

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204040953220.png)

继续，后面一个远跳，操作数为 **8:0**

在上面两行代码结束后，此时已经是保护模式了，保护模式下寻址方式变了，段寄存器中的值为段选择子，段选择的结构如下 ![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204040957062.png)

0x8，二进制 1000，对应着 描述符的索引为 1，也就是去全局描述符 gdt 中找索引为 1 的段描述符。但是呢，在前面我们分析过，全局描述符中的有三项，第一项都为 0，第二三项段基址都为 0，所以段基址为 0，偏移也是 0，所以这个跳转指令，**就是跳转到内存地址的 0 地址处**。

0 地址处存放的是我们 system 这个大模块，而 system 这个模块由 head.s、main.c 及其余模块的操作系统代码合并来的（如何知道的，查看 Makefile 中 tools/system 可看到由 head.o 和 main.o 还有其余模块组成）。

##### 分段与分页

先来一个精髓，开启分段机制和分页机制后

逻辑地址由 **段选择子** 和 **段内偏移** 组成，根据段选择子在 GDT 中找到段基址，类似于数组 `a[b]=c`，a 就是全局描述符表 GDT，b 就是段选择子，c 就是段基址

线性地址由 上文产生的 段基址 + 段内偏移 组成。

线性地址又分为三部分：页目录项、页表项、页内偏移

根据 **页目录项** 在 **页目录表** 中找出 **页表**

根据 **页表项** 在 **页表** 中找出 **页**

在加上 **页内偏移** 就是实际的物理地址

![Pasted image 20221206194827.png](/img/user/Resources/Images/Pasted%20image%2020221206194827.png)

由最外层的 Makefile 可得，system 由 `boot/head.o`、`init/main.o` 及其它组成，并且 system 在 bootsect 中被搬运至 0 处。所以在 setup 中跳转到 0 处，也就是跳转到 head 中了。

接着就开始研究 head.s 了。此处汇编风格变为 AT&T

```c
pg_dir:
.globl startup_32
startup_32:
 movl $0x10,%eax
 mov %ax,%ds
 mov %ax,%es
 mov %ax,%fs
 mov %ax,%gs
 lss stack_start,%esp
 call setup_idt
 call setup_gdt
 movl $0x10,%eax # reload all the segment registers
 mov %ax,%ds # after changing gdt. CS was already
 mov %ax,%es # reloaded in 'setup_gdt'
 mov %ax,%fs
 mov %ax,%gs
 lss stack_start,%esp
```

看下代码，刚开始有个标号 `pg_dir`，这个是页目录，之后设置分页机制的时候，页目录会放这，覆盖这里得代码。

往下走，就是给 eax 赋值为 0x10，给 ds、es、fs、gs 赋值 0x10(0b0001_0000)，也就是 0b0001_0 即 2，索引为 2 的段为数据段。

然后 `lss stack_start,%esp`，将 `stack_start` 高位给 ss，低 16 位给 esp。（之前是 0x9ff00，现在要换到\_stack_start）

stack_start 这个标号在 sched.c 中，（关于为什么是 start_start 而不是\_stack_start 这个是因为 [cdecl 调用规约中第 4 条:**编译后的函数名前缀以一个下划线字符开始**](https://zh.wikipedia.org/wiki/X86%E8%B0%83%E7%94%A8%E7%BA%A6%E5%AE%9A#:~:text=%E5%AF%84%E5%AD%98%E5%99%A8ST0%E4%B8%AD-,%E7%BC%96%E8%AF%91%E5%90%8E%E7%9A%84%E5%87%BD%E6%95%B0%E5%90%8D%E5%89%8D%E7%BC%80%E4%BB%A5%E4%B8%80%E4%B8%AA%E4%B8%8B%E5%88%92%E7%BA%BF%E5%AD%97%E7%AC%A6,-%E8%B0%83%E7%94%A8%E8%80%85)）

```c
long user_stack[4096>>2];

struct {
 long *a;
 short b;
} stack_start = { &user_stack[4096>>2], 0x10};
```

也就是将高位 0x10 给 ss，低位 `user_stack [4096>>2]` 的元素的下一个地址值给 esp，0x10 也就是 0x0001_0000 表示指向全局描述符的第 0x0001_0 个段，也就是第 2 个段为 data 段，其基地址为 0。

`call setup_idt` 设置中断描述符表 `call setup_gdt` 设置全局描述符表

然后又重新设置一遍，为什么要重新设置？在上面设置中断/全局描述符表的时候，修改了 gdt，所以要重新设置才会生效。

接下来重点看 `setup_idt`/`setup_gdt`。

配置完 idt 和 gdt 后，接着继续往下走，跳转到 `after_page_tables` 后，先是几个 `push`，其中包含了 c 语言的世界的地址，然后一个跳转到 `setup_paging`，给 `ecx` 分配大小为 5\*1024（5pages）,然后将 eax 清零，edi 清零。接着将 al 中的数据（0）填充到 edi 起始的位置（0）处，方向为正向，大小为 5\*1024\*4。（也就是说 **从零地址开始前 20k 内存清零**）

> cld;rep;stosl
> cld 设置 edi 或同 esi 为递增方向，rep 做 (%ecx) 次重复操作，stosl 表示 edi 每次增加 4,这条语句达到按 4 字节清空前 5\*1024\*4 字节地址空间的目的。

```c
	jmp after_page_tables
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $_start
	jmp setup_paging
L6:
	jmp L6			# main should never return here, but
				# just in case, we know what happens.

setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
	movl $pg0+7,_pg_dir		/* set present bit/user r/w */
	movl $pg1+7,_pg_dir+4		/*  --------- " " --------- */
	movl $pg2+7,_pg_dir+8		/*  --------- " " --------- */
	movl $pg3+7,_pg_dir+12		/*  --------- " " --------- */
	movl $pg3+4092,%edi
	movl $0xfff007,%eax		/*  16Mb - 4096 + 7 (r/w user,p) */
	std
1:	stosl			/* fill pages backwards - more efficient :-) */
	subl $0x1000,%eax
	jge 1b
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
	ret			/* this also flushes prefetch-queue */

```

接着了解一下分页，在保护模式下开启分段机制后，在代码中给出一个内存地址，要先经过分段机制的转换，才能得到最终的物理地址。

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204061044176.png)

这个是没有开启分页机制的情况下，开启分页后又会 **多一步转换**

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204061044464.png)

分段机制将 **逻辑地址** 转变 **为线性地址**

分页机制在分段机制的基础上将 **线性地址转为物理地址**

分页机制将一个 32 位线性地址分为三部分：

10 位：10 位：12 位，分别为页目录表：页表：页内偏移。

通过高 10 位去页目录表中找出索引对应的页目录项，在该目录项内 中 10 位找出索引对应的页表项，其对应的值在加上页内偏移就是实际的物理地址。

这一切的操作由一个计算机硬件 MMU（Memory Management Unit，内存管理单元）将线性地址（虚拟地址）转换为物理地址

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204061938387.png)

这个页表方案叫二级页表，第一级叫 **页目录表 PDE**，第二级叫 **页表 PTE**，结构如下

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204062005230.png)

然后将 CR0 寄存器的 PG（31 位）置 1，即可开启分页机制，之后 MMU 就可以帮我们进行分页的转换了。

此后指令中的内存地址，就先要经过分段机制的转换，在经过分页机制的转换，最终变成物理地址。

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204062031578.png)

```c
setup_paging:
	movl $1024*5,%ecx		/* 5 pages - pg_dir+4 page tables */
	xorl %eax,%eax
	xorl %edi,%edi			/* pg_dir is at 0x000 */
	cld;rep;stosl
	movl $pg0+7,_pg_dir		/* set present bit/user r/w */
	movl $pg1+7,_pg_dir+4		/*  --------- " " --------- */
	movl $pg2+7,_pg_dir+8		/*  --------- " " --------- */
	movl $pg3+7,_pg_dir+12		/*  --------- " " --------- */
	movl $pg3+4092,%edi
	movl $0xfff007,%eax		/*  16Mb - 4096 + 7 (r/w user,p) */
	std
1:	stosl			/* fill pages backwards - more efficient :-) */
	subl $0x1000,%eax
	jge 1b
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
	ret			/* this also flushes prefetch-queue */
```

前面我们了解到前 4 行代码是 **从 0 开始，将前 1024\*5\*4 字节空间清零**

接着往下走，四个 movl 指令将 `$pg0+7` 这个地址给到 `_pg_dir`（0 地址处），后面依次将 pg1、pg2、pg3 地址给 `_pg_dir + 4、8、12` 处，构建页目录表。

而 pg0、1、2、3 分别为 0x1000、0x2000、0x3000、0x4000

```c
.org 0x1000
pg0:

.org 0x2000
pg1:

.org 0x3000
pg2:

.org 0x4000
pg3:

.org 0x5000
```

所以页目录表中存储的数据也就为 0x1007、0x2007、0x3007、0x3007，对应页表地址为 1、2、3、4（0x1007 >> 12）

![](https://cdn.jsdelivr.net/gh/Gonglja/imgur/img/202204070903895.png)

接着往下走 给 edi 赋值 pg3+4092 也就是 0x4ffc，为什么是这个？没搞清楚

给 eax 赋值 0xfff007，对应的也就是最大 4096 个 4k，也就是 16M 内存，然后一个 std;stosl 命令表示以 edi 为起始位置 0x4ffc，每次减小 4 字节，将对应的 4k 地址写入到页表中。然后 eax 减掉一个 4k，jge 大于等于转移，然后跳转到执行 stosl，这样就将 16M 内的 4k 空间地址写入到页表中。

```c
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
	ret			/* this also flushes prefetch-queue */
```

先给 eax 清 0，然后设置 cr3，设置页目录表地址。

接着就是将 cr0 的第 31 位置 1，写回 cr0，开启分页机制。

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206081846349.png)

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206082006590.png)

设置 pg_dir，到 cr3，将 cr0 的最高位置为 1。

```c
	xorl %eax,%eax		/* pg_dir is at 0x0000 */
	movl %eax,%cr3		/* cr3 - page directory start */
	movl %cr0,%eax
	orl $0x80000000,%eax
	movl %eax,%cr0		/* set paging (PG) bit */
	ret			/* this also flushes prefetch-queue */
```

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206090010988.png)

##### 如何进入 main

```c
after_page_tables:
	pushl $0		# These are the parameters to main :-)
	pushl $0
	pushl $0
	pushl $L6		# return address for main, if it decides to.
	pushl $main
	jmp setup_paging
L6:
	jmp L6
```

在 after_page_tables 中连着 5 个 push，将数据依次压入栈，最后的结构如下

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091558424.png)

注意 setup_paging 的最后一条命令是 ret，ret 被叫做返回指令，返回指令的话肯定得有返回的地址，计算机会机械的把栈顶的元素当作返回地址。在具体的说，就是将 esp 寄存器的值给到 eip 中，而 cs:eip 就是 CPU 要执行的下一条指令的地址。而栈顶此时存放的为 main(start) 函数的地址，所以 ret 后就会跳转到 main(start) 中了。其中 **L6 会作为 main 的返回值**，但 main(start) 是不会返回的，其它 **三个值本意是作为 main(start) 函数的参数**，但没有用到。

> 关于 ret 指令，其实 Intel CPU 是配合 call 设计的，有关 call 和 ret 指令，即调用和返回指令，可以参考 Intel 手册：
> Intel 1 Chapter 6.4 CALLING PROCEDURES USING CALL AND RET

到此，汇编部分就结束了。主要有如下操作，

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202204071044648.png)

整个内存分布如下：

![](https://note-1251905184.cos.ap-shanghai.myqcloud.com/img/202206091610884.png)

#### 大战前期的初始化工作

整个 c 语言世界分为四部分，

1. 参数的取值和计算
2. 各种初始化 init 操作
3. 切换用户态模式，并在一个新的进程中做一个最终的初始化 init
4. 死循环，如果操作系统没有任务运行，则一直陷入这个死循环无法自拔

```c
void main(void)		/* This really IS void, no error here. */
{			/* The startup routine assumes (well, ...) this */
/*
 * Interrupts are still disabled. Do necessary setups, then
 * enable them
 */
	// 1. 参数的取值和计算
	ROOT_DEV = ORIG_ROOT_DEV;
	drive_info = DRIVE_INFO;
	memory_end = (1<<20) + (EXT_MEM_K<<10);
	memory_end &= 0xfffff000;
	if (memory_end > 16*1024*1024)
		memory_end = 16*1024*1024;
	if (memory_end > 12*1024*1024)
		buffer_memory_end = 4*1024*1024;
	else if (memory_end > 6*1024*1024)
		buffer_memory_end = 2*1024*1024;
	else
		buffer_memory_end = 1*1024*1024;
	main_memory_start = buffer_memory_end;

	// 2. 各种初始化 init 操作
#ifdef RAMDISK
	main_memory_start += rd_init(main_memory_start, RAMDISK*1024);
#endif
	mem_init(main_memory_start,memory_end);
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();

	// 3. 切换用户态模式，并在一个新的进程中做一个最终的初始化 init
	sti();
	move_to_user_mode();
	if (!fork()) {		/* we count on this going ok */
		init();
	}
/*
 *   NOTE!!   For any other task 'pause()' would mean we have to get a
 * signal to awaken, but task0 is the sole exception (see 'schedule()')
 * as task 0 gets activated at every idle moment (when no other tasks
 * can run). For task0 'pause()' just means we go check if some other
 * task can run, and if not we return here.
 */
	// 4. 死循环，如果操作系统没有任务运行，则一直陷入这个死循环无法自拔
	for(;;) pause();
}
```

第一部分：参数的取值和计算内存边界

在上文中，通过 setup.s 调用 bios 中断，获取到一些设备参数信息，存放在 0x90000 处。

在此处将参数取出，并计算内存边界

| 内存地址 | 长度 (字节) | 名称 |
| -------- | ---------- | ------------- |
| 0x90000 | 2 | 光标位置 |
| 0x90002 | 2 | 扩展内存数 |
| 0x90004 | 2 | 显示页面 |
| 0x90006 | 1 | 显示模式 |
| 0x90007 | 1 | 字符列数 |
| 0x90008 | 2 | 未知 |
| 0x9000A | 1 | 显示内存 |
| 0x9000B | 1 | 显示状态 |
| 0x9000C | 2 | 显卡特性参数 |
| 0x9000E | 1 | 屏幕行数 |
| 0x9000F | 1 | 屏幕列数 |
| 0x90080 | 16 | 硬盘 1 参数表 |
| 0x90090 | 16 | 硬盘 2 参数表 |
| 0x901FC | 2 | 根设备号 |

第二部分：各种初始化 init 操作，包括内存初始化，中断初始化，块设备初始化，字符设备初始化，tty 初始化，调度初始化等

```c
void main() {
	...
	mem_init(main_memory_start,memory_end);
	trap_init();
	blk_dev_init();
	chr_dev_init();
	tty_init();
	time_init();
	sched_init();
	buffer_init(buffer_memory_end);
	hd_init();
	floppy_init();
	...
}
```

第三部分：切换用户态模式，并在一个新的进程中做一个最终的初始化 init

init() 函数会创建一个进程，设置终端的标准 IO，并且在创建出一个执行 shell 程序的进程用来接收用户的命令，到此就进入了真正的系统中。

```c
void main() {
	...
	sti();
	move_to_user_mode();
	if (!fork()) {		/* we count on this going ok */
		init();
	}
	...
}
```

第四部分：死循环，如果操作系统没有任务运行，则一直陷入这个死循环无法自拔

```c
void main() {
	...
	for(;;) pause();
}
```

##### 管理内存前的三个边界值

```c
void main(void) {
    ...
    memory_end = (1<<20) + (EXT_MEM_K<<10);
    memory_end &= 0xfffff000;
    if (memory_end > 16*1024*1024)
        memory_end = 16*1024*1024;
    if (memory_end > 12*1024*1024) 
        buffer_memory_end = 4*1024*1024;
    else if (memory_end > 6*1024*1024)
        buffer_memory_end = 2*1024*1024;
    else
        buffer_memory_end = 1*1024*1024;
    main_memory_start = buffer_memory_end;
    ...
}
```

其实这段代码就是根据不同的 `memory_end` 大小去分配不同大小的 `buffer_memory_end`，

假设内存有 16M，则 `memory_end = 16*1024*1024;` `buffer_memory_end = 4*1024*1024;`，整个内存划分也就是如下图所示

![Pasted image 20221129181317.png](/img/user/Resources/Images/Pasted%20image%2020221129181317.png)

得到 `memory_end`、`main_memory_start`、`buffer_memory_end` 边界值后，后续怎么设置就看 `mem_init` 和 `buffer_init` 了。

```c
void main() {
	...
	mem_init(main_memory_start,memory_end);
	...
	buffer_init(buffer_memory_end);
	...
}
```

##### mem_init

这部分的代码其实看起来很简单，通过 mem_map 数组去管理内存，怎么管理的呢？

简单来说：通过一个数组去标记内存块（4kbytes）有没有被使用。4k 内存通常被叫为 1 页，这种管理方式叫 **分页管理**，就是把内存分成一页一页的单位去管理。

看代码，代码先对 mem_map 数组全部赋值为 USED，然后在对其中的一部分赋值为 0。其中 USED 表示内存被占用，占用了 100 次；0 表示未使用。

PAGING_PAGES 中 PAGING_MEMORY 为什么要右移 12 位，2^12 = 4kbytes，为分页管理的最小单位

```c
/* these are not to be changed without changing head.s etc */
#define LOW_MEM 0x100000
#define PAGING_MEMORY (15*1024*1024)
#define PAGING_PAGES (PAGING_MEMORY>>12)
#define MAP_NR(addr) (((addr)-LOW_MEM)>>12)
#define USED 100

static long HIGH_MEMORY = 0;
static unsigned char mem_map [ PAGING_PAGES ] = {0,};

// start_mem =  4*1024*1024
// end_mem   = 16*1024*1024
void mem_init(long start_mem, long end_mem)
{
	int i;

	HIGH_MEMORY = end_mem;
	for (i=0 ; i<PAGING_PAGES ; i++)
		mem_map[i] = USED;
	i = MAP_NR(start_mem);
	end_mem -= start_mem;
	end_mem >>= 12;
	while (end_mem-->0)
		mem_map[i++]=0;
}
```

还是假设内存有 16M，上述代码的前部分是对 15M 内存全部标记为 USED，表示内存被使用；后部分从 start_mem 开始到 end_mem 全部标记为 0，表示内存未被使用

![Pasted image 20221129184042.png](/img/user/Resources/Images/Pasted%20image%2020221129184042.png)

如上图所示：1M 以下没有记录，这个区域是内核代码所在，无需也没有权限管理的；

1M-4M 区间是缓冲区，这个地方不是主内存区域，因此直接被标记为 USED，效果就是无法分配。

4M 以上是主内存区域，在初始化时无任何程序申请，因此都为零。

应用程序是如何申请内存呢？此处暂不展开，简单看一下申请内存的过程中，是如何使用 mem_map 这个结构的。

在 memory.c 中有一个函数 get_free_page() 用于在主内存去中申请一页空闲内存页，并返回物理内存页的起始地址。

比如在 fork 子进程的时候，会调用 copy_process 函数来赋值进程的结构信息，其中一个重要的步骤就是申请一页内存，用于存放进程结构信息 task_struct

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p;
	int i;
	struct file *f;

	p = (struct task_struct *) get_free_page();
	...
}
```

看一下 get_free_page 的实现，是内联汇编代码，先不用看懂，注意里面有 mem_map 结构的使用

```c
// 选择 mem_map 中的首个空闲页面，并标记为已使用
unsigned long get_free_page(void)
{
register unsigned long __res asm("ax");

__asm__("std ; repne ; scasb\n\t"
	"jne 1f\n\t"
	"movb $1,1(%%edi)\n\t"
	"sall $12,%%ecx\n\t"
	"addl %2,%%ecx\n\t"
	"movl %%ecx,%%edx\n\t"
	"movl $1024,%%ecx\n\t"
	"leal 4092(%%edx),%%edi\n\t"
	"rep ; stosl\n\t"
	" movl %%edx,%%eax\n"
	"1: cld"
	:"=a" (__res)
	:"0" (0),"i" (LOW_MEM),"c" (PAGING_PAGES),
	"D" (mem_map+PAGING_PAGES-1)
	);
return __res;
}
```

##### trap_init

在 mem_init 内存初始化完成之后，有这样一行代码，中断设置初始化

```c
void main(void) {
	...
	trap_init();
	...
}
```

打开实现，发现里面都是类似的代码，其实里面就两个看似是函数的东西

set_trap_gate 和 set_system_gate

```c
void trap_init(void)
{
	int i;

	set_trap_gate(0,&divide_error);
	set_trap_gate(1,&debug);
	set_trap_gate(2,&nmi);
	set_system_gate(3,&int3);	/* int3-5 can be called from all */
	set_system_gate(4,&overflow);
	set_system_gate(5,&bounds);
	set_trap_gate(6,&invalid_op);
	set_trap_gate(7,&device_not_available);
	set_trap_gate(8,&double_fault);
	set_trap_gate(9,&coprocessor_segment_overrun);
	set_trap_gate(10,&invalid_TSS);
	set_trap_gate(11,&segment_not_present);
	set_trap_gate(12,&stack_segment);
	set_trap_gate(13,&general_protection);
	set_trap_gate(14,&page_fault);
	set_trap_gate(15,&reserved);
	set_trap_gate(16,&coprocessor_error);
	for (i=17;i<48;i++)
		set_trap_gate(i,&reserved);
	set_trap_gate(45,&irq13);
	outb_p(inb_p(0x21)&0xfb,0x21);
	outb(inb_p(0xA1)&0xdf,0xA1);
	set_trap_gate(39,&parallel_interrupt);
}
```

实际上这两个并不是函数，而是两个宏，两个宏都指向了相同的宏定义 `_set_gate`

```c
#define _set_gate(gate_addr,type,dpl,addr) \
__asm__ ("movw %%dx,%%ax\n\t" \
	"movw %0,%%dx\n\t" \
	"movl %%eax,%1\n\t" \
	"movl ecx,current\n\t" \
	"je 1f\n\t" \
	"movw %%dx,%1\n\t" \
	"xchgl %%ecx,current\n\t" \
	"ljmp *%0\n\t" \
	"cmpl %%ecx,last_task_used_math\n\t" \
	"jne 1f\n\t" \
	"clts\n" \
	"1:" \
	::"m" (*&__tmp.a),"m" (*&__tmp.b), \
	"d" (_TSS(n)),"c" ((long) task[n])); \
}
```

CPU 规定，如果 ljmp 指令后面跟的是一个 tss 段，那么，会由硬件将当前各个寄存器的值保存在当前进程的 tss 中，并将新进程的 tss 信息加载到各个寄存器。

![Pasted image 20221202174010.png](/img/user/Resources/Images/Pasted%20image%2020221202174010.png)

简单说就是，**保存当前进程上下文，恢复下一个进程的上下文，跳过去**！

##### fork

首先看 fork 函数，这个函数实际上并不是一个真正的函数，为了高复用性，减少重复工作，而通过 宏 `_syscall0` 定义，展开后就是 完整的 fork 函数。

对系统的调用都是通过 `_syscallx` 实现的，其中的 `x` 就是区别，表示要传递的参数个数，

比如 `_syscall0(int, fork);` 没有参数，会生成 `int fork(void) ...` 函数

在比如 `_syscall3(int,write,int,fd,const char *,buf,off_t,count)` 有三个参数，会生成 `int write(int fd, const char* buf, off_t count)...` 函数

```c
// init/main.c
static inline fork(void) __attribute__((always_inline));
static inline _syscall0(int,fork)

// include/unistd.h
#define _syscall0(type,name) \
  type name(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_##name)); \
if (__res >= 0) \
	return (type) __res; \
errno = -__res; \
return -1; \
}

// 将参数带入后
#define _syscall0(int,fork) \
  int fork(void) \
{ \
long __res; \
__asm__ volatile ("int $0x80" \
	: "=a" (__res) \
	: "0" (__NR_fork)); \
if (__res >= 0) \
	return (int) __res; \
errno = -__res; \
return -1; \
}

// 也就是
int fork(void)
{
	long __res;
	__asm__ volatile ("int $0x80" : "=a" (__res) : "0" (__NR_fork));
	if (__res >= 0)
		return (int) __res;
	errno = -__res;
	return -1;
}
```

至于 `_syscallx` 中的内容，大同小异，主要就是 `int $0x80`，通过传递不同的参数，去执行不同的功能

比如：`_syscall(int, fork)` 展开后就是上述代码，其传递的参数为 `__NR_fork` 宏，也就是 2

```c
#define __NR_fork	2
```

不知还有印象没，在前面我们配置了序号为 0x80 的中断，其中断处理函数为 system_call 所以当触发 0x80 中断后，会跳转到 system_call 中断处理函数。

```c
set_system_gate(0x80, &system_call);
```

在中断处理函数 system_call 中，暂时仅需要关心这一句，它是一个函数指针数组。

`call *sys_call_table(,%eax,4)`

```c
// kernel/system_call.s
system_call:
	cmpl $nr_system_calls-1,%eax
	ja bad_sys_call
	push %ds
	push %es
	push %fs
	pushl %edx
	pushl %ecx		# push %ebx,%ecx,%edx as parameters
	pushl %ebx		# to the system call
	movl $0x10,%edx		# set up ds,es to kernel space
	mov %dx,%ds
	mov %dx,%es
	movl $0x17,%edx		# fs points to local data space
	mov %dx,%fs
	call *sys_call_table(,%eax,4)
	pushl %eax
	movl current,%eax
	cmpl $0,state(%eax)		# state
	jne reschedule
	cmpl $0,counter(%eax)		# counter
	je reschedule
```

所以当 `_syscall(int, fork)` 时，会触发 0x80 中断，之后进入 0x80 中断处理函数 `system_call`，传递的参数为 `__NR_fork` (也就是 2 )，在 sys_call_table 中索引为 2 调用的函数为 sys_fork

```c
fn_ptr sys_call_table[] = { sys_setup, sys_exit, sys_fork, sys_read,
	sys_write, sys_open, sys_close, sys_waitpid, sys_creat, sys_link,
	sys_unlink, sys_execve, sys_chdir, sys_time, sys_mknod, sys_chmod,
	sys_chown, sys_break, sys_stat, sys_lseek, sys_getpid, sys_mount,
	sys_umount, sys_setuid, sys_getuid, sys_stime, sys_ptrace, sys_alarm,
	sys_fstat, sys_pause, sys_utime, sys_stty, sys_gtty, sys_access,
	sys_nice, sys_ftime, sys_sync, sys_kill, sys_rename, sys_mkdir,
	sys_rmdir, sys_dup, sys_pipe, sys_times, sys_prof, sys_brk, sys_setgid,
	sys_getgid, sys_signal, sys_geteuid, sys_getegid, sys_acct, sys_phys,
	sys_lock, sys_ioctl, sys_fcntl, sys_mpx, sys_setpgid, sys_ulimit,
	sys_uname, sys_umask, sys_chroot, sys_ustat, sys_dup2, sys_getppid,
	sys_getpgrp, sys_setsid, sys_sigaction, sys_sgetmask, sys_ssetmask,
	sys_setreuid,sys_setregid, sys_iam, sys_whoami
};
```

在 sys_fork 中，首先 `find_empty_process` 找一个空的进程，然后调用 `copy_process` 完成进程结构的复制

```c
.align 2
sys_fork:
	call find_empty_process
	testl %eax,%eax
	js 1f
	push %gs
	pushl %esi
	pushl %edi
	pushl %ebp
	pushl %eax
	call copy_process
	addl $20,%esp
1:	ret
```

find_empty_process 分为三部分

- 首先判断 last_pid 是否小于 0，小于 0 溢出了，重新赋值为 1，起到保护的作用
- 第一个循环：从 NR_TASKS 个进程数组中，找出一个未使用的 pid
- 第二个循环，从 NR_TASKS 个进程数组中，找出一个未被使用的进程结构，返回索引下标

```c
long last_pid=0;
int find_empty_process(void)
{
	int i;

	repeat:
		if ((++last_pid)<0) last_pid=1;
		for(i=0 ; i<NR_TASKS ; i++)
			if (task[i] && task[i]->pid == last_pid) goto repeat;
	for(i=1 ; i<NR_TASKS ; i++)
		if (!task[i])
			return i;
	return -EAGAIN;
}
```

copy_process

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	struct task_struct *p;
	int i;
	struct file *f;

	p = (struct task_struct *) get_free_page();
	if (!p)
		return -EAGAIN;
	task[nr] = p;

	// NOTE!: the following statement now work with gcc 4.3.2 now, and you
	// must compile _THIS_ memcpy without no -O of gcc.#ifndef GCC4_3
	*p = *current;	/* NOTE! this doesn't copy the supervisor stack */
	p->state = TASK_UNINTERRUPTIBLE;
	p->pid = last_pid;
	p->father = current->pid;
	p->counter = p->priority;
	p->signal = 0;
	p->alarm = 0;
	p->leader = 0;		/* process leadership doesn't inherit */
	p->utime = p->stime = 0;
	p->cutime = p->cstime = 0;
	p->start_time = jiffies;
	p->tss.back_link = 0;
	p->tss.esp0 = PAGE_SIZE + (long) p;
	p->tss.ss0 = 0x10;
	p->tss.eip = eip;
	p->tss.eflags = eflags;
	p->tss.eax = 0;
	p->tss.ecx = ecx;
	p->tss.edx = edx;
	p->tss.ebx = ebx;
	p->tss.esp = esp;
	p->tss.ebp = ebp;
	p->tss.esi = esi;
	p->tss.edi = edi;
	p->tss.es = es & 0xffff;
	p->tss.cs = cs & 0xffff;
	p->tss.ss = ss & 0xffff;
	p->tss.ds = ds & 0xffff;
	p->tss.fs = fs & 0xffff;
	p->tss.gs = gs & 0xffff;
	p->tss.ldt = _LDT(nr);
	p->tss.trace_bitmap = 0x80000000;
	if (last_task_used_math == current)
		__asm__("clts ; fnsave %0"::"m" (p->tss.i387));
	if (copy_mem(nr,p)) {
		task[nr] = NULL;
		free_page((long) p);
		return -EAGAIN;
	}
	for (i=0; i<NR_OPEN;i++)
		if ((f=p->filp[i]))
			f->f_count++;
	if (current->pwd)
		current->pwd->i_count++;
	if (current->root)
		current->root->i_count++;
	if (current->executable)
		current->executable->i_count++;
	set_tss_desc(gdt+(nr<<1)+FIRST_TSS_ENTRY,&(p->tss));
	set_ldt_desc(gdt+(nr<<1)+FIRST_LDT_ENTRY,&(p->ldt));
	p->state = TASK_RUNNING;	/* do this last, just in case */
	return last_pid;
}
```

这个函数被分为几部分来分析：

get_free_page：找到内存结构 mem_map 中空闲的内存，并标记为 1，表示该页已经被使用，算出该页的内存起始地址，返回

于是乎，p 就有了一块空间，但是该内存中并没有数据。

首先将该地址记录在 进程管理结构 `task[]` 中

`*p = *current` **将当前进程的全部值 完全复制给即将创建的进程 p**，所以之后的内存图就是这样的。

![Pasted image 20221203235941.png](/img/user/Resources/Images/Pasted%20image%2020221203235941.png)

接着就是修改 进程的个性化数据，比如时间片 counter，优先级，上下文环境 tss。

此处注意一点 ss0 和 esp0，表示 0 特权级也就是内核态时 `ss:esp` 的指向

接下来就是 进程页表和段表的复制。

看一下 copy_mem ,

```c
int copy_process(int nr,long ebp,long edi,long esi,long gs,long none,
		long ebx,long ecx,long edx,
		long fs,long es,long ds,
		long eip,long cs,long eflags,long esp,long ss)
{
	...
	copy_mem(nr,p);
	...
}

```

https://mp.weixin.qq.com/s?__biz=Mzk0MjE3NDE0Ng==&mid=2247501866&idx=1&sn=64adec9179345945d095a1a1bdebcdac&chksm=c2c5b287f5b23b9175d8eacf7731b22823a576f78e14d8b93b2e8c9814bcb11076967d878a12&cur_album_id=2123743679373688834&scene=189#wechat_redirect

##### init

## 参考

1.  [调试 Linux 最早期的代码](https://mp.weixin.qq.com/s/cx_vaRTcC29h0pWkJPpqQQ)
2.  [Ubuntu20.04 编译安装 qemu](https://blog.csdn.net/weixin_45709295/article/details/120007503)
3.  [qemu 运行虚拟机无反应，只输出一行提示信息:VNC server running on 127.0.0.1:5900](https://blog.csdn.net/qq_36393978/article/details/118353939)
4.  [连接的时候指定-Ttext 和指定-Tmap.lds 的区别](http://blog.chinaunix.net/uid-26833883-id-3746968.html)
5.  [Linux dd 命令](https://www.runoob.com/linux/linux-comm-dd.html)
6.  [BIOS int 13H 中断介绍](http://www.only2fire.com/archives/87.html)
7.  [汇编指令——用 GDB 调试汇编](https://zhuanlan.zhihu.com/p/259625135)
8.  [利用 BIOS 中断 INT 0x10 显示字符和字符串](https://blog.csdn.net/judyge/article/details/52289231)
