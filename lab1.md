### e5:
这个练习是关于链接的,引出了两个术语VMA(链接地址,程序希望加载的逻辑地址)LMA(加载地址,被加载到的实际物理地址).链接地址用于给链接器计算用生成一些跳转的的地址,比如objdump查看boot.out如下:
![](https://s2.loli.net/2023/01/18/H9rK1oBDsPAuaQf.png)
由于在启动阶段分页和分段机制都还没有开启，所以链接地址和加载地址是一样的,链接地址可以在链接时通过参数指定，从编译文件中可以看到，boot的链接地址是在boot/MakeFrag里面指定的，如果修改里面的0x7c00为0x7d00,make clean之后重新make,可以继续用objdump查看，也可以直接从obj/boot/boot.asm里面看到_start的地址变成了0x7d00.进入gdb模式前面部分命令仍让可以正常执行，但是直接continue系统是无法加载出来的。
**原因**
现在的情况是程序被加载到了0x7c00处(这是BIOS的行为，无法控制),但程序里的地址是以0x7d00开始的,由于一些指令不需要地址，比如cli以及mov ds,ax这些，所以程序只需要一条一条往下执行即可.正常情况下si到0x7c1e这里，要说一下的是这里的0x7c1e是真实的地址，指令是lgdt ds:0x7c64(其实这条指令我是用bochs调试才这样的，用gdb调试会显示lgdtl (%esi),但是(%esi)那里的内容并不是这样，可能是我理解不够的原因,一直得不到正确的显示),lgdt是把后面地址处的6个字节装到gdtr寄存器的指令，用`x /6xb 0x7c64`查看内容,结果如下:
`0x17    0x00    0x4c   0x7c  0x00 x00`
0x0017(8086是小端机)是在boot.S里的gdtdesc处定义好的,后面四个字节就是链接器在链接时计算生成的，即计算处gdt的位置在0x7c4c处,查看boot.asm也确实如此
但是如果将boot/MakeFrag里的0x7c00修改成0x7d00之后这里就会变成lgdt ds:0x7d64,同样的方式查看0x7cd4处的内容如下:
`0xc3   0x20   0xe8   0x76  0xff  0xff`
前面两个字节并不是0x0017,显然错误。原因就是链接器只能用链接地址来计算一些地址逻辑上的地址数据，无法控制程序加载的实际地址,如果没有链接地址,gdt应该处于离开头的0x64处,链接器只是替换了一些数据，并不能改变程序的布局,所以在BIOS加载之后,gdt仍然处于离头0x64的地方,而头现在在0x7c00,故gdt就是0x7c64,只要boot.S的结构不变,无论链接时指定什么地址，gdt最后真正的位置都在0x7c64处,而如果乱指定链接地址，链接器会在代码里填入不正确的地址导致程序崩溃

### e9:关于ebp与esp

简单但不那么准确的理解，一个函数在运行时有自己的一段“连续”空间,函数执行时中间要用到的临时变量都是在这里进行的,这段连续空间的开头由ebp限制,访问时配合ebp访问,下面是简单的一个例子:

```c
//main.c
int main()
{
    int a = 100;
    int b = 200;
    return 0;
}
```

gcc main.c -m32 -S产生main.s主要内容如下:

```asm
main:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $16, %esp
        movl    $100, -4(%ebp)
        movl    $200, -8(%ebp)
        movl    $0, %eax
        leave
        ret
```

可以看到先将esp减小了16，虽然两个int只要8字节就够了，但这里减16肯定是腾够空间了，腾出空间后就是配合ebp去访问和写入,这段函数自己的空间叫“栈帧”，每个函数运行时都有自己的栈帧。

#### 为什么函数的开头总是`pushl %ebp;movl %esp,%ebp`?

继续看代码:

```c
//main.c
int f()
{
        return 1;
}

int main()
{
        int a = 100;
        int b = 200;
        f();
        return 0;
}
```

产生的汇编文件如下:

```asm
f:
        pushl   %ebp
        movl    %esp, %ebp
        movl    $1, %eax
        popl    %ebp
        ret
        .size   f, .-f
.globl main
        .type   main, @function
main:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $16, %esp
        movl    $100, -4(%ebp)
        movl    $200, -8(%ebp)
        call    f
        movl    $0, %eax
        leave
        ret
```

可以看到函数开始的两条命令是一样的
上面说到每个函数都有自己的栈帧，而栈帧是由ebp表达的，所以如果某个函数开始运行了一定会产生相应的ebp的值，如果函数体当中又有函数调用，为了以后能回到现在的栈帧肯定需要保存当前ebp的值,这就是第一个命令的作用，刚刚进入函数时ebp的值是属于main的(call指令只会压入一个返回地址，即只会影响esp不影响ebp),此时`pushl %ebp`就是保存其值为了后面返回main时能切回main的栈帧,而`movl %esp %ebp`就是为当前函数产生了一个栈帧，由于现在ebp与esp的值一样，所以目前还没有空间可用,而在f里面并没有用到临时变量,所以没有出现类似`subl $16 %esp`的操作,所以popl %ebp就相当于把刚压进去的ebp又写回来了，ebp回复成main的ebp,之后ret指令回到main,栈帧也就切换成了main的栈帧,如果f中有使用临时变量

```c
int f()
{
        int c = 10;
        return 1;

}

int main()
{
        int a = 100;
        int b = 200;
        f();
        return 0;
}
```

产生的汇编内容如下:

```asm
f:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $16, %esp
        movl    $10, -4(%ebp)
        movl    $1, %eax
        leave
        ret
        .size   f, .-f
.globl main
        .type   main, @function
main:
        pushl   %ebp
        movl    %esp, %ebp
        subl    $16, %esp
        movl    $100, -4(%ebp)
        movl    $200, -8(%ebp)
        call    f
        movl    $0, %eax
        leave
        ret

```

出现了跟main一样的leave指令,leave指令相当于执行两条指令`movl %ebp %esp;popl %ebp`因为对esp进行了sub操作，直接pop出来的不是之前压入的ebp,所以需要先将esp的值改为ebp的值(因为这恰好是esp减小之前的值)再pop才能得到最开始push进去的ebp

e10:调试test_backtrace过程对C编译的字符串产生疑问