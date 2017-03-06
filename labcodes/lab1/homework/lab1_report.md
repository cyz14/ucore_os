# Lab1 实验报告

## 练习1
1. 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

### create kernel
```makefile
# 设置kernel的include查找文件夹
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/

# 设置kernel的src文件夹
KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm

# 设置编译kernel时的参数
KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

# create kernel target
kernel = $(call totarget,kernel)

# 设置kernel的依赖规则文件
$(kernel): tools/kernel.ld

# 链接所有的obj生成最终的kernel
# echo 输出信息
# ld 调用
# objdump 反汇编生成 kernel.asm, -S: source file
# objdump -t Print the symbol table entries of the file.
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

# 生成 bin/kernel
$(call create_target,kernel)

```

### create bootblock
```makefile
# list c files from boot/
bootfiles = $(call listf_cc,boot)
# compile every c file to objs
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

# generate target name
bootblock = $(call totarget,bootblock)

# ld all objs to get bootblock.o
# $(LDFLAGS): -nostdlib 不链接标准库
# -N: Set the text and data sections to be readable and writable.  Also,
# do not page-align the data segment, and disable linking against
# shared libraries.  If the output format supports Unix style magic
# numbers, mark the output as "OMAGIC".
# -e entry: Use entry as the explicit symbol for beginning execution of your program, rather than the default entry point.
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)
```

### create 'sign' tools

```makefile
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
```

### create ucore.img

```makefile
# Generate target name
UCOREIMG	:= $(call totarget,ucore.img)

# initialize and write content into ucore.img
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

# Generate bin/ucore.img
$(call create_target,ucore.img)
```

```shell
# 初始化img，所有内容清零
dd if=/dev/zero of=bin/ucore.img count=10000

# 把bootblock写入ucore.img的第一个扇区
dd if=bin/bootblock of=bin/ucore.img conv=notrunc

# 把ucore的kernel写入第一个扇区后面的地方
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
```

2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
	according to tools/sign.c
	主引导扇区内容不超过 510 byte，且地址为 510 处标记为 0x55,地址为 511 处标记为 0xAA

## 练习二

执行 lab1_result: `make lab1-mon`
自动开始调试，停在 0x7c00: cli 处

```as
(gdb) x /10i 0x7c02
=> 0x7c02:	xor    %ax,%ax
   0x7c04:	mov    %ax,%ds
   0x7c06:	mov    %ax,%es
   0x7c08:	mov    %ax,%ss
   0x7c0a:	in     $0x64,%al
   0x7c0c:	test   $0x2,%al
   0x7c0e:	jne    0x7c0a
   0x7c10:	mov    $0xd1,%al
   0x7c12:	out    %al,$0x64
   0x7c14:	in     $0x64,%al
```
和 bootasm.S 以及 bootblock.asm 中的命令稍有不同，比如源文件中的指令是 `xorw %ax, %ax `

在 lab1init 里面加上如下语句使 gdb 在每次停下时反汇编当前语句
```
define hook-stop
x/i $pc
end
```

## 练习三：分析bootloader进入保护模式的过程

bootmain.S

### 为何开启A20，如何开启A20
等待8042 Input buffer为空；
发送Write 8042 Output Port （P2）命令到8042 Input buffer；
等待8042 Input buffer为空；
将8042 Output Port（P2）得到字节的第2位置1，然后写入8042 Input buffer；

### 如何初始化GDT表

### 如何使能和进入保护模式


## 练习四
如何读取硬盘中的信息
判断ELF文件格式，bootmain.c

## 练习五：实现函数调用堆栈跟踪函数（需要编程）
填写两句代码

## 练习六：完善中断初始化和处理（需要编程）

8259 中断控制器的初始化
IDT 的初始化
时钟中断初始化
使能中断

vector.S 中定义了中断服务例程的起始地址