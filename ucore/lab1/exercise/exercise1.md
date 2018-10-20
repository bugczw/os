# 练习1：理解通过make生成执行文件的过程

*  操作系统镜像文件ucore.img是如何一步一步生成的？
*  一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？


命令为`make clean`和`make V=`，用于生成ucore.img磁盘镜像，其生成的输出见`out1.txt`。
其中`make clean`清除以前编译内容，`make V=`设置标记，展现Make的执行过程。



## 1. 编译生成内核代码

命令为`gcc -I... -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -I... -c kern/***.c -o obj/kern/***.o`。

该条命令主要调用的的gcc编译参数：

*  `-I...`设定了引用文件查找目录，即第一个寻找头文件的目录
*  `-fno-builtin`只接受以"__"开头的内建函数
*  `-nostdinc`使编译器不再系统缺省的头文件目录里面找头文件,一般和-I联合使用,明确限定头文件的位置 
*  `-Wall`  生成所有警告信息
*  `-ggdb`尽可能的生成gdb的可以使用的调试信息
*  `-gstabs`以stabs格式声称调试信息,但是不包括gdb调试信息.
*  `-m32`生成32位机器的汇编代码。相对的`-m64`则生成64位机器汇编代码
*  `-fno-stack-protector`不生成栈保护代码
*  `-c`对源代码进行预处理、编译、汇编，产生的是源代码的目标文件(*.o)
*  `-o`为将产生的可执行文件用指定的文件名

编译生成内核过程具体见输出文件1-48行,主要编译了

*  `kern/init/`目录中的`init.c`
*  `kern/libs/`目录中的`stdio.c`,`readline.c`
*  `kern/debug/`目录中的`panic.c`,`kdebug.c`,`kmonitor.c`
*  `kern/driver/`目录中的`clock.c`,`console.c`,`picirq.c`,`intr.c`
*  `kern/trap/`目录中的`trap.c`,`vectors.S`,`trapentry.S`
*  `kern/mm/`目录中的`pmm.c`
*  `libs/`目录中的`string.c`,`printfmt.c`



## 2. 链接生成内核映像 

命令为`ld -m elf_i386 -nostdlib -T tools/kernel.ld -o bin/kernel obj/***.o`。

该条命令：

*  `-m elf_i386`生成32位ELF映像
*  `-nostdlib`关闭了内建的库,仅搜索显示指出的搜索地址。链接脚本指出的目录（包括命令行指出的链接脚本）被忽略
*  `-T tools/kernel.ld`使用链接器脚本`tools/kernel.ld`，这个脚本描述了代码和数据在内存中的布局，以及设定了内核入口地址
*  `-o bin/kernel obj/***.o`把`obj`下的.o文件链接生成`bin/kernel`文件

链接生成内核映像过程具体见输出文件49-50行。



## 3. 编译生成bootloader代码

命令为`gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc -fno-stack-protector -Ilibs/ -Os -c boot/***.S -o obj/boot/***.o`。

参数命令见1中参数解释，需要注意的是
*  `-Os`表示对生成代码的大小进行优化，开启此选项的目的是满足启动扇区510字节代码的限制，若优化后仍超过大小，就需要手动编写汇编来优化了

编译生成bootloader代码过程具体见输出文件51-54行,主要编译了

*  `boot/`目录中的`bootasm.S`,`bootmain.c`



## 4. 链接生成bootloader映像

命令为`ld -m elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/***.o -o obj/bootblock.o`。

参数命令见1中参数解释，需要注意的是
*  `-N`设置代码段和数据段都可读可写，关闭动态链接
*  `-e start`指定入口点符号为`start`
*  `-Ttext 0x7C00`设置代码段起始地址为`0x7c00`
*  `obj/boot/***.o -o obj/bootblock.o`把`obj/boot/`下的.o文件链接生成`obj/bootblock.o`文件

链接生成bootloader映像过程具体见输出文件58-59行,将目标文件转化为一个执行程序（bootblock.out，即一个Bootloader，一个执行程序）



## 5. 生成bootloader二进制代码

命令为`objcopy -S -O binary obj/bootblock.o obj/bootblock.out`。

该条命令拷贝二进制代码bootblock.o到bootblock.out：

*  `-S`不拷贝重定位信息和调试信息
*  `-O binary`拷贝二进制代码
*  `obj/bootblock.o obj/bootblock.out`将ELF格式的`obj/bootblock.o`文件中的代码段拷贝到`obj/bootblock.out`



## 6. 生成启动扇区

为生成启动扇区，先编译生成`tools/sign.c`工具。该工具检查`obj/bootblock.out`文件的大小是否超过510字节，然后利用这个文件生成启动扇区`bin/bootblock`。启动扇区的特点见下。

生成启动扇区过程具体见输出结果55-57行.


 
## 7. 初始化磁盘镜像文件

命令为`dd if=/dev/zero of=bin/ucore.img count=10000`。

该条命令：

*  `dd`转换和拷贝文件
*  `if=/dev/zero`从全零的一个设备文件读取,`if`代表输入文件
*  `of=bin/ucore.img`写入到`bin/ucore.img`,`of`代表输出文件
*  `count=10000`共10000个扇区（共5120000字节）,`count`代表被复制的块数

初始化磁盘镜像文件过程具体见输出结果62行


## 8. 将启动扇区写入镜像文件

命令为`dd if=bin/bootblock of=bin/ucore.img conv=notrunc`。

该条命令：

*  `if=bin/bootblock`从6.生成的启动扇区文件读取
*  `of=bin/ucore.img`写入到`bin/ucore.img`
*  `conv=notrunc`不将`bin/ucore.img`文件清空

将启动扇区写入镜像文件过程具体见输出结果66行，生成Bootloader软件



## 9. 将内核映像写入镜像文件

命令为`dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc`。

该条命令：

*  `if=bin/kernel`从2.生成的内核映像文件读取
*  `of=bin/ucore.img`写入到`bin/ucore.img`
*  `seek=1`写入时，从第1扇区开始写入，即从输出文件开头跳过1个块后再开始复制
*  `conv=notrunc`不将`bin/ucore.img`文件清空

将内核映像写入镜像文件过程具体见输出结果70行，生成kernel软件




## 启动扇区的特点

详细阅读`sign.c`代码文件，发现能够被BIOS识别的启动扇区有如下特点：

*  符合扇区基本要求，大小512字节
*  最后两个字节，即第510和511个字节，分别为`0x55 0xAA`；若表示为一个16位的字，则为`0xAA55`（小端序）

若是有分区信息，中间还将有分区表。





## 小结

由Makefile文件中的ucore.img相关代码可以看出，UCOREIMG这个文件依赖于另外两个文件kernel和bootblock。

镜像文件为10000个块的文件，每个块默认512字节，默认用0填充， 把bootblock中的内容写到第一个块，然后从第二个块开始写kernel中的内容