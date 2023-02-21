# 这个lab分为A,B两个部分,A主要围绕Env结构展开,其实这是进程的雏形,B主要是对中断的一些实验

lab2完成了关于内存管理的一些函数,到现在还一直都是写的内核的代码,还无法运行一个用户程序,这个lab就是完成一些为用户程序服务的工作.整个lab2其实就是完成了mem_init这个函数及其调用的函数.

一个用户程序要运行起来最基本的条件就是**内存+代码**,jos将一个用户程序所要涉及的资源定义成一个Env,其中的env_pgdir就是该用户进程要使用的页目录表的位置.Trapframe是用户进程要进行中断时用到的结构,可以暂时不管.

## E1:完善mem_init,使其能通过check_kern_pgdir函数

与之前的pages一样,现在需要申请一块内存来存放所有的Env结构,回去memlayout.h里看那个内存的布局图就能发现这块区域是在用户区的,因为用户程序可能会访问进程的状态等信息,但用户不能修改该区域的信息,所以页表项的属性应为用户只读

```c

envs = (struct Env*)boot_alloc(NENV * sizeof(struct Env));
boot_map_region(kern_pgdir,UENVS,PTSIZE,PADDR(envs),PTE_U);
```

## E2:完成env相关函数,装载用户程序

正常来讲用户程序一般在硬件上存储,但现在的Jos还没有文件系统,所以现在是把用户程序的代码嵌到内核里面的.嵌的方式如下:

1. GNUmakefile会生成许多二进制文件到obj/user/下
2. 在kern/Makefrag里链接这些二进制文件时加入了-b binary参数,这样链接器就把这些二进制文件当作普通的纯二进制文件进行链接(一般的.o文件是有elf头的),并且自动为这些二进制文件生成相应的符号以供代码里使用
3. 代码里面通过声明这些符号为一个地址进行访问

总结起来就是相当于用户程序的代码放在了内核里面,内核可以通过链接器产生的符号去访问这块地方,因为正常情况下应该是通过文件系统去得到用户程序的映像的,这里是直接把映像给放到内核里面了

lab2完成了mem_init函数并调用成功,现在就是回到i386_init这个函数,lab3的这个函数已经发生了改变,下面就是要去完成调用到的函数

### env_init

> 对申请到的envs这块空间进行初始化

跟page_init很像,将对应属性赋值然后初始化好env_free_list

```c

void
env_init(void)
{
 // Set up envs array
 // LAB 3: Your code here.
 size_t i;
 for(i=0;i<NENV;++i){
  envs[i].env_id= 0;
  envs[i].env_link = envs+i+1;
  envs[i].env_status = ENV_FREE;
 }
 envs[NENV-1].env_link = NULL;
 env_free_list = envs;
 // Per-CPU part of the initialization
 env_init_percpu();
}
```

env_init_percpu重新装载了gdt,然后对对段寄存器进行了初始化

### env_setup_vm

> 为Env分配一个页目录表并初始化好kernel部分的表项

这个函数就是为一个新的Env申请一个pgdir,每个进程的pgdir的UTOP之上的内容应该都是一样的(除了UVPT部分,这里放的是当前Env的页表),因为这一块指向了内核和一些只读数据,所以每个用户程序的pgdir的UTOP之上都与内核的一致

```c
static int
env_setup_vm(struct Env *e)
{
   //为一个用户进程申请"内存空间"然后初始化一些基本的内容
 int i;
 struct PageInfo *p = NULL;
 // LAB 3: Your code here.
 if(!(p=page_alloc(ALLOC_ZERO)))
    return -E_NO_MEM;
 ++p->pp_ref;
 e->env_pgdir = (pde_t*)page2kva(p);
 //Map directory below UTOP
 for(i=0;i<PDX(UTOP);++i){
  e->env_pgdir[i] = 0;
 }
 //copy the same entrys of kernel
 for(i=PDX(UTOP);i<NPDENTRIES;++i){
  e->env_pgdir[i] = kern_pgdir[i];
 }
 
 // UVPT maps the env's own page table read-only.
 // Permissions: kernel R, user R
 e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

 return 0;
}
```

直接将kern_pgdir[PDX(UTOP):]的内容拷了过来,不用担心用户程序修改内核内容,因为这些表项都被设置成了用户只读.另外每个env_pgdir的UVPT处写入了自己的页表,所以这里不一样了.

### region_alloc

> 为进程e申请len字节的空间并将将其映射到虚拟地址va

有了pmap.c里的page_insert函数这个函数就容易实现了,只是要注意一下页对齐,因为目前的空间的申请是以页为单位的

```c

static void
region_alloc(struct Env *e, void *va, size_t len)
{
 // LAB 3: Your code here.
 // (But only if you need it for load_icode.)
 
 uintptr_t start = (uintptr_t)ROUNDDOWN((uint32_t)va,PGSIZE);
 uintptr_t end = (uintptr_t)ROUNDUP((uint32_t)va+len,PGSIZE);
 struct PageInfo *p = NULL;
 uintptr_t i;
 for(i=start;i<end;i+=PGSIZE){
  if((p = page_alloc(ALLOC_ZERO)) == 0)
    panic("region_alloc:page_alloc failed!");
  if(page_insert(e->env_pgdir,p,(void*)i,PTE_U|PTE_W)!=0){
  page_free(p);
  panic("region alloc:page insert failed!");
  }
 }
}
```

不过这个函数只会在load_icode时用到,以后会用其它方式实现用户对空间的申请

### load_icode

>解析二进制文件,将其内容加载到一个用户进程内

之前说过链接器为每个二进制文件生成了一个符号,现在就用到了,把二进制文件的地址传给这个函数它就会去读取那里的内容,解析,然后装到用户进程内,之前已经为用户进程装载了"内存空间",现在就是要赋予进程"代码"

由于二进制文件是之前编译好的,知道是ELF格式,所以函数里就是按ELF格式解析的

```c

static void
load_icode(struct Env *e, uint8_t *binary)
{
   
 // LAB 3: Your code here.
 struct Elf *elf = (struct Elf*)binary;
 if(elf->e_magic != ELF_MAGIC)
  panic("load_icode:not elf format!");
 if(elf->e_entry == 0)
  panic("the binary can't be excuted!");
  //查看后面env_run和env_pop_tf的代码,env_tf.tf_eip将是
  //一个新进程执行的第一条指令的地址,所以要把程序的入口地址放在这里
  //另外去查看user.ld可以看到将用户程序的链接到了0x800020这里,所以用readelf可以看到entry就是0x800020
 e->env_tf.tf_eip = elf->e_entry;
 lcr3(PADDR(e->env_pgdir));//那binary还能找到吗？
 //下面就是解析elf头文件,将对应的段加载到对应的位置
 struct Proghdr *ph,*eph;
 ph = (struct Proghdr*)((uint8_t*)elf+elf->e_phoff);
 eph = ph+elf->e_phnum;
 for(;ph<eph;++ph){
  if(ph->p_type == ELF_PROG_LOAD){
   //memsz应该>=filesz
   if(ph->p_memsz < ph->p_filesz)
    panic("load_icode:p_memsz < p_filesz");
   //先申请空间
   region_alloc(e,(void*)ph->p_va,ph->p_memsz);
   //然后拷贝内容
   memmove((void*)ph->p_va,binary+ph->p_offset,ph->p_filesz);
   //将多余的这段空间内容置0
   memset((void*)(ph->p_va+ph->p_filesz),0,ph->p_memsz-ph->p_filesz);
  }
 }
 // Now map one page for the program's initial stack
 // at virtual address USTACKTOP - PGSIZE.

 // LAB 3: Your code here.
 //为进程申请栈的空间,因为各个进程的地址空间是独立的,所以相同的虚拟地址会有不同的映射,不用担心进程之间互相干扰
 region_alloc(e,(void*)(USTACKTOP-PGSIZE),PGSIZE);
}
```

lcr3那里将用户的pgdir的物理地址装载到了cr3,就进入到了用户的地址空间,这时候就有一个问题--binary所指的地址还能不能找到内容？

答案是肯定的,前面说过所有的用户代码链接进了内核,所以所有的代码都在KERNBASE之上,可以去查看obj/kern/kernel.sym,也确实是这样的,而每个进程在初始化的时候会拷贝kern_pgdir的内容,KERNEBASE之上的地址映射是全部拷贝了,所以在内核和用户的地址空间中KERNBASE之上的内容是一样的,而KERNBASE之上的表项被设置为了用户只读,所以不用担心用户程序修改内核的内容

### env_create

>为给定的二进程程序创造一个运行的环境Env

有了前面load_icode这个函数的实现就非常简单,先调用env_alloc申请一个空闲的Env然后用load_icode将程序装载进去即可

```c

void
env_create(uint8_t *binary, enum EnvType type)
{
 // LAB 3: Your code here.
 int r;
 struct Env *e;
 if((r=env_alloc(&e,0))!=0)
  panic("env_create:%e",r);
 load_icode(e,binary);
 e->env_type = type;

}
```

### env_run

>在用户级别运行一个给定的Env

一个进程需要的条件都已经在Env中定义好了--地址空间+线程.现在就是在怎么利用Env的数据来运行它.

1. 如果是一个用户进程调用这个函数,那就需要切换一下curenv的env_status为ENV_RUNNABLE,这样做是为了让其后面能被唤醒

2. 让curenv指向给定的Env并将其状态切换为ENV_RUNNING,增加一次env_runs的值

3. 用lcr3将Env的页目录表的物理地址加载到CR3,切换到目标Env的地址空间

4. 调用env_pop_tf装载Env存储的寄存器的内容并去执行Env的代码

```c

void
env_run(struct Env *e)
{
  if(curenv!=NULL){
  curenv->env_status = ENV_RUNNABLE;
 }
 curenv = e;
 curenv->env_status = ENV_RUNNING;
 ++curenv->env_runs;
 lcr3(PADDR(curenv->env_pgdir));
 env_pop_tf(&curenv->env_tf);

 panic("env_run not yet implemented");
}
```

env_pop_tf的内容就是装载寄存器然后使用iret命令进入用户模式并执行代码,这跟下面的中断内容非常相关

## E3:阅读手册相关中断的内容

interrupt与exception都表示了一种情形:在系统的某个地方产生了一个信号需要处理器注意,往往处理器对待这种情形就是停下当前的工作,马上去处理这个信号.
这里就会发生特权级的转换:从用户模式转换到内核模式.系统就需要设计一种模式去限制用户程序产生中断的条件以及对应的中断的行为,否则用户程序可以随意产生中断切换自己的特权级从而去调用内核的操作来掌控所有权力.

这种机制需要一些硬件的支持,x86主要通过提供以下两个硬件服务来协助系统的中断设计:

1. 中断描述符表(Interrupt Descriptor Table):CPU可以保证所有的中断只能由内核设计的IDT进入内核模式.
x86允许最多定义256个中断入口,每个入口由一个0~255的数字代表,称这个数字为vector.其中一些已经被x86提前定义:
![被提起定义的vector](https://s2.loli.net/2023/02/13/32SHKlIXinr6bG1.png)

当对应的中断产生时CPU会以这个vector为索引去内核存放IDT的地方找到一个中断描述符,该描述符包含了处理该中断的地址

![IDT的寻址方式](https://s2.loli.net/2023/02/13/X8ovCDdHfeBsqai.png)
2. Task State Segment:CPU在进入中断处理程序之前需要保存一些数据如当前的CS与EIP,这样等中断返回时CPU才能回到此刻的状态继续运行,而这个地方需要远离用户程序否则内核返回时可能会被恶意程序欺骗.出于这个原因当中断发生且是特权级转变的中断发生时CPU会同时切换到一个内核用的栈,这个栈由TSS指出,之后CPU会在这里压入SS,ESP,CS,EIP...等内容

## E4:编辑trapentry.S与trap.c,使其能处理前32个中断

![trapentry.S与trap.c的关系](https://s2.loli.net/2023/02/14/h4MKUgfIyPtkH9L.png)
从上面这张图可以看出来trapentry.S就是在真正开始处理之前做一些准备然后调用trap.c里的trap函数进行处理.有些准备是只能在这里完成的比如存储当前的中断号,因为后面的处理程序是没法得知当前发生的中断号的,只有在入口这里提前压入以供后面使用.

```as

/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */

TRAPHANDLER_NOEC(t_divide, T_DIVIDE)
TRAPHANDLER_NOEC(t_debug, T_DEBUG)
TRAPHANDLER_NOEC(t_nmi, T_NMI)
TRAPHANDLER_NOEC(t_brkpt, T_BRKPT)
TRAPHANDLER_NOEC(t_oflow, T_OFLOW)
TRAPHANDLER_NOEC(t_bound, T_BOUND)
TRAPHANDLER_NOEC(t_illop, T_ILLOP)
TRAPHANDLER_NOEC(t_device, T_DEVICE)
TRAPHANDLER(t_dblflt, T_DBLFLT)
TRAPHANDLER(t_tss, T_TSS)
TRAPHANDLER(t_segnp, T_SEGNP)
TRAPHANDLER(t_stack, T_STACK)
TRAPHANDLER(t_gpflt, T_GPFLT)
TRAPHANDLER(t_pgflt, T_PGFLT)
TRAPHANDLER_NOEC(t_fperr, T_FPERR)
TRAPHANDLER(t_align, T_ALIGN)
TRAPHANDLER_NOEC(t_mchk, T_MCHK)
TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)
/*
 * Lab 3: Your code here for _alltraps
 */
 _alltraps:
        pushl %ds
        pushl %es
        pushal

        movl $GD_KD,%eax
        movw %ax,%ds
        movw %ax,%es

        push %esp
        call trap
```

要注意的是当CPU进行到这里时说明有中断发生,CPU已经自动压入了一些内容,并从IDT里找到了当前入口的地址.

两个宏的内容基本上就是声明一个函数符号,然后压入给定的num之后跳转到_alltraps,两者的区别就是一个压入了0,因为有些中断会产生err_code有些不会,便于后面的程序统一处理,在不会产生err_code的入口手动压入一个0来保持结构的一致性.

声明符号是为了能在trap.c的trapinit里面能引用该符号,因为IDT里需要存储该地址

然后根据要求完成_alltraps的内容即可,`push %esp`的作用是将当前的esp的值当作参数传给trap函数,因为一路压下来esp就在一个TrapFram的起始地址

### trap_init

> 初始化idt

```c
void t_divide();
void t_debug();
void t_nmi();
void t_brkpt();
void t_oflow();
void t_bound();
void t_illop();
void t_device();
void t_dblflt();
void t_tss();
void t_segnp();
void t_stack();
void t_gpflt();
void t_pgflt();
void t_fperr();
void t_align();
void t_mchk();
void t_simderr();
void t_syscall();
void
trap_init(void)
{
 extern struct Segdesc gdt[];

 // LAB 3: Your code here.
 
 SETGATE(idt[T_DIVIDE], 0, GD_KT, t_divide, 0);
 SETGATE(idt[T_DEBUG], 0, GD_KT, t_debug, 0);
 SETGATE(idt[T_NMI], 0, GD_KT, t_nmi, 0);
 SETGATE(idt[T_BRKPT], 0, GD_KT, t_brkpt, 3);
 SETGATE(idt[T_OFLOW], 0, GD_KT, t_oflow, 0);
 SETGATE(idt[T_BOUND], 0, GD_KT, t_bound, 0);
 SETGATE(idt[T_ILLOP], 0, GD_KT, t_illop, 0);
 SETGATE(idt[T_DEVICE], 0, GD_KT, t_device, 0);
 SETGATE(idt[T_DBLFLT], 0, GD_KT, t_dblflt, 0);
 SETGATE(idt[T_TSS], 0, GD_KT, t_tss, 0);
 SETGATE(idt[T_SEGNP], 0, GD_KT, t_segnp, 0);
 SETGATE(idt[T_STACK], 0, GD_KT, t_stack, 0);
 SETGATE(idt[T_GPFLT], 0, GD_KT, t_gpflt, 0);
 SETGATE(idt[T_PGFLT], 0, GD_KT, t_pgflt, 0);
 SETGATE(idt[T_FPERR], 0, GD_KT, t_fperr, 0);
 SETGATE(idt[T_ALIGN], 0, GD_KT, t_align, 0);
 SETGATE(idt[T_MCHK], 0, GD_KT, t_mchk, 0);
 SETGATE(idt[T_SIMDERR], 0, GD_KT, t_simderr, 0);
 SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);
 

 // Per-CPU setup 
 trap_init_percpu();
}
```

trap_init的工作很简单,就是使用SETGATE宏去设置idt里的内容,即构造中断描述符,最后的trap_init_percpu函数就是装载idt与tss.

## Challeng1:简化trapentry.S与trap_init

可以在trapentry.S里定义一张表,在trap_init里直接通过循环来初始化idt,trap_init需要trapentry.S的内容其实就是对应中断程序的入口地址,所以只要在trapentry.S里构造一张存储地址的表供trap_init调用即可
参考<https://github.com/plutoshe/JOS_OperatingSystemlab/tree/master/lab3>

```asm

#define MYHANDLER(name, num)      \
 .text;    \
 .globl name;  /* define global symbol for 'name' */ \
 .type name, @function; /* symbol type is function */  \
 .align 2;  /* align function definition */  \
 name:   /* function starts here */  \
 pushl $(num);       \
 jmp _alltraps; \
 .data; \
 .long name 

#define MYHANDLER_NOEC(name, num)     \
 .text; \
 .globl name;       \
 .type name, @function;      \
 .align 2;       \
 name:        \
 pushl $0;       \
 pushl $(num);       \
 jmp _alltraps;       \
 .data;      \
 .long name

#define MYHANDLER_NULL() \
 .data; \
 .long 0

/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
.data
.align 2
.global vectors

vectors:
.text
MYHANDLER_NOEC(trap_handler0, 0)
MYHANDLER_NOEC(trap_handler1, 1)
MYHANDLER_NOEC(trap_handler2, 2)
MYHANDLER_NOEC(trap_handler3, 3)
MYHANDLER_NOEC(trap_handler4, 4)
MYHANDLER_NULL()
MYHANDLER_NOEC(trap_handler6, 6)
MYHANDLER_NOEC(trap_handler7, 7)
MYHANDLER_NOEC(trap_handler8, 8)
MYHANDLER_NULL()
MYHANDLER(trap_handler10, 10)
MYHANDLER(trap_handler11, 11)
MYHANDLER(trap_handler12, 12)
MYHANDLER(trap_handler13, 13)
MYHANDLER(trap_handler14, 14)
MYHANDLER_NULL()
MYHANDLER_NOEC(trap_handler16, 16)
MYHANDLER(trap_handler17, 17)
MYHANDLER_NOEC(trap_handler18, 18)
MYHANDLER_NOEC(trap_handler19, 19)
```

自定义的宏就多了一个.data段存储入口地址,同时因为中间有些是x86的保留vector没有用到,但要构造的表是连续的,所以需要在对应位置生成一个.data段,存储内容为0

由于链接器脚本会将相同的段放在一起,所以这个文件里的所有.data段合在一起,这样就连续了,此时vectors就相当于一个存储了地址的数组

```c
//trap_init的内容可以做如下修改,当然中间一些特殊的设置还是需要单独处理
extern uint32_t vectors[];
 int i;
 for (i = 0; i < 20; i++) {
  SETGATE(idt[i], 0, GD_KT, vectors[i], 0);
 }
```

## Question1

1. 不用一个handler处理所有中断的原因是无法区别需要处理的中断,可以看到在所有的中断最后都会去调用trap函数,但在此之前需要将自己的中断号push一下,这样trap才能区分各个中断,如果所有中断一来就直接调用trap那trap是无法区分的.
2. 这里不需要做什么,因为在trap_init那里将这个描述符的DPL设置为了0,用户特权级为3,想直接访问特权级为0的内容就会触发General Protection Fault中断,编号为13.可以看到breakpoint中断的特权级为3,所以修改softint.c的代码去触发3号中断的话就不会产生GP异常.

## E5:修改trap_dispatch使其能处理page fault中断

经过E4之后会把所有的中断引向trap函数,它的主要工作还是对真正开始处理中断前进行一些准备工作,如`cld`以及判断当前环境是否允许进行中断.如果当前中断是由用户进程产生那可能发生了栈的切换,那curenv的env_tf需要拷贝一份过来,为中断恢复做准备,然后进入trap_dispatch解析不同trap

trap_dispatch的工作就很简单,根据trapno调用相应的中断处理程序,而page_fault的处理已经实现了,所以直接调用就行:

```c
static void
trap_dispatch(struct Trapframe *tf)
{
 // Handle processor exceptions.
 // LAB 3: Your code here.
 switch(tf->tf_trapno){
  case T_PGFLT:
   page_fault_handler(tf);
   return;
 }
 // Unexpected trap: The user process or the kernel has a bug.
 print_trapframe(tf);
 if (tf->tf_cs == GD_KT)
  panic("unhandled trap in kernel");
 else {
  env_destroy(curenv);
  return;
 }
}
```

## E6: 编写trap_dispatch使其能通过breakpoint中断进入monitor程序

其实就是将breakpoint的中断行为设为monitor调用即可

```c
static void
trap_dispatch(struct Trapframe *tf)
{
 // Handle processor exceptions.
 // LAB 3: Your code here.
 switch(tf->tf_trapno){
  case T_PGFLT:
   page_fault_handler(tf);
   return;
  case T_BRKPT:
   monitor(tf);
   return;
 }
 // Unexpected trap: The user process or the kernel has a bug.
 print_trapframe(tf);
 if (tf->tf_cs == GD_KT)
  panic("unhandled trap in kernel");
 else {
  env_destroy(curenv);
  return;
 }
}
```

## Challenge2:修改代码使其monitor能支持单步调试

查阅博客:
当产生int3中断后调用的处理函数是monitor,如果只是简单的执行到下一个断点那只要关闭EFLAGS的TF位然后在monitor里面调用env_run(curenv)即可,tf位为1则允许单步调试,每执行一条指令就会产生一个DEBUG中断

```c
int mon_continue(int argc,char **argv,struct Trapframe *tf)
{
 if(argc > 1){
  cprintf("invalid number of argcs\n");
  return 0;
 }
 if(tf == NULL){
  cprintf("continue error\n");
  return 0;
 }
 tf->tf_eflags &= ~FL_TF;
 env_run(curenv);
 panic("continu error\n");
 return 0;
}
```

使用`make run-breakpoint`测试breakpoint如下:
![breakpoint测试1](https://s2.loli.net/2023/02/20/4kwpY8aQmP3bgAD.png)
![breakpoint测试2](https://s2.loli.net/2023/02/20/ST5FA6W7QXMz3Jv.png)
可以看到两次eip的值是不一样的,可以去breakpoint.asm里查看这两个地址处的命令具体是什么

而单步调试就是要将该位置1

```c
int mon_si(int argc,char **argv,struct Trapframe *tf)
{
 if(argc > 1){
  cprintf("invalid number of argcs\n");
  return 0;
 }
 if(tf == NULL){
  cprintf("step error\n");
  return 0;
 }
 tf->tf_eflags |= FL_TF;
 env_run(curenv);
 panic("step error\n");
 return 0;
}
```

开启tf位后每执行一条指令将会产生一个Debug中断,所以需要去trap_dispatch处理一下:

```c
case T_DEBUG:
   monitor(tf);
   break;
```

仍然运行breakpoint程序:

![si结果](https://s2.loli.net/2023/02/20/yBmu9De3fwRAKOL.png)

可以发现触发了debug中断,eip的值没有一下跳到下一个breakpoint,而是下一条指令的位置

## Question2

1. breakpoint中断在初始化IDT的时候需要将其设置DPL设置为3才能如期运行,因为断点就是用户程序触发的,如果将其DPL设为0将会引起GP异常无法达到断点效果
2. 上面的代码实现的机制就是设计好一套入口,让对应中断正确被处理.这里的入口限制了用户程序不能随意进入,只能从系统提前规划好的地方进入.

## E7:添加用户系统调用

这里为用户实现系统调用的方式是定义一个IDT入口,其vector为0x30,显然其DPL应该为3.然后就是参数传递的问题,普通的函数调用是通过压栈来传递参数的,这里是通过寄存器来传递参数,最多支持5个参数,按顺序传入edx,ecx,ebx,edi,esi五个寄存器,然后将要调用的系统调用号放到eax,调用的返回结果也是规定放到eax.

首先在trapentry.S里添加其入口:

```asm
TRAPHANDLER_NOEC(t_syscall, T_SYSCALL)
```

然后在trap_init里构造门:

```c

SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);
```

接着去trap_dispatch里调用实际的处理函数syscall(这个syscall是kern/syscall.c里的)

```c
case T_SYSCALL:
   tf->tf_regs.reg_eax = syscall(tf->tf_regs.reg_eax,tf->tf_regs.reg_edx,tf->tf_regs.reg_ecx,tf->tf_regs.reg_ebx,tf->tf_regs.reg_edi,tf->tf_regs.reg_esi);
   return;
```

然后去kern/syscall.c里完整一下syscall.c的内容:

```c
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
 // Call the function corresponding to the 'syscallno' parameter.
 // Return any appropriate return value.
 // LAB 3: Your code here.

 //panic("syscall not implemented");

 switch (syscallno) {
  case SYS_cputs:
   sys_cputs((char*)a1,a2);
   break;
  case SYS_cgetc:
   return sys_cgetc();
  case SYS_env_destroy:
   return sys_env_destroy(a1);
  case SYS_getenvid:
   return sys_getenvid();
 default:
  return -E_INVAL;
 }
 return 0;
}
```

这里说一下lib/syscall.c和kern/syscall.c的区别.
用户代码肯定不能直接调用内核对应的syscall,比如sys_puts来打印，必须得通过中断，而且还要将对应的参数传入对应的寄存器.
但对用户来说一个系统调用应该像调用了一个普通函数一样，不应该去了解怎么把参数传到对应的寄存器，该调用哪个syscall以及应该从哪个vector进入中断。所以lib/syscall.c就封装了一层"接口",让用户直接像普通函数调用一样调用对应syscall,所以lib/syscall.c里的syscall是static inline的,它只是填好了对应的参数，真正回应调用的是在`int 0x30`之后的T_SYSCALL中断，然后该中断又会根据lib/syscall.c里传入eax的编号调用真正的内核syscall。

梳理一下就是lib里与kern里所有同名的syscall都是一层封装，目的是为了能让用户程序轻松地调用对应的系统调用。

## chanllenge3:用sysenter和sysexit指令替换int 30与iret

这两个指令是为了能更快地完成系统调用，可以查看一下相关资料，但是从描述来看与已经实现的方式出入较大，需要改动的地方较多且说可能不支持后面的一些lab所以就不去实现了。

## E8:添加代码使user/hello.c能正确运行

从user/Makefrag和user.ld可以看出来每个编译好的程序前面还加入了由lib/entry.S编译成的entry.o,这个里面的_start才是用户程序真正的入口，可以去obj/user对应的sym文件看下,0x800020这里都是_start.而entry.S里做的工作就是设置了一些数据供用户程序使用，其中就包括存储env的数组envs.
然后会调用libmain函数,在这里对thisenv进行初始化,通过ENVX(sys_getenvid())可以获得当前进程在envs的位置,加上envs就是当前进程的地址:

```c
thisenv = envs + ENVX(sys_getenvid());
```

之后libmain再去调用用户程序里的umain函数真正进入用户程序,由于之前没有对thisenv进行初始化，为NULL,相当于用户进程访问地址0,引发page fault

## E9:增加一些安全检查

在load_icode的时候为程序分配的栈空间是一个页的大小，但是用户程序可能会使用超过这个栈的大小，从而引发page fault。一般操作系统会精心处理这种情况的page fault使得用户好像在一片随意大小的栈上工作

很多系统调用会让用户传一个指针过来，这些指针指向用户内存区域的某些可读写内容，然后内核通过这个指针去执行某些系统调用，这样就会有下面两个问题出现:

1. 一个page fault可能是内核引起的(这将导致整个系统崩溃),也可能是用户程序引起的(比如传了一个指针给内核，但指向的位置会引发page fault),这种情况内核是可以选择处理的，这个时候需要一种方法来区别page fault是谁引起的

2. 内核通常拥有更高的权限去读写内存，如果一个用户程序恶意传入一个指针去读写字节没有权限读写的区域，没有检查的情况下就会很危险。

第一问题可以通过检查tf_cs的低三位来区别是内核还是用户程序引发的page fault,如果是kernel引起就直接panic系统

```c
void
page_fault_handler(struct Trapframe *tf)
{
 uint32_t fault_va;

 // Read processor's CR2 register to find the faulting address
 fault_va = rcr2();

 // Handle kernel-mode page faults.

 // LAB 3: Your code here.
 if((tf->tf_cs&3)!=3)
  panic("kernel page fault\n");

 // We've already handled kernel-mode exceptions, so if we get here,
 // the page fault happened in user mode.

 // Destroy the environment that caused the fault.
 cprintf("[%08x] user fault va %08x ip %08x\n",
  curenv->env_id, fault_va, tf->tf_eip);
 print_trapframe(tf);
 env_destroy(curenv);
}
```

第二个问题通过去查env的页表可以解决:

```c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
 // LAB 3: Your code here.
 uintptr_t start =(uintptr_t)ROUNDDOWN(va,PGSIZE);
 uintptr_t end = (uintptr_t)ROUNDDOWN(start+len,PGSIZE);
 pte_t *entry = NULL;
 for(;start<=end;start+=PGSIZE){
  entry = pgdir_walk(env->env_pgdir,(void*)start,0);
  if(start>ULIM||entry==NULL||((uint32_t)(*entry)&perm)!=perm){
   if(start<(uintptr_t)va)
    user_mem_check_addr = (uintptr_t)va;
   else
    user_mem_check_addr = start;
   return -E_FAULT;
  }
 }
 return 0;
}
```

然后在相应的地方进行检查即可
