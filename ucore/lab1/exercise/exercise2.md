# 练习2：使用qemu执行并调试lab1中的软件

*  从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
*  在初始化位置0x7c00设置实地址断点,测试断点正常。
*  从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
*  自己找一个bootloader或内核中的代码位置，设置断点并进行测试。


## 1 修改初始化文件
 
修改`lab1/tools/gdbinit`,内容为:

```
set architecture i8086
target remote :1234
```

其中`set architecture i8086`设置当前调试的CPU是8086，将指令集设置为16位，若不设置则GDB默认32位，导致反汇编失败。


## 2 调试模式运行

运行`make debug`开始实验。

此时，QEMU停在`0x0000fff0`，使用`x/i $cs`命令得到`0xf000: add %al,(%bx,%si)`,使用命令`x/i $eip`命令得到`=> 0xfff0: add %al,(%bx,%si)`,使用命令`x/i $pc`命令得到`=> 0xfff0: add %al,(%bx,%si)`。

由此得到是模式下对应的地址为`$cs<<4 | $eip = 0xffff0`

使用`x/i 0xffff0`命令，得到开机的执行的第一条指令为`0xffff0: ljmp $0xf000,$0xe05b`，即一个长跳转命令。

因此BIOS的开始地址为`$cs<<4 | 0xe05b = 0xfe05b`

使用`si`命令，果真跳转到`0xfe05b`开始执行了。


## 3 设置断点，进行调试

执行`b *0x7c00`在初始化位置`0x7c00`处设下断点，执行`c`使得QEMU继续运行，之后发现在`0x7c00`暂停了，断点有效。

此地址是bootloader入口点地址，可看boot/bootasm.S的start地址处。执行`x/10i 0x7c00`查看反汇编，输出结果为
```
=> 0x7c00:      cli
   0x7c01:      cld
   0x7c02:      xor    %ax,%ax
   0x7c04:      mov    %ax,%ds
   0x7c06:      mov    %ax,%es
   0x7c08:      mov    %ax,%ss
   0x7c0a:      in     $0x64,%al
   0x7c0c:      test   $0x2,%al
   0x7c0e:      jne    0x7c0a
   0x7c10:      mov    $0xd1,%al
```

仔细观察，反汇编结果与`\boot\bootasm.S`中16-30行一致，与`\obj\bootblock.asm`中12-34行有些许区别。这是由于反汇编生成`bootblock.asm`时，使用的是32位指令集。

值得一提的是，使用`si`单步执行，可以同时使用`layout`切换格局，可以同时查看汇编代码以及寄存器的值。 