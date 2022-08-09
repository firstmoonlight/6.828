
所谓`lazy allocation`，就是内存的延迟分配。应用程序在申请内存的时候，内核通过`stbrk`系统调用给它分配了内存，但是并未将这些虚拟地址映射到实际地址，直到应用程序使用到了该部分地址的时候，触发了缺页错误，此时，内核才会真正地分配物理内存。

# Part One: Eliminate allocation from sbrk()
修改代码如下：
```
diff --git a/sysproc.c b/sysproc.c
index 0686d29..ef36495 100644
--- a/sysproc.c
+++ b/sysproc.c
@@ -51,9 +51,10 @@ sys_sbrk(void)
   if(argint(0, &n) < 0)
     return -1;
   addr = myproc()->sz;
-  if(growproc(n) < 0)
-    return -1;
-  return addr;
+  //if(growproc(n) < 0)
+    //return -1;
+  myproc()->sz = addr + n;
+  return addr;
 }
```
测试结果如下，触发了14的中断。
![126e1506bf651209ffb7ad3a8ae3ebb7.png](en-resource://database/4458:1)


# Part Two: Lazy allocation
Modify the code in trap.c to respond to a page fault from user space by mapping a newly-allocated page of physical memory at the faulting address, and then returning back to user space to let the process continue executing. 

1. 页故障的中断号为14，从告警`pid 3 sh: trap 14 err 6 on cpu 0 eip 0x112c addr 0xc004--kill proc`可以看出，`traps.h`的注释也说明了`#define T_PGFLT         14      // page fault`。
2. 我们需要照抄`allocuvm`的流程，而不是直接调用它，因为`allocuvm`中调用的是`PGROUNDUP`宏，而不是`PGROUNDDOWN`宏，会导致我们映射的地址并不会涵盖页面故障的地址。
3. 寄存器`cr2`中保存的是出现页故障的时候，32位线性地址。


```
  case T_PGFLT:
    {
      struct proc *curproc = myproc();
      //获取页故障的虚拟地址
      uint va_start = rcr2();
      uint a = PGROUNDDOWN(va_start);
      char *mem = kalloc();
      if(mem == 0){
        cprintf("allocuvm out of memory\n");
        return;
      }
      memset(mem, 0, PGSIZE);
      if(mappages(curproc->pgdir, (char*)a, PGSIZE, V2P(mem), PTE_W|PTE_U) < 0){
        cprintf("allocuvm out of memory (2)\n");
        kfree(mem);
        return;
      }
    }
    break;

```

# optional challenge等以后有空再做
