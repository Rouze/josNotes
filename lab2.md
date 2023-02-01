# 这一个lab主要是关于内存的，更具体地说是关于内存布局的

因为我是跟《操作系统真象还原》这本书走过一遍的，所以具体的一些关于物理分页的知识就不再细说，简略概括一下:

1. 在这个lab中我们要操作的内存最小单位是"页",一个页是4KB的大小
2. 开启分页后所有地址都会经过一个MMU硬件处理，映射到实际的物理地址
3. 后面说到的页表根据语境

boot程序最后通过entry进入到内核程序，链接器把内核代码链接到0xf0100000,而内核实际是被加载到0x100000处，所以要想正常运行内核代码这里就已经需要进行映射了.kern/entrypgdir.c里就手动构建了一个PDE页表和PTE页表临时使用,通过objdump -t可以看到entry_pgdir在0xf0115000,entry_pgtable在0xf016000(真正地址要减去KERNELBASE)一张PTE表可以映射4MB的空间，对于当前阶段的内核已经够用了

## E1:完成pmap.c里的几个函数，使其能通过两个check函数

要完成的几个函数是关于物理页的操作的，要注意的是当前这个lab都是在为内核服务，这些函数都是内核使用的
简单地回忆一下,lab1中的boot程序通过entry进入到内核，其实也就是进入到了entry.S里的内容，这里主要的工作就是开启分页机制然后调用i386_init函数，i386_init在调用mem_init时就会使用到pmap.c里的内容，所以根据mem_init的代码来看看具体做了什么

首先是侦测内存大小,这个工作代码已经完成，有兴趣可以去看下实现细节，要知道的就是i386_detect_memory这个函数侦测完后会对本文件里的两个全局变量进行赋值:npages--物理内存的大小，npages_basemem--base内存的大小(关于basemem与extmem参见lab1),这两个变量是以页为单位表示内存大小
![展示i386_detect_memory效果](https://s2.loli.net/2023/01/27/raoLAEqCN68Xdy2.png)
然后就是在构建内核的页表，因为当前页表只能映射4MB的空间,且是启动程序写死的，现在需要为内核制作一个页表能映射更大的空间,且根据提前设计好的内存布局来填写内容，具体的内存布局可以在inc/memlayout.h里看到:

```c
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
 */
```

可以先不管上面的复杂，知道现在要做的就是要为内核制作页表，能将上面的布局给反应出来

### boot_alloc

> 以页为单位申请能容纳n字节大小的空间,并返回这片空间的首地址

这个函数只有在jos还没装好真正的页表之前用，其实就是为了制作上面说的页表服务。函数的原理也很简单，首先有一个由链接器生成的end符号(可以在kern/kernel.ld里看到)，在这里把它看作一个地址，这个地址指向bss段的末尾，也就是能够使用的虚拟地址首地址。函数维护一个静态指针nextfree，指向最新的可用页的首地址，通过宏ROUNDUP使其返回整页整页的地址，所以这个函数是整页整页申请空间的，即使n=1,也会分配一页的空间。另外由于链接器生成的地址是指定的虚拟地址，所以这个函数是从内核的虚拟地址申请的空间.

```c

static void *
boot_alloc(uint32_t n)
{
 static char *nextfree; // virtual address of next byte of free memory
 char *result;
 // Initialize nextfree if this is the first time.
 // 'end' is a magic symbol automatically generated by the linker,
 // which points to the end of the kernel's bss segment:
 // the first virtual address that the linker did *not* assign
 // to any kernel code or global variables.
 if (!nextfree) {
  extern char end[];
  nextfree = ROUNDUP((char *) end, PGSIZE);
 }

 // Allocate a chunk large enough to hold 'n' bytes, then update
 // nextfree.  Make sure nextfree is kept aligned
 // to a multiple of PGSIZE.
 //
 // LAB 2: Your code here.
 result = nextfree;
 nextfree = ROUNDUP(nextfree+n,PGSIZE);
 if((uint32_t)nextfree-KERNBASE>(npages*PGSIZE))
  panic("Out of memory!\n");
 return result;
}
```

完成之后回到mem_init,可以看到kern_pgdir就是要建立的页目录表，申请了一个页的大小给它并清空数据。接着插入了一个页目录项到页目录表，这个页目录项就是指向这张表的，具体原因可以暂时不管，后面才会用到。

然后就是创造一个"数组",数组的元素是PageInfo,结构非常简单只包含一个指针和引用计数，这个数组就是当前管理内存的一个数据结构，可以把整个内存看作一个挨着一个的页，每个页的信息就是一个PageInfo管理，所以申请npages个PageInfo的空间出来存放这个数组并对其进行初始化。

### page_init

> 对刚刚的pages数组进行正确的初始化

现在要对这个数组里的元素进行正确的赋值，如果PageInfo里的pp_ref为0表示该页没有被用，空闲中。现在来看看哪些页已经被用过了:

1. 第0页已经用了因为0-4kb这段内存包含了实模式下的IDT和一些BIOS的东西，虽然可能用不着但以防万一表示该页已被使用

2. 剩下的basemem都可用即[PGSIZE,npages_basemem*PGSIZE]可用

3. 接着来到IOhole,这里也不能使用，即[IOPHYSMEM,EXTPHYSMEM]不可用

4. 然后就来到extend memory也就是内核所在，用boot_alloc(0)可以得到内核使用的最远地址，可以换算得到用了几页

```c
void page_init(void)
{
 size_t i;
 for (i = 0; i < npages; i++) {
  if(i==0){
   pages[i].pp_ref = 1;
   //对应第3，4点,因为IO空洞持续到0x100000也就是跟extmem是紧邻的，所以[npages_basemem,PGNUM(PADDR(boot_alloc)]都不可用
  }else if(i>=npages_basemem && i<PGNUM(PADDR(boot_alloc(0)))){
   pages[i].pp_ref = 1;
  }else{
   pages[i].pp_ref = 0;
   pages[i].pp_link = page_free_list;
   page_free_list = &pages[i];
  }
 }
}

```

上面的代码里还有一个存在于本文件的static struct PageInfo*的变量page_free_list,这个变量用于创建一个链表，这个链表存储所有的空闲页，在后面的page_alloc会用到.

### page_alloc

> 申请一个物理页,根据传入参数来决定对这段空间的初始化操作,成功则返回的是一个PageInfo的指针,失败则为null

有了page_init建立的链表这个函数的实现就非常简单，直接从链表里取一个出来即可,注释里提示不要在函数里面去修改pp_ref的值,把这个工作交给调用者。另外链表里存的都是PageInfo的地址,要操作它们代表的页的地址需要用到page2kva函数

```c

struct PageInfo *
page_alloc(int alloc_flags)
{
 // Fill this function in
 if(page_free_list == NULL){
  return NULL;
 }
 struct PageInfo* pg = page_free_list;
 page_free_list = page_free_list->pp_link;
 pg->pp_link = NULL;
 if(alloc_flags & ALLOC_ZERO){
  memset(page2kva(pg),0,PGSIZE);
 }
 return pg;
}

```

### page_free

> 释放PageInfo所代表的物理页

这个函数也很简单,首先进行一些错误情况的处理,无误后把传入的PageInfo加入到空闲链表即可,这样就相当于释放了

```c
void
page_free(struct PageInfo *pp)
{
 // Fill this function in
 // Hint: You may want to panic if pp->pp_ref is nonzero or
 // pp->pp_link is not NULL.
 if(pp->pp_ref!=0||pp->pp_link!=NULL){
  panic("page_free error!\n");
 }
 pp->pp_link = page_free_list;
 page_free_list = pp;
}
```

完成上面几个函数之后应该就能通过check_page_free_list(1),check_page_alloc()两个函数的检测了，make qemu之后应该会打印相关信息

## E2:了解x86的地址转换与段保护

给的图很好的反应了Virtual,Linear,Physical Address之间的关系:

![三个地址的关系](https://s2.loli.net/2023/01/31/vHeGRrq5FoYO1cb.png)

x86机器的内存访问都是段基址:偏移地址的形式,进入保护模式后加入了gdt,selector等内容对内存的访问进行了一些保护，具体的内容可以查看实验里提到的Intel 80386 Reference Manual的相应章节或者在我《真象还原》里的保护模式相应博客。
**简而言之就是程序里使用的地址都是一个偏移地址，需要加上选择子对应的段基址生成一个线性地址,线性地址又需要经过页表转换单元进行映射，最后得到的才是真正的物理地址**两次转换都是硬件自动完成的，内核要做的就是去定义相关的数据结构从而控制程序使用的地址最终会映射到的物理地址

## E3:用GDB和qemu查看内存内容

由于我没有使用实验提供的qemu,所以无法支持`info pg`命令，另外直接make qemu我的终端会和qemu显示一样的内容，看不到qemu的monitor界面,参照资料我修改了GNUmakefile里的QEMUOPTS如下:
`QEMUOPTS = -drive file=$(OBJDIR)/kern/kernel.img,index=0,media=disk,format=raw -monitor stdio -gdb tcp::$(GDBPORT)`
这样终端就是qemu monitor的界面.
gdb跟qemu monitor都有x与xp命令,前者查看虚拟地址内容后者查看物理地址内容

## E4:完成指定函数,使其能通过check_page

### pgdir_walk

> 返回一个页表项指针，该页表项是访问虚拟地址va会使用到的

函数主要的主要内容就是模拟页表单元的工作,首先从得到的pgdir中访问va对应的页目录项，如果该页表项的PTE_P位为0则表示va所属的页表还没有创建,根据参数决定是否创建该页表

```c
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
 // Fill this function in
 pte_t *pte = NULL;
 pde_t *pde = NULL;
 struct PageInfo* new_page = NULL;
 uint32_t dic_off = PDX(va);
 uint32_t page_off = PTX(va);
 pde = pgdir+dic_off;
 if(!(*pde & PTE_P)){
  if(create){
   new_page = page_alloc(ALLOC_ZERO);
   if(new_page == NULL)
    return NULL;
   ++new_page->pp_ref;
   //页目录项里存的是物理地址
   *pde = (page2pa(new_page)|PTE_P|PTE_W|PTE_U);
  }else{
   return NULL;
  }
 }
 pte = KADDR(PTE_ADDR(*pde));
 //返回的是虚拟地址
 return &pte[page_off];
}
```

### boot_map_region

> 将虚拟地址[va,va+size]映射到物理地址[pa,pa+size],va与pa都是页对齐的

只要找到va对应的PTE，将其内容填为pa即可

```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
 // Fill this function in
 pte_t *entry = NULL;
 size_t added = 0;
 for(added=0;added<size;added+=PGSIZE){
  entry = pgdir_walk(pgdir,(void*)(va+added),1);
  *entry = (pa+added) | perm | PTE_P;
 }
}
```

### page_lookup

> 返回虚拟地址va所对应的页，并让pte_store指向对应的页表项

```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
 // Fill this function in
 struct PageInfo *pp=NULL;
 pte_t *entry = pgdir_walk(pgdir,va,0);
 if(entry == NULL)
  return NULL;
 if(!(*entry & PTE_P))
  return NULL;
 pp = pa2page(PTE_ADDR(*entry));
 if(pte_store != NULL)
  *pte_store = entry;
 return pp;
}
```

### page_remove

> 解除虚拟地址va与对应页表的映射关系

```c
void
page_remove(pde_t *pgdir, void *va)
{
 // Fill this function in
 struct PageInfo *pp;
 pte_t *entry;
 if((pp=page_lookup(pgdir,va,&entry))!=NULL){
  tlb_invalidate(pgdir,va);
  page_decref(pp);
  *entry = 0;
 }
}
```

### page_insert

> 将虚拟地址va映射到PageInfo pp对应的物理页地址

```c
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
 // Fill this function in
 pte_t *entry = pgdir_walk(pgdir,va,1);
 if(entry == NULL)
  return -E_NO_MEM;
 //++这一步要放在remove之前，防止va已经映射到pp,导致后面remove时把pp给释放掉了
 ++pp->pp_ref;
 if(*entry & PTE_P){
  page_remove(pgdir, va);
  tlb_invalidate(pgdir,va);
 }
 *entry = (page2pa(pp)|perm|PTE_P);
 pgdir[PDX(va)] |= perm;
 return 0;
}
```

## E5:完成mem_init剩下的内容

E4主要完成了一些操作页表内容的函数，有了E4的基础现在开始构建最开始说到的那张内核的页表

* 首先是将PageInfo数组映射到线性地址UPAGES处且权限为用户只读
`boot_map_region(kern_pgdir,UPAGES,PTSIZE,PADDR(pages),PTE_U);`

* 然后是为内核映射栈区域,栈的大小并不是PTSIZE,而是KSTKSIZE
`boot_map_region(kern_pgdir,KSTACKTOP-KSTKSIZE,KSTKSIZE,PADDR(bootstack),PTE_W);`

* KERNELBASE以上的所有地址映射到从0开始的物理地址
`boot_map_region(kern_pgdir,KERNBASE,0xffffffff-KERNBASE,0,PTE_W);`
