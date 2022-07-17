准备工作：切换到lab2的分支，然后将lab1的代码合入到lab2中
```
$git checkout -b lab2 origin/lab2
$git merge lab1
```
# Part 1: Physical Page Management
## Exercise1
第一部分只有一个Exercise，即写一个`physical page allocator`。
It keeps track of which pages are free with a linked list of struct PageInfo objects, each corresponding to a physical page.这句话告诉我们这个allocator的基本形式是一个链表，链表的节点是`struct PageInfo objects`。
![66318c1ce1f6b4b300a6c7ab8e6b01ba.png](en-resource://database/4343:1)







### boot_alloc
在内核刚初始化的时候，我们只能通过移动指针来分配内存。`boot_alloc`将指针从`end`开始，增加到`end + n`，表示分配了`n`个字节的内存。
```
static void *
boot_alloc(uint32_t n)
{
	static char *nextfree;	// virtual address of next byte of free memory
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
+++	result = nextfree;
+++ nextfree = ROUNDUP((char *) nextfree + n, PGSIZE); 

	return result;
}
```

### mem_init
`pages`全局变量保存着所有页表的信息。
在我的环境中，`Physical memory: 131072K available, base = 640K, extended = 130432K`，因此总共有32768个页表。
```
//////////////////////////////////////////////////////////////////////
"// Allocate an array of npages 'struct PageInfo's and store it in 'pages'."
"// The kernel uses this array to keep track of physical pages: for"
"// each physical page, there is a corresponding struct PageInfo in this"
"// array.  'npages' is the number of physical pages in memory.  Use memset"
"// to initialize all fields of each struct PageInfo to 0."
"// Your code goes here:"
+++ pages = (struct PageInfo *)boot_alloc(npages * sizeof(struct PageInfo));
+++ memset(pages, 0, sizeof(npages * sizeof(struct PageInfo)));
```

### page_init
由于没有仔细理解`struct PageInfo`中`pp_ref`的意义，因此下面的代码都忽略了对`pp_ref`的赋值。`pp_ref is the count of pointers to this page, for pages allocated using page_alloc.`，即在分配的时候，需要将`pp_ref`置1，表示`page_alloc`时，有一个指针指向该页表。
```
void
page_init(void)
{
	// The example code here marks all physical pages as free.
	// However this is not truly the case.  What memory is free?
	//  1) Mark physical page 0 as in use.
	//     This way we preserve the real-mode IDT and BIOS structures
	//     in case we ever need them.  (Currently we don't, but...)
	//  2) The rest of base memory, [PGSIZE, npages_basemem * PGSIZE)
	//     is free.
	//  3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must
	//     never be allocated.
	//  4) Then extended memory [EXTPHYSMEM, ...).
	//     Some of it is in use, some is free. Where is the kernel
	//     in physical memory?  Which pages are already in use for
	//     page tables and other data structures?
	//
	// Change the code to reflect this.
	// NB: DO NOT actually touch the physical memory corresponding to
	// free pages!
	size_t i;
	for (i = 0; i < npages; i++) {
		pages[i].pp_ref = 0;
		pages[i].pp_link = page_free_list;
		page_free_list = &pages[i];
	}

	// 1) Mark physical page 0 as in use.
	pages[1].pp_link = 0;
	
	//3) Then comes the IO hole [IOPHYSMEM, EXTPHYSMEM), which must never be allocated.
	size_t io_start = IOPHYSMEM / PGSIZE, io_end = EXTPHYSMEM / PGSIZE;
	pages[io_end].pp_link = &pages[io_start - 1];

	// 4) Then extended memory [EXTPHYSMEM, pages + npages * sizeof(struct PageInfo)), has been used
	size_t pages_end = (pages + npages * sizeof(struct PageInfo)) / PGSIZE;
	pages[pages_end].pp_link = &pages[io_start - 1];
		
}
```

### page_alloc 
```
struct PageInfo *
page_alloc(int alloc_flags)
{
	// Fill this function in
	if (page_free_list == NULL) {
		return NULL;
	}
	
	struct PageInfo * res = page_free_list;
	page_free_list = page_free_list->pp_link;
	res->pp_link = NULL;
	
	if (alloc_flags & ALLOC_ZERO) {
		physaddr_t page_addr = page2kva(res);
		memset((void *)page_addr, 0, PGSIZE);
	}
	
	return res;
}
```

### page_free
```
//
// Return a page to the free list.
// (This function should only be called when pp->pp_ref reaches 0.)
//
void
page_free(struct PageInfo *pp)
{
	// Fill this function in
	// Hint: You may want to panic if pp->pp_ref is nonzero or
	// pp->pp_link is not NULL.
	if (pp->pp_ref != 0 || pp != NULL) {
		panic("page_free: arg struct PageInfo error\n");
	}
	pp->pp_link = page_free_list;
	page_free_list = pp;
}
```

### 启动结果
![34ddf517ac168f2e77b916ea1737a90d.png](en-resource://database/4349:1)




# Part 2: Virtual Memory
## Exercise 2. 
**Look at chapters 5 and 6 of the Intel 80386 Reference Manual, if you haven't done so already.**
主要是看5.2 and 6.4这两个章节，学习`page translation`和 `page-based  `。

### 5.2 Page Translation
线性地址转化为物理地址的图例如下：
首先将线性地址分为3个部分，最高位的10个比特表示该地址在页表目录的位置，次高的10个比特表示该地址在页表的位置，最后从页表指向的物理地址处，偏移12个字节，就可以找到最终的物理地址了。
![8263a316ab182e63c1797f4f2b4cddb8.png](en-resource://database/4373:1)
![4befe5fe0a5f2689a552a58240ec264c.png](en-resource://database/4375:0)

### 6.4 Page-Level Protection
基于页表的保护，主要是页表的第1和2比特。
第1比特限定了程序的读写权限
`1. Read-only access (R/W=0)`
`2. Read/write access (R/W=1)`
`When the processor is executing at supervisor level, all pages are both readable and writable. When the processor is executing at user level, only pages that belong to user level and are marked for read/write access are writable; pages that belong to supervisor level are neither readable nor writable from user level.`
第2比特将页表的权限分为两个级别`Supervisor Level`和`User Level`。
`1. Supervisor level (U/S=0) -- for the operating system and other systems software and related data.`
`2. User level (U/S=1) -- for applications procedures and data. `
`The current level (U or S) is related to CPL. If CPL is 0, 1, or 2, the processor is executing at supervisor level. If CPL is 3, the processor is executing at user level.`
![3506af7918cbfe76fb272a3984d2c78f.png](en-resource://database/4371:1)

## Exercise 3.
Use the xp command in the QEMU monitor and the x command in GDB to inspect memory at corresponding physical and virtual addresses and make sure you see the same data.
//TODO

### Question
Assuming that the following JOS kernel code is correct, what type should variable x have, uintptr_t or physaddr_t?
```
        mystery_t x;
        char* value = return_a_pointer();
        *value = 10;
        x = (mystery_t) value;
```
解答：应该是虚拟地址，对于操作系统来说，并不能对物理地址进行解引用，所有的地址都是虚拟地址。


## Reference counting
当多个虚拟地址同时映射到同一个物理页面时，需要用`struct PageInfo `中的`pp_ref `来标记，有多少个虚拟地址同时指向这个物理页面。当`pp_ref  == 0`时，表明这个页面可以被释放了。（通过系统调用的内存地址都是在`UTOP` 之下，`UTOP`之上的地址是在启动过程中通过`boot_alloc`来分配的，不必也不需要被释放）。**Be careful when using page_alloc. The page it returns will always have a reference count of 0, so pp_ref should be incremented as soon as you've done something with the returned page (like inserting it into a page table).**

## Page Table Management


## Exercise 4. 
In the file kern/pmap.c, you must implement code for the following functions.       
```
        pgdir_walk()
        boot_map_region()
        page_lookup()
        page_remove()
        page_insert()
```
`check_page(),` called from mem_init(), tests your page table management routines. You should make sure it reports success before proceeding.

### pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create)
这个函数的作用是给定一个线性地址`va`，页表目录`pgdir`，返回页表`PTE`的地址。
首先是，页表目录的地址是哪个？`kern_pgdir`，这个是函数`pgdir_walk`的第一个实参。
其次，需要创建判断页表目录中，页表是否存在。0